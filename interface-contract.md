# 接口对账表 · 跨文档唯一来源

版本 v1.0 · 2026-11-02
配套：phase-0 / phase-1 / phase-2 / phase-3 / lag1-masking-design / overview
效力：**任何文档与本文冲突，以本文为准。** 改本文的任何一行，必须回来清点全部引用方。

---

## 0. 为什么需要这个文件

v1.1 是逐条打补丁的修订。单点看起来都对，但补丁没同步到接口上，于是产生了至少六处跨文档矛盾：

| 矛盾 | 表现 |
|---|---|
| A7 降级为紧急制动 | phase-2 把日常截断关到 20k token，phase-1 §7.1 的 `bash` 软上限仍是 200 行 / 8KB |
| §6 改成"边际收益不为零" | phase-2 §4.2 的 `reducedBy: "2a"` 仍让 2B 跳过所有被 2A 碰过的消息——按那条规则，2B 收益恒为零 |
| 总览称注入块"不随轮数增长" | phase-3 §8.4 自己承认 500 token/轮累积到 25k |
| 阶段 3 的回收依赖 2B | 2B 默认关闭，且排在阶段 3 之后——链条断在中间 |
| blob 有两个路径 | `<sessionTmp>/blobs/` 与 `ctx/<sessionId>/blobs/`，且一个在临时目录（会被清理，与 I5 冲突） |
| "全关逐字节一致"当全局不变量 | phase-1 §15 已说明注册工具就改前缀；总览与 phase-2 §2 仍无条件引用 |

这些矛盾不在任何一份文档内部，**在接口上**。所以接口需要一份单一来源，而不是每份文档各写一遍。

---

## 1. 共享常量

| 常量 | 值 | 唯一所有者 | 引用方 |
|---|---|---|---|
| `a7.hardCapTokens` | `20000` | phase-2 §7 | phase-1 §7、phase-2 §3 | 
| `read.fullMaxLines` | `1500` | phase-1 §11 | phase-1 §5.1、overview 图 3 |
| `read.forceOutlineAbove` | `1500` | phase-1 §11 | phase-1 §5.1 |
| `read.hardMaxLines` / `read.hardMaxBytes` | `20000` / `5242880` | phase-1 §11 | phase-1 §5.4 |
| `minTokens` | `1000` | lag1 §10 | phase-0 §6.5 模拟必须读同一值 |
| `minRatioOverSuffix` | `4`（语义见 §4） | lag1 §10 | phase-0 §6.5 |
| `maskBudgetTokens` | `160`，多块 `160+40×(块数−1)`，硬上限 `320` | lag1 §10 | lag1 §6、phase-2 §4 |
| `inject.budgetTokens` | `500`，硬上限 `800` | phase-3 §13 | phase-3 §8 |
| `constraints.hardLimit` | `15` | phase-3 §13 | phase-3 §7 |
| `charPerToken` | `3.5` | lag1 §10 | phase-0 §3.1 估算器 |

**禁止事项**：不允许任何阶段定义第二个 `bash` 截断阈值。v1.1 的 `bash.softLines/softBytes` 已删除，理由见 phase-1 §7.1。

---

## 2. 共享断言（同名同义，不得各写一份）

| 断言 | 精确定义 | 适用范围 |
|---|---|---|
| `switch_equivalence` | 全部开关关闭时，组装后的消息列表与未安装插件逐字节一致 | **仅阶段 0 / 2A / 2B / 3**（不改工具集）。阶段 1 注册工具即改变请求前缀，其基线只能是"插件未加载 + 重启进程" |
| `unseen_truncation = 0` | 不存在"截断了但未标注省略行区间"的事件 | 2A |
| `double_reduced_count = 0` | 同一原始内容被 **L2** 削减两次（两个 blob 指向同一内容） | 2A / 2B |
| `needed_but_removed = 0` | 被削减内容在 **recall 之后仍**导致任务失败或重复探索的条数 | 全部削减机制 |
| `prefix_instability` | 相邻两轮前缀区指纹不同的次数 | 阶段 0 §6.3 |

