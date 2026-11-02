# 阶段 0 · 度量与归因 设计文档

版本 v1.1 · 2026-11-02（v1.0 · 2026-10-30）
所属：上下文治理方案 阶段 0（前置阶段，其余阶段都以本阶段的输出为输入）
依赖：无
被谁依赖：2A（无损削减）、阶段 3（状态外部化）、2B（遮蔽）、阶段 1（读取纪律）、阶段 4（子 agent 隔离）

运行前提：pi 跑在 **OpenAI 与 DeepSeek** 上——自动前缀缓存、**无写入溢价**。§3.2 的倍率表按此填写，§6.5 的模拟公式随之参数化。

---

## 0. 定位

**解决什么**：回答"token 到底花在哪"以及"值不值得做后续四个阶段"。

**不解决什么**：不做任何上下文修改、不调模型、不改 pi 的 session 文件、不做实时监控、不做跨会话检索。

**为什么它必须是第一个阶段**（三条，按严重性）：

1. **不先量就动手，可能修错地方。** 你以为瓶颈是"历史积累"，实际上可能是"少数几次超大读取"，也可能是"system prompt 每轮变化导致缓存全废"。前者要裁剪，后者只要改一行字符串——成本差两个数量级。
2. **后续所有阶段的指标必须在这里定义一次。** 阶段 2 的 go/no-go 要用"步数、重复犯错次数、每任务成本"，这些口径如果到阶段 4 才想起来定义，前面的数据全部作废、没法比较。
3. **没有基线就没有 A/B。** 度量本身不产生价值，但它让后面每一步的价值可被证明，也能让错的方案被及时关掉。

---

## 1. 设计原则

| 原则 | 具体含义 |
|---|---|
| **只读** | 输入文件全程不写。运行前后校验 mtime + sha256 不变（这条要写成断言，不是注释） |
| **离线** | 不调用任何模型。纯计算，可在 CI 里跑 |
| **可重放** | 同一输入 + 同一配置 → 逐字节相同的输出（时间戳只进元数据，不进正文） |
| **口径显式** | 每个数字都标注口径（raw / effective、含/不含缓存、估算/实测） |
| **误差可见** | 估算值必须附残差报告，不允许把估算当实测输出 |

---

## 2. 输入

### 2.1 数据源

pi 的 session 文件：`~/.pi/agent/sessions/<project>/<session>.jsonl`

结构（以当前版本为准；实现时以实际文件为准，解析层必须容忍未知字段与未知 entry 类型）：

```
第 1 行: session 头（版本、cwd、时间等）
之后每行: 一个 entry
  { id, parentId, type, timestamp, ...payload }
  type ∈ { message, model_change, thinking_level_change, compaction,
           branch_summary, custom, custom_message, label, session_info }
```

- `message` 的 payload 里有 role（user / assistant / toolResult）与 content blocks（text / thinking / tool_use / tool_result）
- **未知 type 一律跳过并计数**，不要抛错（pi 升级会加类型）

### 2.2 路径解析

session 是树。分析的基本单位是**一条 leaf path**：从某个叶子沿 `parentId` 回溯到根，再反转。

| 场景 | 取法 |
|---|---|
| 默认 | 取"最后一个 message entry 所在分支"的 leaf；若 `session_info` 里有记录则以其为准 |
| `--all-branches` | 对每个没有子节点的 entry 各算一条路径，分别输出 |
| 路径上有 `compaction` | 该 entry 之前的 message 不再进入上下文（被摘要替代），但**仍然计入 raw 口径** |

**双口径是硬要求**，否则增长曲线会算错：

| 口径 | 定义 | 用途 |
|---|---|---|
| `raw` | 路径上全部 message 的 token | 看历史总量、看压缩救回了多少 |
| `effective` | 应用 compaction 之后实际会送进模型的 token | 看真实成本、算增长斜率 |

### 2.3 额外来源（必须单独归因，否则必然误判）

