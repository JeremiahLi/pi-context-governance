# 阶段 3 · 状态外部化 设计文档

版本 v1.0 · 2026-10-30
所属：上下文治理方案 阶段 3（四份里最重、长期价值最高）
前置：阶段 0（度量）
被谁依赖：阶段 2B 的 I4（结论先行）依赖本文的 decisions 记录；阶段 4（子 agent）依赖本文的 handoff

> **本阶段的优先级已被 [ADR-0001](adr-0001-quality-first-defaults.md) 提前**（2026-11-02）：新顺序为 0 → 2A（仅无损）→ **3** → 2B → 1。本文 §11 关于「阶段 3 必须不晚于 2B」的依赖关系成立，而 `overview` 的路线图与此冲突，以 ADR 为准。另：§8 的「注入块大小恒定」在实现前需先解决旧注入块的回收问题（ADR 未决项 U2）。

---

## 1. 定位

### 1.1 一句话

把"agent 应该记住的东西"从对话历史里搬出来，写进**可校验的文件**，每轮只回注一小块。

### 1.2 它和 2A/2B 的本质区别

2A 和 2B 都在**减少上下文**，代价是可靠性下降（信息变少）。状态外部化是唯一一个"减少上下文的同时**提高**可靠性"的机制：它把易丢失的、隐式的状态，变成可验证的、显式的文件。

所以它的价值主张不是省 token，是：

| 收益 | 说明 |
|---|---|
| 抗遗忘 | 不依赖"模型还记得"，读到就是知道 |
| 抗虚报 | 完成状态由条件推导，不由模型声明 |
| 可恢复 | 进程死了、上下文清空了，只凭文件继续干 |
| 可审计 | 改过什么、跑过什么、为什么这么决定，都有记录 |
| 人机共编 | 人可以直接改计划与约束，模型必须看见 |

这五条**不会随模型升级消失**——这是它区别于 2B 的地方。

### 1.3 不解决什么

- 不做跨会话的语义检索（那是另一层）
- 不做任务分解（阶段 4）
- 不替代 pi 的 session JSONL（那是事实日志，本阶段是**结构化视图**）

### 1.4 诚实的前提警告

pi 官方立场是"No built-in to-dos，用 TODO.md 就行"。**如果你只需要"记着下一步做什么"，一个 TODO.md 确实够了**，本文是过度设计。

本文解决的是 TODO.md 解决不了的四件事：完成状态可验证、失败与被否决方案有记录、约束不可被遗忘、能冷启动恢复。只有当这四件事对你成立时，本阶段的复杂度才划算。

---

## 2. 三个写者（这是全部设计的支点）

之前的设计错在"让模型维护状态块"——不确定、要花 token、会漂移。正确的做法是**按来源分权**：

| 写者 | 写什么 | 为什么是它 | 模型能否改 |
|---|---|---|---|
| **harness** | 事实：改了哪些文件、跑了什么命令、测试结果、副作用 | 它观察得到，且不带主观判断 | **不能**（只读） |
| **模型** | 判断：方案选择、被否决的备选、遗留问题 | 只有它能做价值判断 | 能，但必须引用事实 id |
| **人** | 约束：不许动什么、验收标准、优先级 | 这是人的权力，不是模型的 | **不能**（只读） |

一条硬规则：**事实与判断必须物理分离，不能存同一个文件。** 混在一起，判断会被当成事实继承，而判断是可能错的——这正是之前批评"摘要链只有放大误差的通路"的根源。

---

## 3. 文件布局

```
<repo>/.pi/state/
  plan.md                 # 计划与进度（项目级，跨会话）
  decisions.md            # 判断记录（项目级，跨会话）
  constraints.md          # 约束（项目级，人写为主）
  facts/
    <sessionId>.jsonl      # 事实日志（会话级，只追加）
  handoff.md              # 交接包（自动生成，可手补）
  .gitignore 建议         # 默认建议把 facts/ 和 handoff.md 排除
```

为什么这样分：