### 2.1 `needed_but_removed` 的口径修正

v1.1 定义为"被削减内容后来被 recall / 重读 / 导致失败的条数"，并要求 `= 0`。**这条按定义不可能通过**：recall 是 2B 的正常使用路径（lag1 §7.3 明确说它是"复核证据"的手段），只要模型正常使用 recall，计数就必然非 0。

改为只看"召回之后仍然不够"：recall 之后模型是否重复探索同一目标、是否因信息缺失而失败。recall 本身不计入。另设 `recall_hit_rate` 作正向观测（召回后正常推进的比例），不设门槛。

### 2.2 断言型指标的使用边界

`blind_edit_blocked > 0`、`middle_highlight_hit > 0` 这类"必须大于 0 才说明机制生效"的指标，**只能作为测试用例的断言，不能作为生产环境的健康检查**。生产环境里模型守规矩、输出中段无报错时它们本来就是 0，把 0 当故障会让值守误判。生产侧只观测，不告警。

---

## 3. 文件路径（唯一版本）

```
~/.pi/agent/
  sessions/<project>/<session>.jsonl      # 源真相，本插件只读
  state/<projectId>/                      # 本插件工作区
    blobs/<entryId>.<ext>                 # 不可重放输出原文（持久）
    blobs/index.jsonl                     # entryId → {kind, bytes, sha, source, createdAt}
    masks.jsonl                           # 冻结的遮蔽记录，只追加
    pins.json
    metrics.jsonl
    offsets.json                          # 派生，可删可重建
    facts/<sessionId>.jsonl               # 事实日志：仓库外

<repo>/.pi/state/
  plan.md · decisions.md · constraints.md · handoff.md    # 人要看、要提交
```

三条硬规则：

1. **blob 只有一个位置**：`state/<projectId>/blobs/`。v1.1 的 `<sessionTmp>/blobs/`（phase-2 §4.5）与 `ctx/<sessionId>/blobs/`（lag1 §8）是同一份东西的两个错误路径——前者是临时目录，会被清理，与 I5「证据必须仍可取得」直接冲突。
2. **`facts/` 不在仓库内**（v1.2 变更）。命令输出会进 facts，§5 的脱敏盖不全时，`.gitignore` 只是"建议"就等于泄漏面扩大。需要提交给人看的是 `plan.md` / `decisions.md` / `constraints.md`，事实日志不进 git。
3. **回读统一**：`read(<blobPath>)` 是模型可见的动作，`recall(entryId)` 是语义封装，内部读同一个 blob。两者不各建一套存储。

---

## 4. 成本参数与判据

provider 倍率见 phase-0 §3.2；本项目两家（OpenAI `c=0.5`、DeepSeek `c=0.1`）均 `w=1.0`。

```
Δ     = w·m + (w−c)·S − c·M
n*    = max(0, Δ) / (c·(M − m))
判据  = c·(M − m)·(R − t) > w·m + (w−c)·S
```

**`minRatioOverSuffix` 的语义修正（v1.2）**

v1.1 的 §5.1 注释称"`4×S` 大致对应 DeepSeek 回本 ≤ 4 轮"、"`1.8×S` 对应 4 轮"。按上式重算：

| 代入 | 结果 |
|---|---|
| `M = 4S`（S = 2k / 8k / 30k） | 回本 **1.27–1.48 轮**，不是 4 轮 |
| `n* = 4` 反解 | `M ≈ 1.8·S + 448`，S 越大越趋近 `1.8S` |

所以 `ratio = 4` 的真实含义是"**回本 ≤ 1.5 轮**"的保守门槛，比注释描述的严格得多。配置值保留 `4`（对质量更有利），但注释与 phase-0 的模拟输出必须按真实含义解释。