实际输入 token 通常远大于"消息内容之和"，差额来自：

- 工具定义（pi 默认 4 个工具，加上扩展可能十几个）
- skills / AGENTS.md 注入
- 扩展每轮注入的消息
- 系统提示

这些必须单列成 `overhead` 类别。**否则你会把"工具定义占 12k"误认为"工具输出是大头"，从而做一个完全错方向的优化。**

获取方式（优先级从高到低）：
1. provider 返回的 `usage.input_tokens - Σ(消息估算)` → 差值即 overhead（最可靠）
2. 解析 system prompt 与 AGENTS.md 的实际内容（次可靠，依赖 pi 的拼装规则）
3. 都没有则标注 `overhead = unknown`，并在报告首行警告

---

## 3. token 计量

### 3.1 两段式：估算 + 校准

多 provider 环境下拿不到统一 tokenizer，所以分两步：

**第一步：分桶估算**

```
est(text) = ceil(cjk_count / 1.6 + other_count / 3.8) + 4
```

- `cjk_count`：正则 `[\u4e00-\u9fff\u3040-\u30ff\uac00-\ud7af]` 命中数（中日韩字符约 1.5–1.7 字/token）
- `other_count`：其余字符（代码与英文约 3.6–4.0 字符/token）
- `+4`：每条消息的固定开销（role 标记、分隔符）

**第二步：用真实 usage 校准**

对有 `usage` 的轮次做最小二乘拟合：

```
input_tokens_i ≈ k · est(messages_i) + b
```

报告 `k`、`b`、以及 **MAPE（平均绝对百分比误差）**。

| MAPE | 判定 |
|---|---|
| ≤ 15% | 可用于绝对金额估算 |
| 15%–30% | 只可用于相对比较（谁比谁大） |
| > 30% | 报告首行标注"估算不可信"，只输出占比不输出金额 |

**允许的取巧**：若同一会话里模型未变且 usage 完整，直接对每一步用 `usage` 的真实值，估算只用于"新建的、还没有 usage 的"部分（遮蔽模拟需要）。

### 3.2 缓存字段

若 usage 提供，记录每轮：

```
input_tokens, cache_read_input_tokens, cache_creation_input_tokens, output_tokens
```

- `cache_read_ratio = cache_read / (cache_read + cache_creation + uncached_input)`
- 若 provider 不提供 → 该列输出 `n/a`，不猜

**价格倍率必须按 provider 显式配置，不能写死在公式里。** 这是 v1.0 的一处错误：把 Anthropic 的显式缓存计费当成了通用物理。

| provider | `c` 命中价 | `w` 重算价 | 缓存方式 | 粒度 / TTL（需实测） |
|---|---|---|---|---|
| **OpenAI** | **0.5** | **1.0** | 自动前缀 | 1024 token / 5–10 min |
| **DeepSeek** | **0.1** | **1.0** | 自动前缀 | 64 token / 小时级 |
| （Anthropic，本文不适用） | 0.1 | 1.25 | 显式 `cache_control` | 按块 / 5 min–1 h |

`w = 1.0` 的含义是：改写位置之后的内容**按未命中价重算，没有额外溢价**。两家与 Anthropic 的结构性差异就在这里，它直接改变 §6.5 与 §13 的结论方向。

---

## 4. 数据结构