| 文件 | 生命周期 | 理由 |
|---|---|---|
| `plan.md` / `decisions.md` | 跨会话、随项目 | 一个项目的计划不该因换会话而消失 |
| `facts/<session>.jsonl` | 会话级 | 避免单文件无限增长；跨会话合并靠注入时聚合 |
| `constraints.md` | 长期 | 约束是项目级的，通常比会话长命 |
| `handoff.md` | 每次生成 | 一次性的交接视图，随时可重建 |

---

## 4. `plan.md`：完成状态由条件推导，不由模型声明

这是本文最重要的机制。

### 4.1 格式

```md
# 计划 v2（2026-10-30 14:20）

## s1 复现问题
done_when: cmd_exit0:"npm test -- auth"
status: done（f_7b1）

## s2 定位根因
done_when: manual
status: done（f_7c2, D1）

## s3 在中间件统一校验
done_when: test:"auth.spec.ts" 且 file_contains:"src/auth.ts":"rateLimit("
status: **blocked** — auth.spec.ts 仍有 2 个失败（f_7e5）

## s4 补测试
done_when: test:"auth.spec.ts"
status: pending

## 变更记录
- v1→v2（D2）：把"改 8 个 handler"改成"改中间件"，因为 8 处重复
```

### 4.2 关键点：`done_when` + harness 求值

- 条目由**模型**写（包括 `done_when`）
- `status` 由 **harness** 求值后写入，模型不能直接改
- 支持的判定器（确定性，无模型参与）：

| 判定器 | 语义 |
|---|---|
| `cmd_exit0:"<cmd>"` | 存在一条对应 cmd fact 且 exit=0（最近一次） |
| `test:"<suite>"` | 对应 test fact 且 failed=0 |
| `file_contains:"<path>":"<pattern>"` | 当前文件内容含该模式 |
| `file_exists:"<path>"` | 文件存在 |
| `git_clean:"<path>"` | 该路径无未提交改动 |
| `manual` | 需人确认（配置可禁，默认仅对"定位/判断"类步骤开放） |

**为什么不让模型直接标 done**：模型有强烈的"宣布完成"倾向——它会把"我改完了"当成"问题解决了"。让 `status` 从可验证条件推导，等于把"完成"的定义权从模型手里拿走。这是本设计里唯一一处**故意不信任模型**的地方。

### 4.3 求值时机

- 当前步骤的 `done_when`：**每轮组装时**求值一次（只算当前 + 下一步，不算全部，避免开销）
- 前一步：在步骤转换时求值一次，此后不再重算（结果写入 status）

`manual` 类步骤不自动推进，需要在注入块里显示"等待确认"。

---

## 5. `facts/*.jsonl`：harness 独占的事实日志

### 5.1 只追加，永不修改

```jsonc
{"v":1,"id":"f_7a3","ts":"2026-10-30T14:12:03+08:00","kind":"edit",
 "path":"src/auth.ts","before":{"sha":"a1b2"},"after":{"sha":"c3d4","lines":842},
 "hunks":3,"step":"s3"}
{"v":1,"id":"f_7b1","ts":"...","kind":"cmd","cmd":"npm test -- auth","exit":1,
 "durationMs":4200,"output":{"lines":812,"blob":"blobs/f_7b1.log"},"step":"s3"}
{"v":1,"id":"f_7c2","ts":"...","kind":"test","suite":"auth.spec.ts","passed":8,"failed":2,
 "firstFailure":"expected 401 got 200 (auth.spec.ts:44)","step":"s3"}
{"v":1,"id":"f_7d0","ts":"...","kind":"sideEffect","sub":"server","cmd":"npm run dev",
 "pid":1234,"port":3000,"status":"running"}
{"v":1,"id":"f_7e5","ts":"...","kind":"git","branch":"feat/ratelimit","dirty":["src/auth.ts"]}
```

`kind` 取值：`read` / `edit` / `write` / `cmd` / `test` / `git` / `install` / `sideEffect`