**行为成本的缺口（必须写进报告）**：上式计算的是纯 token 账，不含 re-read 的行为成本。一次多余的 `read` 约 10k 当量，而 Δ 的量级是几百到 2k——`re_read_rate` 从 5% 升到 25%，每四条遮蔽就多一次完整读，吃掉 5–20 条遮蔽的收益。所以 `判据` 给出的是**上界**，不是预期值；2B 的 go/no-go 必须同时看 `re_read_rate` 的绝对门槛（< 10%）与增幅（< 20%）。

---

## 5. 脱敏（唯一版本）

正则以**捕获组**书写，替换用 `$1***`。v1.1 写成 `(?:...)` 非捕获组却替换 `$1`，输出是空的 `=***`——密钥红线不该带病。

```
# 规则 1：键值形态（保留键名，遮蔽值）
((?:api[_-]?key|access[_-]?token|secret|password|authorization|passwd|pwd)\s*[:=]\s*)\S+     → $1***

# 规则 2：常见 token 前缀
sk-[A-Za-z0-9_-]{20,}                        → sk-***
gh[pousr]_[A-Za-z0-9]{20,}                   → gh*_***
AKIA[0-9A-Z]{16}                             → AKIA***

# 规则 3：JWT（三段 base64url）
eyJ[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}   → eyJ***

# 规则 4：Bearer 头
Bearer\s+[A-Za-z0-9._~+/=-]{20,}             → Bearer ***

# 规则 5：连接串（含内嵌口令）
(?:postgres(?:ql)?|mysql|mongodb(?:\+srv)?|redis|amqp|mssql)://[^\s"']+   → [connection-string]

# 规则 6：私钥块
-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----   → [private key]

# 规则 7：.env 形态整行（仅在识别为 .env / *.env 上下文时启用）
^[A-Z][A-Z0-9_]{2,}\s*=\s*\S{8,}$            → <KEY>=***
```

规则 5 与规则 7 是 v1.2 新增：v1.1 的六条正则盖不住 JWT、连接串和 `.env`，而命令输出进 `facts/` 之后，这三类是最常见的实际泄漏源。

---

## 6. 阶段依赖与顺序

```
0 度量 → 2A（仅无损）→ 3 状态外部化 → 2B 遮蔽 → 1 读取纪律（影子先行）→ 4 子 agent
```

硬依赖：

- **2B 的 I4 判定依赖 phase-3 的步骤状态** → 3 必须先于 2B 上线。
- **phase-3 的注入块不再依赖 2B**（v1.2 变更）。v1.1 把"回收旧注入块"挂到 2B 上，但 2B 默认关闭且排在阶段 3 之后——链条断在中间。新方案是注入块根本不进历史（phase-3 §8.4），2B 是否开启与它无关。
- 阶段 1 的视图改动只依赖影子模式审查，不依赖其他阶段。

---

## 7. 指标归并（同名不同义的三处）

| 指标 | v1.1 的分歧 | 统一为 |
|---|---|---|
| `truncation_rate` | phase-1 §12"2A 截断触发率" / phase-2 §8"触发 A7 的工具结果比例" | 同一个定义，以 phase-2 §8 为准；phase-1 引用不重定义 |
| `re_read_rate` | lag1 §13.1"增幅 < 20%" / phase-1 §12"< 10%" | 两个都要看：绝对值 < 10% **且** 增幅 < 20%（基线本身可能是 15%） |
| 注入块 token | `inject_tokens`（单块） / `inject_tokens_total`（上下文中全部） | 临时拼接方案下两者应恒等；若 `total > 1.6×单块`，说明拼接渗进了历史，是 bug |

---

## 8. 变更流程

1. 改本文任何一行前，先全目录 `grep` 该常量的引用点。
2. 五份设计文档引用本文而**不复制数值**（表格里可以写值，但必须标"见 contract §1"）。
3. 实现可以偏离文档，但必须回来改这张表——否则它就是第五份会过期的文档。

---

## 9. 修订记录

**v1.0 · 2026-11-02**：新建。起因是 v1.1 的补丁式修订留下了六处跨文档矛盾（§0）；本文把共享常量、断言、路径、判据、顺序收敛到一处。