```jsonc
// TurnRecord：一轮（一次用户消息到下一次用户消息之间）
{
  "turn": 7,
  "user_text_tokens": 42,
  "assistant_tokens": 1180,          // 含 thinking 与 tool_use 参数
  "tool_results": [
    { "callId": "c1", "tool": "read", "tokens": 12480,
      "target": "src/auth.ts", "bytes": 48213, "lines": 361 }
  ],
  "turn_tokens_effective": 13702,
  "turn_tokens_raw": 13702,
  "usage": { "input": 41200, "cacheRead": 36800, "cacheWrite": 4100, "output": 640 },
  "overhead_est": 8600,
  "compaction": null
}

// SessionSummary
{
  "sessionId": "...", "leaf": "...", "branches": 1,
  "model": "claude-sonnet-*",
  "turns": 42, "steps": 96, "toolCalls": 71,
  "tokens": { "rawTotal": 812000, "effectivePeak": 128400,
              "final": 118000, "outputTotal": 31000 },
  "shares": { "system": 0.06, "toolDefs": 0.09, "user": 0.03,
              "assistant": 0.14, "toolResults": 0.61, "other": 0.07 },
  "byTool": { "read": 0.42, "grep": 0.11, "bash": 0.07, "edit": 0.01 },
  "byPath":  { "src/auth.ts": 38200, "src/middleware.ts": 21400 },
  "growth": { "slopePerTurn": 2650, "r2": 0.94 },
  "cache": { "readRatio": 0.89, "instabilityEvents": 3 },
  "repeatErrors": 2,
  "estError": { "mape": 0.11, "k": 1.06, "b": 12 }
}

// SimEvent：遮蔽模拟的单次事件
{
  "atTurn": 9, "entryId": "...", "tool": "read",
  "provider": "deepseek",
  "M": 12480, "m": 160, "S": 2240,
  "delta": 928, "paybackTurns": 0.75,
  "wouldMask": true, "skipReason": null
}
```

---

## 5. 指标定义（口径必须一致，后续阶段直接复用）

| 指标 | 定义 | 口径 |
|---|---|---|
| `tool_share` | 工具输出 token / effective 总量 | 按轮取均值，非全程占比 |
| `tool_share_by_tool` | 各工具输出 token / 工具输出总量 | |
| `top_paths` | 按 target 路径聚合 token 的 top N | 只对 read/grep/edit |
| `growth_slope` | effective 总量对轮次的线性拟合斜率 | 剔掉 compaction 造成的负跳变 |
| `effective_peak` | 全程 effective 最大值 | 判断是否真的撞窗口 |
| `compaction_events` | compaction entry 数 + 每次前后跳变 | |
| `cache_read_ratio` | §3.2 | 有 usage 才报 |
| `prefix_instability` | 相邻两轮"前缀区"（system + 早期注入）hash 不同的次数 | 见 §6.3 |
| `turns` / `steps` / `tool_calls` | 用户消息数 / assistant 消息数 / tool_use 数 | |
| `repeat_error_count` | 归一化错误文本重复出现的次数 | 见 §6.4 |
| `mask_sim.savings` | 模拟开启遮蔽后累计省下的 token | 逐轮递推，非静态估算 |
| `mask_sim.payback` | 每条事件的 `n*` 分布（中位数 / P90） | |
| `mask_sim.s_hist` | `S` 的分档直方图（`S` 只计上一轮已进入缓存的 token，不含本轮新增） | **这是决定成败的参数** |

---

## 6. 四个分析模块

### 6.1 归因

按五个维度切：role、tool、path、turn、overhead。每个维度输出 top 10 + 其余聚合。

输出示例（report.md 表格）：

```
| 维度 | 项 | tokens | 占 effective |
|---|---|---|---|
| role | toolResults | 78,400 | 61.0% |
| role | assistant | 18,200 | 14.2% |
| overhead | toolDefs | 11,500 | 8.9% |
| tool | read | 54,100 | 42.1% |
| path | src/auth.ts | 38,200 | 29.7% |
```

### 6.2 增长与压缩事件

- effective 总量随轮次的曲线（报告里用文本柱状或交给后续可视化）
- 斜率与 R²
- 每次 compaction 的前后跳变（记录 `before / after / 摘要长度`）
- **判断是否真的撞过窗口**：`effective_peak / contextWindow`。若 < 0.5，说明从未接近上限，压缩的紧迫性被高估

### 6.3 缓存抖动检测