### 5.2 为什么单独放 jsonl 而不是写进 markdown

- 机器要查（求值 `done_when`、核对 sha）→ 结构化
- 人不需要读全量（读摘要即可）→ 不进 markdown
- 只追加 → 并发安全、可审计、崩溃可恢复（跳过残缺末行）

### 5.3 界面上的可见性

`facts` 不直接给人看，但 `plan.md` 的 status 行会引用 fact id（`f_7b1`），人点进去（或 grep）就能查到"这条完成状态凭什么"。

---

## 6. `decisions.md`：判断记录，必须带"可推翻条件"

### 6.1 格式

```md
## D3 · 在中间件统一校验，不在每个 handler 里重复
- 引用: f_7a3, f_7b1
- 备选: 每个 handler 内联校验 —— 否决：8 处重复，改动要动 8 个文件
- 结论: 在 authMiddleware 里做，handler 只读 req.user
- 可推翻条件: 若需要按路由差异化策略，改为策略表
- 状态: 生效（2026-10-30）

## D4 · 不改换 Redis 客户端（尝试后放弃）
- 引用: f_7c2, f_7e5
- 尝试: 换成 ioredis 的 sentinel 模式
- 否决原因: 本地无 sentinel 环境，测试无法复现；回滚（f_7e5）
- 可推翻条件: 若有 sentinel 环境
```

### 6.2 三个必填字段

| 字段 | 为什么必须 |
|---|---|
| **引用**（fact id） | 让判断可追溯到证据。没有引用的判断等于传闻 |
| **否决的备选 + 原因** | 这是最容易被丢弃、也最贵的信息。丢了它，agent 会重试同一个死路 |
| **可推翻条件** | 让后续知道"什么时候该重新考虑"。没有它，结论会变成不能质疑的教条 |

### 6.3 写入时机

步骤结束时**强制**：没有新增 decision 就不允许进入下一步（配置 `requireDecisionPerStep`，默认对"有代码改动"的步骤开启）。

理由：不强制就永远不写。这是唯一能保证 I4（结论先行）成立的手段——2B 的 `conclusionRef` 直接指向这里的条目 id。

---

## 7. `constraints.md`：只有人能生效

```md
# 约束

## 生效
- C1 [人] 不要修改 migrations/ 下任何文件（历史迁移已上线）
- C2 [人] 用 pnpm，不要用 npm 生成 lockfile
- C3 [人] 验收标准：auth.spec.ts 全绿 + 手动验证 429 返回

## 待确认（模型提议，未经人确认不生效）
- C4 [模型] 建议不引入新依赖
```

规则：
- `## 生效` 区只有人能写。harness 拒绝模型写入该区（写进去会被丢弃并记一条 `rejected_write`）
- `## 待确认` 区模型可追加，注入时**明确标为"未确认，仅供参考"**
- 注入块里只放生效区，且硬上限 10 条（超了按时间倒序截断，并在注入块标注"还有 N 条"）

---

## 8. 注入策略（缓存约束下的关键决定）

### 8.1 只在尾部注入，大小恒定

```
[尾部注入块 · 目标 500 token，硬上限 800]

[state]
plan: s3/7 blocked · 当前 s3「在中间件统一校验」
  done_when: test:auth.spec.ts 且 file_contains:src/auth.ts:"rateLimit("
  ✗ auth.spec.ts 仍 2 failed（f_7e5）
constraints: C1 不改 migrations/ · C2 用 pnpm · C3 验收=测试全绿+手动 429
recent: f_7a3 edit src/auth.ts(+38/−12) · f_7e5 test 8✓2✗
open: D3 的"按路由差异化"待定 · [stale] middleware.ts 已被改，f_6a1 的 sha 过期
[详情: .pi/state/]
```

三条硬约束：

| 约束 | 理由 |
|---|---|
| 只在**尾部**注入 | 插在历史中间或 system 附近会让后续前缀全部失效 |
| 大小**恒定** | 否则注入块本身成为新的膨胀源（Letta 的 memory block 就是这么烂掉的） |
| 只取**活跃**部分 | 当前+下一步、全部生效约束、最近 3 条事实、未决判断、最多 3 条 stale |

### 8.2 恒定性怎么保证

注入块由函数生成，输入是"活跃集"，输出长度与轮数无关：

```
active = { currentStep, nextStep, activeConstraints[0:10],
           recentFacts[-3:], openDecisions, staleWarnings[0:3] }
inject = render(active)      # 500 ± 150 token
```

**验收里有一条专门测这个**：注入 token 的 P95/P50 < 1.3（不随轮数增长）。

### 8.3 为什么不放 system prompt

因为 system prompt 是前缀的头。往里塞每轮变化的内容 = 每轮把整个缓存废掉。这是阶段 0 §6.3 的缓存抖动检测要抓的典型错误，本设计主动规避。

---

## 9. 校验与漂移处理

状态文件最大的风险不是写得不好，是**与现实脱节**。四处校验：

| 时机 | 校验什么 | 失败处理 |
|---|---|---|
| 每轮组装 | 当前 step 的 `done_when` | status 重算，可能从 done 退回 blocked |
| 每轮组装 | `recentFacts` 里引用的 path 的 sha | 标记 `[stale]`，不删除记录 |
| 步骤转换 | 前一步的全部 `done_when` | 写回 status；若与模型声称不符，记 `done_claim_mismatch` |
| 启动 | `facts` 末行完整性、plan 语法 | 跳过残缺行；语法错误则保留旧版并告警 |

**标脏而不是修正**：sha 变了说明世界变了，但"变成了什么"只有重新观察才知道。harness 不猜，只标 `[stale]`，让模型决定要不要重读。修正等于替模型做判断，那是越权。

---

## 10. 冷启动与 handoff

### 10.1 唯一硬验收：不给对话历史，只给 state

这是判断"状态是否真的外部化了"的唯一方法。流程：

1. 任务做到第 60%，杀掉进程
2. 新会话启动，只加载 `.pi/state/`（不加载任何对话历史）
3. 生成 `handoff.md` 作为第一条注入
4. 继续做，看能否完成

### 10.2 `handoff.md` 的结构（自动生成，可手补）

```md
# 交接 · feat/ratelimit · 2026-10-30T14:30

一句话: 给 auth 中间件加限流，卡在测试的两个失败用例上。

当前: s3（中间件统一校验）— blocked
已做: s1 复现（f_7b1）· s2 定位（D1）· s3 已改 auth.ts（f_7a3）
未做: s4 补测试 · s5 手动验证 429
已知坑: D4 换了 Redis 客户端导致环境不可复现，已回滚
下一步: 修 auth.spec.ts:44 的 401 断言
注意: C1 不改 migrations/ · C2 用 pnpm
```

生成规则：全部来自 plan / decisions / facts / constraints 的派生，**不引入新信息**。人可以手补"隐含背景"，但要标 `[人]`。

### 10.3 与 compaction 的关系

compaction 之后，历史被摘要替代，但 state 完整保留。**这就是 state 的真正价值**：压缩是有损的，state 是无损的结构化视图。两者不是替代关系——压缩负责"对话怎么变短"，state 负责"事实不会丢"。

---

## 11. 与阶段 2B 的接口

| 2B 需要的 | 由本文提供 |
|---|---|
| `conclusionRef`（I4 的证据） | `decisions.md` 的条目 id（如 D3） |
| 判断"该步骤是否已收尾" | `plan.md` 的 status（由 `done_when` 推导，非模型声称） |
| recall 时的"世界是否已变" | `facts` 里的 path + sha |

**所以阶段 3 必须在 2B 之前或同时上线。** 如果先上 2B 而 state 没做，I4 就只能靠"模型说有结论"来判断——那等于没有判断，遮蔽会在证据被使用之前把它丢掉。

---

## 12. 人类编辑与冲突