目的：验证"缓存命中率低"是不是压缩造成的——很可能不是，而是前缀每轮在变。

```
对每一轮：
  prefixFingerprint = sha256(system_prompt_content + 第一轮注入内容)
相邻轮比较，不同则记一次 instabilityEvent，并输出首个差异行（前 120 字符）
```

常见元凶（报告里直接点名）：
- system prompt 里含当前时间 / 日期
- 每轮注入 git status、文件树
- 扩展注入的动态状态块放在历史之前
- 切换模型 / thinking level（这会改变前置内容）

**这个模块的价值可能超过整套压缩方案**：如果每轮都 miss，修一处字符串就能拿回全部缓存收益，成本近乎零。

### 6.4 重复错误检测（为阶段 4 的 A/B 预埋）

对 bash 输出与 assistant 文本抽错误行：

```
/(?:error|exception|fail(?:ed)?|cannot|not found|✗|Error:)/i
```

归一化后再统计：
- 去掉时间戳、绝对路径、行号、hash、随机 id、数字
- 取前 80 字符作为指纹
- 同一指纹出现 ≥ 2 次 → 计一次 `repeat_error`

**必须在阶段 0 定义**，否则阶段 4 无法比较"是否重复踩同一个坑"。

### 6.5 遮蔽模拟（不调模型）

这是阶段 0 最有价值的输出：在没有一行产品代码的情况下，算出阶段 2 到底赚不赚。

**算法（逐轮递推，不是静态公式）**：

```
state: maskedIds = {}, effective[t]
for t in turns:
    # 1) 先按 lag-1 规则找 eligible（与实现同一套判定）
    for msg in context(t):
        if not isToolResult(msg): continue
        if !lagSatisfied(msg, t): continue          # I1
        if !concluded(msg): continue                # I4（阶段 0 可先按"下一轮即视为收尾"近似）
        M = tokens(msg); m = estimateMaskTokens(msg)
        S = cachedTokensAfter(msg, context(t))   # 只算上一轮已进入缓存的部分，不含本轮新增
        if M < max(minTokens, ratio * S): continue
        Δ = w*m + (w - c)*S - c*M                # c/w 取自 §3.2 的 provider 表
        n = max(0, Δ) / (c*(M - m))
        paybackOk = c*(M - m)*(R - t) > w*m + (w - c)*S   # 会话级判据，R 见 §6.2
        emit SimEvent(...)
        maskedIds.add(msg.id)
    # 2) 计算本轮 effective（被遮蔽的用 m 替代）
    effective[t] = sum(m if id in maskedIds else tokens(...))
```

三个必须一起输出的东西：

1. **收益曲线**：`effective_original[t]` vs `effective_masked[t]`，以及累计省下的 token
2. **敏感性分析**：把"假设在 t-5 / t-10 轮前遮蔽"的 `Δ` 也算一遍，直接展示 FIFO 违规的代价陡增（这是设计文档 §2.4 那张回本表的实测版）

3. **判据占比**：按 §2.4 的会话级不等式逐条判定 `paybackOk`，输出"满足条件的条目 / 总候选条目"。**这是 2B 做不做的唯一依据**：≥ 50% 建议开启；低于 50% 延后，但不"预期不做"——原方案把这件事写成了定性预期，在 OpenAI 上方向是错的。

**局限（必须写进报告）**：模拟假设"被遮蔽后模型行为不变"。真实的步数变化、重读行为只能靠阶段 4 的真跑。模拟只回答"账算不算得过来"，不回答"模型会不会变笨"。

---

## 7. CLI 契约

```
ctx-report scan --dir <sessionsDir> [--since YYYY-MM-DD] [--all-branches] --out <dir>
ctx-report sim  --session <id|path> [--config <mask.json>] --out <dir>
ctx-report calibrate --dir <sessionsDir> --out <catalog-calib.json>
ctx-report check --dir <sessionsDir>            # 只做健康检查，秒级
```