### 12.1 人能改什么

| 文件 | 人可改 | 说明 |
|---|---|---|
| `plan.md` | ✅ 增删条目、改 done_when、调顺序 | 计划是人的权力 |
| `constraints.md` 生效区 | ✅ | 只有人 |
| `decisions.md` | ✅（追加为主，可标"覆盖"） | 人覆盖时加 `[人·覆盖]` |
| `facts/` | ❌ | 事实不可编辑，只能追加新观察 |
| `handoff.md` | ✅（标 `[人]`） | 补充隐含背景 |

### 12.2 冲突处理：人优先，但必须显式提示

人改了 plan 但 harness 的 facts 显示不同状态时：

```
[state]
⚠ 人已把 s3 标为 done，但 done_when 未满足（auth.spec.ts 仍 2 failed · f_7e5）
  按人的标记继续，还是先修测试？
```

**不做冲突提示 = 人的修正会被静默忽略**，这是这类系统最常见的失效方式：人以为自己改了，模型按旧状态继续。

### 12.3 写保护

- 模型写 `constraints.md` 的生效区 → 丢弃 + 记 `rejected_write`
- 模型改写 `plan.md` 的 status 字段 → 丢弃（status 只由 harness 写）
- 模型删 `decisions.md` 的历史条目 → 拒绝（只允许追加与标"已推翻"）

---

## 13. 配置

| key | 默认 | 说明 |
|---|---|---|
| `state.dir` | `.pi/state` | 目录 |
| `inject.budgetTokens` | `500` | 目标 500，硬上限 800 |
| `inject.maxConstraints` | `10` | |
| `inject.recentFacts` | `3` | |
| `inject.staleWarnings` | `3` | |
| `plan.evaluateOnAssemble` | `true` | 每轮求值当前+下一步 |
| `plan.allowManualDone` | `true` | `manual` 判定器是否允许（仅判断类步骤） |
| `decisions.requirePerStep` | `true` | 有代码改动的步骤必须有 decision |
| `facts.retentionDays` | `30` | 超期归档为 .gz |
| `constraints.hardLimit` | `15` | 超了拒绝新增并提示精简 |
| `redact` | 见阶段 2 §3.5 | facts 落盘前脱敏 |

---

## 14. 度量

| 指标 | 定义 | 目标 |
|---|---|---|
| `inject_tokens_p95/p50` | 注入块 token 的分布 | 比值 < 1.3（恒定性） |
| `stale_rate` | 标脏条目 / 总引用条目 | < 20%（高说明校验太频繁或 world 变化太快） |
| `done_claim_mismatch` | 模型声称完成但 `done_when` 不满足的次数 | **必须为 0**（正确性断言） |
| `constraint_violation` | 违反生效约束的次数 | **必须为 0**（最重要的质量指标） |
| `cold_start_success` | 仅凭 state 继续的任务成功率 | ≥ 80% |
| `decision_reuse` | 后续引用了多少条 decision | 仅观测（低说明判断没被复用，可能在重复试错） |
| `human_edit_count` | 人修改次数 | 仅观测（长期为 0 说明可编辑是摆设） |
| `rejected_write` | 模型越权写被拒次数 | 仅观测（高说明提示词没讲清楚权限） |

`done_claim_mismatch` 和 `constraint_violation` 是断言型指标，不是优化目标——非 0 就是 bug。

---

## 15. 验收

1. **冷启动**：10 个任务各做到 60% 后杀进程，新会话只加载 state 继续，成功率 ≥ 80%
2. **约束遵守**：20 个任务中注入 3 条约束，违反次数 = 0
3. **done 真实性**：`done_claim_mismatch = 0`；抽查 10 个 done 条目，全部有 fact 支撑
4. **注入恒定**：50 轮会话里 `inject_tokens` 的 P95/P50 < 1.3
5. **人改生效**：人改 plan 后，下一轮注入块反映修改；与 facts 冲突时出现显式提示
6. **越权拦截**：模型尝试写 constraints 生效区 / 改 status / 删 decision，全部被拒并计数
7. **崩溃安全**：facts 末行残缺时能正常加载
8. **脱敏**：facts 与注入块中无明文密钥