| 子命令 | 输入 | 输出 | 退出码 |
|---|---|---|---|
| `scan` | session 目录 | `report.md`、`report.json`、`turns.csv` | 0 成功 / 1 无可解析会话 / 2 参数错 |
| `sim` | 单会话 + 遮蔽配置 | `sim.json`、`report.md` 追加"模拟"章节 | 同上 |
| `calibrate` | session 目录 | 校准系数与 MAPE | 同上 |
| `check` | 目录 | 会话数、分支数、usage 覆盖率、异常 entry 数 | 同上 |

**只读断言**：每个子命令启动与结束时各算一次输入文件 sha256，不一致则退出码 3 并报错。

---

## 8. 输出物

### 8.1 `turns.csv` 列（固定顺序，便于后续 diff）

```
turn,user_tokens,assistant_tokens,tool_tokens,turn_total_effective,
input,cache_read,cache_write,output,overhead_est,compaction,masked_sim,sim_saving
```

### 8.2 `report.md` 骨架

```
# 会话度量报告 <sessionId>
> 口径说明 / 估算误差 MAPE / 数据缺口警告

## 1. 结论摘要（三行以内，直接给判断）
## 2. 归因（五维表）
## 3. 增长与压缩事件
## 4. 缓存抖动（含首个差异行）
## 5. 遮蔽模拟（收益曲线 / Δ 分布 / S 直方图 / 敏感性）
## 6. 建议动作（按 §9 判定）
## 7. 已知局限
```

### 8.3 `report.json`

字段见 §4 的 `SessionSummary`，外加 `sim` 数组。

---

## 9. 判定规则（报告最后一节必须给出结论，不允许只给数据）

| 观测特征 | 结论 | 首要动作 |
|---|---|---|
| 工具输出占比 > 50%，且 top1 工具 > 30% | 积累是主因 | 阶段 2（遮蔽）优先，同时做该工具的读取纪律 |
| 存在少数巨块（单次读取 > 总量 20%） | 单次过大是主因 | 阶段 1 优先；阶段 2 收益有限 |
| `prefix_instability > 0` 且 `cache_read_ratio` 低 | 缓存被自己废掉 | **先修抖动**，零成本，可能比压缩赚得多 |
| 会话中位数 < 30 轮，或模拟 `S` 中位数 > 8k | 回本慢 | 阶段 2 降级为可选，先做阶段 1 + 3 |
| `effective_peak / window < 0.4` | 从未接近上限 | 压缩的紧迫性被高估，按成本而非质量决策 |
| `overhead > 25%`（工具定义 + 系统提示） | 固定开销大 | 精简工具集 / skills，比压缩历史更直接 |

多条同时命中时，按"零成本优先 → 低成本 → 高成本"排序执行。

---

## 10. 验收标准

1. 对 ≥ 3 个真实会话跑通，无未捕获异常；未知 entry 类型计数 > 0 也不中断
2. 有 usage 的会话，MAPE ≤ 15%（阶段 0 的硬门槛，因为后续所有金额计算都靠它）
3. `report.md` 必须能直接回答四个问题：工具输出占比多少、top 5 消耗源、是否值得遮蔽、缓存是否在抖
4. 只读断言通过：输入文件 sha256 前后一致
5. 100MB 会话运行时间 < 30s（纯流式解析，不建全量索引）
6. 同一输入两次运行，`report.json` 逐字节相同（时间戳只进顶层 `generatedAt`）

---

## 11. 非目标与已知局限

| 项 | 说明 |
|---|---|
| 不做实时监控 | 阶段 0 是离线分析。实时成本面板是后续独立需求 |
| 不做跨会话聚合的语义分析 | 只做统计聚合（按项目、按时间），不做检索 |
| 不追求绝对精确 | 估算 + 校准，目标是把误差控制在能让决策成立的范围内 |
| 不覆盖非 pi 格式 | 输入契约就是 pi 的 JSONL；其他 agent 的日志另做适配层 |
| 分支合并 | 多分支各算各的，不做合并统计（合并语义不明确） |