---

## 16. 失败模式

| 症状 | 根因 | 处理 |
|---|---|---|
| 模型不写 decision | 提示词没约束住 | `requireDecisionPerStep` 强制；没有 decision 不允许进下一步 |
| 模型忽略 state，还是靠对话历史 | state 位置太靠前 / 太长 | 放在尾部最后；压缩到 500 token；冷启动测试会暴露 |
| 注入块越用越大 | 活跃集定义太宽 | 硬上限 + 只取最近 N 条；用 P95/P50 监控 |
| plan 变成谎言堆 | status 被模型改写 | status 只由 harness 写；越权写入丢弃 |
| 人改了没生效 | 无冲突提示 | §12.2 的显式提示是硬要求 |
| facts 无限增长 | 没有归档 | `retentionDays` + gz 归档 |
| 冷启动失败 | handoff 缺关键背景 | handoff 允许人补 `[人]` 段；失败案例要复盘补字段 |
| 每轮求值开销大 | 对全部 step 求值 | 只算当前 + 下一步 |

---

## 17. 实施步骤（六步，约 5–6 天）

| 序号 | 交付 | 验收 |
|---|---|---|
| 1 | 目录与最小 schema：`plan.md` + `constraints.md` + `facts` 三种 kind | 手工建文件能被解析 |
| 2 | harness 写 facts（挂在工具调用后） | edit/cmd/test 三类事实正确落盘，脱敏生效 |
| 3 | `done_when` 求值器 + status 写回 | 5 个判定器各有单测；模型改 status 被拒 |
| 4 | 注入块生成 + 恒定性测试 | 50 轮回归 `inject_tokens` P95/P50 < 1.3 |
| 5 | `decisions.md` 写协议 + 强制校验 | 缺 decision 时无法进入下一步 |
| 6 | `handoff.md` + 冷启动恢复测试 | 10 个任务成功率 ≥ 80% |

第 6 步是真正的验收：**在它通过之前，前面五步都只能算"写了一些文件"**。

---

## 18. 附录 A：为什么 `done_when` 是本设计的核心

回顾之前对摘要式压缩最重的一条批评：**结论成为一等公民，却没有纠错机制，只有放大误差的通路**。

`done_when` 是对这条批评的直接回答：

| 问题 | 摘要式压缩 | 本设计 |
|---|---|---|
| 谁定义"完成" | 模型自己说 | 可验证条件 |
| 错了怎么发现 | 发现不了 | 每轮重算，能退回 blocked |
| 依据是什么 | 无 | fact id，可追溯到命令输出与 sha |

同理，`decisions.md` 的"可推翻条件"字段，是对"结论不可质疑"的修补：每条判断都自带失效条件，后续模型知道什么时候该重新考虑，而不是把三个月前的决定当教条。

这两条加起来，才让"外部化状态"不只是"把笔记写成文件"，而是**一个能自我校验、能被推翻的状态层**。

## 19. 附录 B：与 TODO.md 的差异（诚实版）

| | TODO.md | 本设计 |
|---|---|---|
| 记下一步做什么 | ✅ | ✅ |
| 完成状态可验证 | ❌（模型自己勾） | ✅ `done_when` 求值 |
| 失败与被否决方案 | ❌ | ✅ D4 这类条目 |
| 约束不可遗忘 | ❌ | ✅ 生效区 + 越权拦截 |
| 冷启动恢复 | ⚠️ 看人写得多细 | ✅ handoff 自动生成 |
| 复杂度 | 一个文件 | 四个文件 + 求值器 + 注入器 |

如果你的任务都是"一次对话内能做完"的，上面四行 ✅ 一个都用不上。这套东西的成本是真实的（实现 5–6 天，后续维护），收益只在长任务、跨会话、多人协作的场景里兑现。