---

## 12. 实施步骤（四个半天）

| 序号 | 交付 | 验收 |
|---|---|---|
| 1 | 解析层：流式读 JSONL、树重建、leaf 选择、双口径 token 统计 | 能打印单会话的 turns/steps/tool_calls 与 raw/effective 总量 |
| 2 | 估算器 + 校准 + 残差报告 | 三个会话输出 MAPE；MAPE > 30% 时报告带警告 |
| 3 | 归因（五维）+ 缓存抖动检测 + 重复错误检测 | `report.md` 生成，四个问题都能回答 |
| 4 | 遮蔽模拟（逐轮递推）+ 敏感性分析 + 判定输出 | `sim.json` + 结论段落；与设计文档 §2.4 的回本表量级一致 |

第 4 步的验收有个隐含校验：**如果模拟出的 `S` 中位数远超 5k，说明 lag-1 的判定实现有问题**（因为 lag-1 的定义就是"改动点之后只有一个回合"）。这会反过来验证阶段 2 的实现前提。

---

## 13. 附录 A：为什么 `S` 是唯一要盯的参数

阶段 2B 的账全部落在 `Δ = w·m + (w−c)·S − c·M`。在本项目的两家 provider 上 `w = 1.0`，于是简化为 `Δ = m + (1−c)·S − c·M`——OpenAI `m + 0.5S − 0.5M`，DeepSeek `m + 0.9S − 0.1M`。三个变量里：

- `m` 由模板决定，可控且稳定（约 160）
- `M` 由工具输出决定，不可控
- `S` **由遮蔽时机决定，是唯一的设计变量**

所以阶段 0 的模拟必须输出 `S` 的分布，而不是只输出一个"能省多少"的总数。分布告诉你：如果 P90 的 `S` 是 20k，那说明有相当一部分场景的滞后远超一轮，阶段 2 需要先修判定逻辑，而不是先调模板。

## 14. 附录 B：常见误判

| 现象 | 常见错误解读 | 正确解读 |
|---|---|---|
| 输入 token 很大 | "上下文太长，要压缩" | 先减 overhead（工具定义/系统提示），可能占 25% |
| 缓存命中率低 | "压缩破坏了缓存" | 多半是前缀里有每轮变化的内容（时间戳、状态块） |
| 工具输出占比高 | "要压缩工具输出" | 先看分布：少数巨块 → 改读取粒度；长尾小块 → 才用遮蔽 |
| 会话很长 | "一定撞窗口了" | 看 `effective_peak / window`；很多长会话其实只用了一半 |

---

## 15. 修订记录

**v1.1 · 2026-11-02**

| 改动 | 原内容 | 原因 |
|---|---|---|
| 缓存倍率按 provider 参数化（§3.2） | 假定 `w=1.25, c=0.1` | 项目跑 OpenAI（`c=0.5`）与 DeepSeek（`c=0.1`），两家都**没有写入溢价**（`w=1.0`） |
| 模拟公式参数化（§6.5、§13） | `Δ = 1.15·S + 1.25·m − 0.1·M` | 只是 Anthropic 特例，不能当通用式 |
| `S` 的定义收紧（§5、§6.5） | "改动点之后的内容" | 必须排除本轮新增 token，否则模拟系统性高估 `S`——附录 §12 那句"若 S 中位数远超 5k 说明实现有问题"，很可能就是这个口径错误本身 |
| SimEvent 示例数值（§4） | `delta=1893, paybackTurns=1.6` | 与公式不符。按 DeepSeek 口径应为 `928` 与 `0.75` |
| 新增第三个输出（§6.5） | 只输出收益曲线与敏感性 | 增加"判据占比"，它是 2B 决策的唯一输入 |

指标口径、五维归因、缓存抖动检测、重复错误检测**均未改动**。
