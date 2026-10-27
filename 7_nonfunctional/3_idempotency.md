# 幂等与重试（Idempotency & Retry）

> 一句话：**opencode 对"同一件事做两次"的回答是"铸 id + 第一次准入获胜"——客户端铸造全局唯一消息 id，服务端双层 reconcile（inbox 表 ?? 已投影消息）让重试撞上已准入项时返回已有结果、忽略新负载；重试侧则是一张分类矩阵（isRetryable × 超时专项预算 × 恢复预算）加上尊重 retry-after 但 15 分钟封顶的指数退避——at-least-once 是诚实的边界，工具副作用不承诺 exactly-once。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，sed 抽查 ≥14 处）。
> 权威出处：`AGENTS.md` "Session Core" 节（idempotent admission 官方约束）+ `packages/core/src/session/inbox.ts:154`（"First admission wins" 注释原文）。
> [返回 README](README.md) | 前置 [../6_modules/4_session.md](../6_modules/4_session.md) §4（准入流）/ [../6_modules/5_inbox-coordination.md](../6_modules/5_inbox-coordination.md)（inbox 全景）/ [../6_modules/3_model-protocol.md](../6_modules/3_model-protocol.md) §8（isRetryable 分类）
> 本篇是**幂等视角的收口**：双层 reconcile 的会话流细节见 4_session §4，客户端铸 id/outbox 见 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §1，重试预算三本账见 [5_resource-limits.md](5_resource-limits.md) §3——只引用不重写。本篇新增价值：**把散在 8 个文件里的"重复检测"收进一个统一框架**（谁铸 id / 谁对账 / 谁退避 / 谁数数），并推演"第一次准入获胜"的语义推论。

---

## 0. 恐惧清单（这个产品怕什么）

```
恐惧源：网络不可靠 + 进程会死 + provider 会限流 —— 三者共同制造"同一逻辑动作发生两次"
├─ 恐惧 1：网络超时后客户端重发 prompt → 服务端双准入 → 模型回答两遍
│    └─ 防线：客户端铸 message id + 服务端 reconcile 首准入获胜（§2/§3）
├─ 恐惧 2：客户端重试撞上"已准入甚至已投递"的项 → 覆盖用户编辑后的内容
│    └─ 防线：重试负载被整体忽略（inbox.ts:154 "First admission wins"），
│      乐观 UI 只对新 id 回滚已确认状态（data.ts:1614-1617，§3.3）
├─ 恐惧 3：LLM provider 429 限流 → 全体重试同步共振 → 限流风暴
│    └─ 防线：jittered 指数退避 + retry-after 尊重但 15min 封顶（§5）
├─ 恐惧 4：进程崩溃后恢复，把"已执行到一半的工具"再执行一遍
│    └─ 诚实的答案：不防。恢复是 at-least-once（restart.ts:52-53 原文），
│      靠恢复指令约束模型行为（"without repeating completed work"）（§7）
├─ 恐惧 5：事件流断线重连 → TUI 重放已渲染的流式文本 → 屏幕重复
│    └─ 防线：fragment 对账器前缀比对（§6.1）
└─ 恐惧 6：重试掩盖真 bug —— 永远"暂时性"的失败被无限重试
     └─ 防线：isRetryable 把不可重试类一票否决 + 各预算封顶（§4/§5）
```

---

## 1. 幂等键体系：谁铸造 id

幂等的第一性问题：**重试方必须能用同一个键识别"这是同一次意图"**。opencode 的键全部是客户端铸造、服务端只认不造（服务端只在键缺席时兜底补一个，此时幂等面消失）：

| 键 | 谁铸 | 格式与唯一性来源 | 实码 |
|---|---|---|---|
| user message id | **客户端**（app composer / TUI / SDK） | `msg_` + `ascending()` | `session.ts:149` `input.id ?? SessionMessage.ID.create()`；schema `session-message.ts:23-28` |
| assistant message id | 服务端 runner | 同上（每次重试轮次可换新，见 §4 Continue） | `llm.ts:284` 等 `SessionMessage.ID.create()` |
| inbox item id | 同 message id（inbox 行 id = message id） | 同上 | `inbox.ts:296-298` 投影直用 `request.id` |
| 事件 seq | 服务端 bus（每 aggregate 单调） | DurableEnvelope `{aggregateID, seq, version}` | `event.ts:34`；4_session §4 事件序 |
| execution claim | 服务端（session 维度 busy 标记） | session_inbox 之外的执行占位表 | `execution/restart.ts:47-50` 头注释 |
| retry 复用 id | **客户端 retry 状态**（同 agent+model 才复用） | 提交时 `retryID ?? create()` | `app/composer/submit.ts:302-314` |

`ascending()` 的唯一性机制（`schema/src/identifier.ts`，全文 36 行实测）：

```typescript
// identifier.ts:16-33（节选）
// 时间戳毫秒 × 0x1000 + 进程内计数器 → 12 hex 字符前缀（时间可排序）
const current = BigInt(timestamp) * 0x1000n + BigInt(counter)
// 后 14 字符：crypto.getRandomValues 的 62 进制随机
const bytes = crypto.getRandomValues(new Uint8Array(length - 12))
```

**唯一性 = 毫秒时间戳（升序，B-tree 友好）+ 进程内同毫秒计数器 + 14 字符 CSPR 随机**。跨进程碰撞概率由随机段兜底（62^14 ≈ 10^25）；ascending 前缀让 inbox 的 `orderBy(asc(enqueued_seq))` 之外还有第二重时间序可用。`descending()` 取反码——用于想按时间倒序扫描的表。事件 id 与消息 id 可互转（`session-message.ts:28` `fromEvent: evt_→msg_` 前缀替换）。

关键推论：**幂等键是端到端的**——app 铸的 `msg_xxx` 一路穿过 client 乐观层、HTTP、server handler、inbox 表、session_message 表，始终是同一个字符串。断线重连后客户端拿着同一个 id 重发，全链路任何一层见过它都能认出来。

### 1.1 客户端 retry 状态如何复用 id

app composer 的提交侧有一个独立的"重试上下文"（`app/composer/submit.ts:302-314`）：adapter 状态里存着上次失败的 `{id, agent, providerID, modelID, variant}`，用户在同一 agent+model 下重试时**复用旧 id**：

```typescript
// submit.ts:302-314（节选）
const retry = input.adapter.state.retry.current()
const retryID =
  retry &&
  retry.agent === selection.agent &&
  retry.providerID === selection.model.providerID && ... 条件全匹配
    ? retry.id
    : SessionMessage.ID.create()
```

语义：重试 = 同一意图的再表达 → 同一键；换了 agent 或模型 = 新意图 → 新键。配合服务端首准入获胜，用户狂点重试按钮也不会产生重复消息。同文件 `:401` `restored.target.retry.set(...)` 在提交后记录这个上下文。

### 1.2 事件侧的 echo 确认

client 乐观层的"服务端已确认"信号不是靠响应体，而是靠事件 echo：`data.ts:793-799` 收到 inbox 事件时检查 `input.has(sessionID, inboxID)`——本地点过的 id 从服务端事件里回来了，即确认。app 层的 `session.admitted(messageID)`（`app/session/composer/adapter.ts:40`）组合两处：

```typescript
admitted: (messageID) => data.session.input.has(id, messageID) || !!data.session.message.get(id, messageID)
```

submit 失败路径据此裁决（`submit.ts:575`）：`if (messageID && session.admitted(messageID)) return`——失败时若消息其实已被服务端确认，**不回滚、不报错**，把已确认状态当作事实。这就是 outbox 的核心职责：已确认的不回滚（5_prompt-chain §1 已画时序）。

---

## 2. reconcile 双层对账深读

[4_session.md](../6_modules/4_session.md) §4 已画准入时序；本节下钻两层的分工与"第一次准入获胜"的语义。入口在 `Session.prompt`（`session.ts:151`，`uninterruptibleMask` 内）：

```
Session.prompt(sessionID, input)              session.ts:145-178
└─ admission.reconcile({id, sessionID,        :151-157  ← 会话层：问一次
    type:"user", delivery})
   └─ SessionInbox.reconcile                  inbox.ts:155-167
      ├─ 第一层：find(db, id)                 :161  inbox 表查未投递行
      │    └─ SessionInboxTable 主键命中 → 还有 pending 项
      ├─ 第二层：promotedFromMessage(...)     :162  ↘ 已投递则从投影反解
      │    ├─ SessionMessageTable 查同 id     :119-124
      │    ├─ row.session_id !== sessionID    :126  跨会话 → LifecycleConflict
      │    │  或 type ∉ {user,synthetic}            跨类型 → LifecycleConflict
      │    └─ decodeMessage 反解回 Info       :128-142（保留原 created 时间）
      ├─ existing === undefined → 未见过     :163  返回 undefined（调用方走 prepare+admit）
      ├─ compaction 或 !matches(existing)    :164-165 → LifecycleConflict
      └─ 命中 → 返回 existing                :166  ★ 负载被忽略
```

两层的分工：**第一层管"还在排队"的重复，第二层管"已经变成消息"的重复**。因为 inbox 表是 pending-only（投递即删行，`inbox.ts:324-336` projectDelivered 的 delete 即"表只存未消费工作"），单查 inbox 表会在"已投递"窗口漏检——第二层从 `session_message` 投影反解补上这个窗口。这是必要的：重试往往发生在网络超时之后，而超时期间消息很可能已被投递甚至已被模型回答。

`admit`（`inbox.ts:169-202`）在此基础上加一道投影竞态兜底：发布 `InboxEnqueued` 时若 projector 撞主键（`projectAdmitted` 的 `onConflictDoNothing` + 空返回 → `Effect.die(LifecycleConflict)`，`inbox.ts:293-316`），`catchDefect` 回读 `find` 再返回已存行（`:191-197`）——**并发双 admit 也只有一个赢**。

### "第一次准入获胜"的推论

官方约束（AGENTS.md "Session Core" 节原文）："the first admission wins and the retried payload, metadata, and delivery mode are ignored, whether the item is still pending or already delivered"。

- **是好事**：客户端可以无脑重试。网络层怎么重发都不会产生第二条消息、不会覆盖用户在服务端已确认的状态；app 的失败回滚也以此为准——只有"本调用新铸的 id 且未被服务端 echo 确认"才回滚乐观行（`client/solid/data.ts:1613-1617` 注释原文："A retry may reuse an ID that is already rendered — and possibly already durable. Admit optimistically only for new IDs so a failed retry cannot roll back acknowledged state"；回滚条件 `:1650-1655` `if (fresh && outbox.delete(id)) retractLocal(...)`）。
- **也是坑**：重试改 payload **静默无效**。测试钉住这一点（`core/test/session-prompt.test.ts:679-699`）：同 id 先 prompt "Fix the failing tests" 再 prompt "Delete the failing tests"，返回的第一条，`retried.payload.text` 仍是 "Fix"，`admittedCount === 1`。如果客户端 bug 让"编辑后重发"复用了旧 id，用户的编辑会无声丢失且无报错——唯一线索是返回的 payload 与发送的不一致。同族测试：`:703`（delivery 模式被忽略）、`:940`（metadata 被忽略）、`:1005`（synthetic 重试不看 payload）。
- **边界即冲突**：同 id 跨会话、跨类型是 LifecycleConflict（`inbox.ts:164-165`），映射为 `PromptConflictError`（`session.ts:172`）——键碰撞不给静默吞，给显式错误。compaction 不参与通用 reconcile（其冲突规则特化：一会话至多一个 pending compaction，`admitCompaction` `inbox.ts:204-228`，详见 [5_inbox-coordination.md](../6_modules/5_inbox-coordination.md)）。

---

## 3. 重试矩阵：全仓重试点实测

同一句"重试"，在仓库里至少有六种形态。逐个列"重试什么 / 几秒 / 几次 / 幂等靠什么"：

| # | 重试点 | 重试什么 | 间隔与次数 | 幂等靠什么 | 实码 |
|---|---|---|---|---|---|
| 1 | **Step 重试**（LLM 调用失败） | 整个 llm.stream 步骤 | 2s 指数起、10s 封顶、jitter、最多 10 次；超时类仅 3 次 | **重放历史**：每步从 reloaded history 重建请求（AGENTS.md "Each step continues from reloaded history"）；无 durable 输出才 Retry，已开输出则 Continue（换新 assistant id 续写） | `runner/retry.ts:45-60`、`runner/step.ts:189-203, 266-270`、`runner/llm.ts:276-292` |
| 2 | **HTTP 准入重发一次** | composer 的 prompt POST | 无间隔、**恰好 1 次**（`.catch(() => 再发一次)`） | 铸 id + 服务端 reconcile（§2） | `app/composer/submit.ts:545` |
| 3 | **客户端 util retry** | 任意 transient 网络操作 | 500ms 起 ×2 指数、10s 封顶、默认 3 次 | 无内置幂等——`retryIf` 由调用方定 | `util/src/retry.ts:36-56` |
| 4 | **TUI 事件流重连** | SSE/WS 订阅 | 固定 250ms 循环、无限次（直到 footer 关闭/abort） | attempt 代际淘汰 + hydrate 全量对账 + fragment 去重（§6.1） | `tui/mini/stream-v2.transport.ts:1528-1539` |
| 5 | **崩溃恢复 resume** | 孤儿 claim 的整个 turn | 无退避（boot 时 sweep 一次）；每 turn 最多 10 次、durable 计数 | at-least-once（§7）；恢复指令约束模型 | `execution/restart.ts:33, 77-98, 221-228` |
| 6 | **websearch provider 降级轮换** | 换下一个搜索 provider | cooldown 按 429 的 retry-after；随机策略下逐个尝试 | 无副作用（只读查询），不需幂等键 | `core/src/websearch.ts:200-210` |

要点逐条：

**#1 的分类闸门**。`isRetryable`（`ai/provider-error.ts:81-116`，分类细节 3_model-protocol §8 已写）只回答"该类错误重试有没有意义"；step.ts 再叠两个条件：context overflow 不重试（走 compaction 恢复，`step.ts:187-190` 的 `isContextOverflowFailure` 排除）；**已有 durable 输出后的流中断**即便 isRetryable 为 false 也允许 Continue（`step.ts:191-194` 的 `recorded.outputStarted && isInterruptedStream`）——因为输出已经落库，重放历史续写是安全的。Retry 与 Continue 的分界就是幂等性的分界：**能不能安全地当作"没发生过"**。

Retry 与 Continue 在 runner 的分派（`runner/llm.ts:276-292`）还体现另一层语义差异：

```typescript
// llm.ts:277-292（节选）
Retry: (outcome) =>
  retry.wait({ decision: outcome.decision, ... }),   // 原 assistant id 重试
Continue: Effect.fnUntraced(function* (outcome) {
  yield* retry.wait({ ... })
  yield* bus.publish(SessionEvent.Synthetic, { ... CONTINUE_AFTER_INCOMPLETE_STREAM })
  assistantMessageID = SessionMessage.ID.create()    // ★ 换新 id 续写
}),
```

Retry 复用同一 assistant message id（投影上的"这一步还在重试"）；Continue 因为旧 assistant 已有部分 durable 输出，**铸造新 id 开下一段**——旧输出不可撤销，所以续写是"新消息"而非"重发"。这是幂等键体系里少有的"服务端主动换键"场景，换键本身就声明了"旧意图已部分生效"。

**#2 为什么只重发一次**。准入 POST 的失败窗口极窄（本地 HTTP，非跨网），且服务端 reconcile 让第二次发送绝对安全；不退避是因为"失败"大概率是请求恰在服务端重启窗口（`server/src/process.ts:231` 对 starting/stopping 状态回 503 + `retry-after: 1` 秒头）——一次立即重试穿过重启间隙足够，更多次该交给用户。

**#3 是兜底库**。`isTransientError` 沿 cause 链最多挖 5 层匹配 `econnreset`/`etimedout` 等（`util/retry.ts:12-30`）；"Generated clients wrap fetch failures…" 注释明说动机。它不铸键——用于幂等由被调方保证的读操作。

**#4 的代际机制**。重连循环里每次连接是一个 `attempt` 代（`stream-v2.transport.ts:1490-1539`）：旧代的 consume promise 抛错即被 `.catch(() => error)` 吸收，`current(attempt)` 检查淘汰过期代的事件；重连成功后 `hydrate({reconnect: !state.initial})` 全量对账 + `buffered.splice(0)` 回放缓冲（`:1507-1514`）——**重连不是"接着读"而是"重新对账"**。

```
重连循环（stream-v2.transport.ts:1488-1540 概形）
│
├─ connect(): 建立 SSE/WS → 等首事件 server.connected
│   ├─ consume 协程持续收事件；booting 期先进 buffered
│   ├─ hydrate(attempt, {render, reconnect})   ← 全量对账（问服务端现状）
│   ├─ buffered.splice(0) 逐个 apply          ← 对账期间积压的事件
│   └─ await consume                          ← 转入实时流
│
└─ catch（断流）→ trace "recv.reconnect" → status "reconnecting"
    ├─ input.reconnect(signal) 换新 client     （凭证/端口可能已变）
    └─ wait(250, signal) → 回到 connect()
       （attempt 代 +1；旧代事件被 current() 淘汰，不污染新视图）
```

无限次重连不设预算的原因：显示层的重连**无副作用**（每次都从对账开始），与 §3 #1 有副作用的 LLM 重试不同类——重试预算只该花在有代价的动作上。

**#5 与 #1 的账本隔离**。崩溃恢复的 10 次预算（`restart.ts:33` DEFAULT_MAX_ATTEMPTS）与 step 重试的 10 次是**两本独立账**：前者数"这个 turn 被自动拉起过几次"（durable，先记再恢复，`restart.ts:77-91` 的 `countResume` 在 resume 前落库——注释原文 "Durable before the resume runs, so a crash inside the resumed turn is counted by the next sweep and the budget cannot be dodged"），后者数"这一步内 LLM 调用失败几次"（内存，随 step 生命周期）。两本账不串联：一次崩溃恢复后的 turn 内部照样有完整的 10 次 step 重试额度。5_resource-limits §3 的"三本账"表述即含此账。

---

## 4. 退避策略实码

核心在 `runner/retry.ts:42-56`（Effect Schedule 组合子，注释原文给出了精确总账）：

```typescript
// retry.ts:42-44 注释原文：
// Exponential from 2s capped at 10s per gap, for 10 retries: 2, 4, 8, then
// 10 × 7, about 84s of waiting when every attempt fails (67–101s with jitter).
const schedule = Schedule.max([
  Schedule.min([Schedule.exponential("2 seconds"), Schedule.spaced("10 seconds")]),
  Schedule.recurs(10),
]).pipe(
  Schedule.jittered,                    // ← 防"恐惧 3"限流风暴共振
  Schedule.modifyDelay(({ input, duration: delay }) => {
    const minimum = retryAfter(input)   // RateLimit/ProviderInternal 才有
    const duration = minimum === undefined ? delay : Duration.max(delay, Duration.millis(minimum))
    return Effect.succeed(...)          // 与退避取 max：尊重 provider 但不被它绑架
  }),
)
```

四个设计点：

1. **`min(exponential, spaced)` = 指数但封顶**：间隔 2→4→8→10→10…，单次等待不超过 10s。
2. **`jittered`**：随机化打散同步重试——多 session 同时撞 429 时不会在同一毫秒齐射。
3. **retry-after 的三重尊重**（`retry.ts:31-40`）：只信 RateLimit/ProviderInternal 两类的 `retryAfterMs`；与本地退避取 `max`（provider 说等 3 分钟就等 3 分钟）；但被 `RETRY_AFTER_MAX = 15 minutes` 封顶——`:31` 注释原文："Bound provider-requested delays so a hostile or buggy retry-after cannot stall a session for hours"（恶意或 bug 的 retry-after 头不能拖死会话）。上游解析器（`ai/route/executor.ts:24-37`）接受 `retry-after-ms` 毫秒、`retry-after` 秒数或 HTTP-date 三种格式。
4. **等待可观测且精确**：`wait`（`retry.ts:97-113`）先发 `SessionEvent.RetryScheduled`（带 `at: scheduled + delay` 未来时间戳）再 sleep 剩余量——UI 能显示"下次重试在几点"，且 publish 与 sleep 之间的时钟流逝被减掉，不多等。

### 4.1 全仓退避形态对照

退避不止 step 重试一处，形态随代价升级：

| 场景 | 退避 | 实码 |
|---|---|---|
| step 重试 | 指数 2s→10s 封顶 + jitter + retry-after max | `runner/retry.ts:45-56` |
| util retry | 500ms ×2 指数 → 10s 封顶，**无 jitter** | `util/src/retry.ts:42-55` |
| TUI 事件流重连 | **固定 250ms**，不退避 | `stream-v2.transport.ts:1539` |
| websearch 冷却 | 按 429 retry-after 原值（`cooldownMillis`） | `websearch.ts:204, 216` |
| server 重启窗口 | `retry-after: 1` 秒头提示客户端 | `server/process.ts:214, 231` |

规律：**有全局限流共振风险的（step）才需要 jitter；本地瞬时操作（util）指数够用；无副作用的显示层（TUI 重连）连退避都不要，固定间隔最快恢复视野**。retry-after 的语义也分层：provider 的被 15min 封顶，自家 server 的 1 秒头是善意的排队提示，websearch 的 cooldown 原样尊重（搜索 provider 的限流窗口通常只有几秒）。

---

```typescript
// retry.ts:42-44 注释原文：
// Exponential from 2s capped at 10s per gap, for 10 retries: 2, 4, 8, then
// 10 × 7, about 84s of waiting when every attempt fails (67–101s with jitter).
const schedule = Schedule.max([
  Schedule.min([Schedule.exponential("2 seconds"), Schedule.spaced("10 seconds")]),
  Schedule.recurs(10),
]).pipe(
  Schedule.jittered,                    // ← 防"恐惧 3"限流风暴共振
  Schedule.modifyDelay(({ input, duration: delay }) => {
    const minimum = retryAfter(input)   // RateLimit/ProviderInternal 才有
    const duration = minimum === undefined ? delay : Duration.max(delay, Duration.millis(minimum))
    return Effect.succeed(...)          // 与退避取 max：尊重 provider 但不被它绑架
  }),
)
```

四个设计点：

1. **`min(exponential, spaced)` = 指数但封顶**：间隔 2→4→8→10→10…，单次等待不超过 10s。
2. **`jittered`**：随机化打散同步重试——多 session 同时撞 429 时不会在同一毫秒齐射。
3. **retry-after 的三重尊重**（`retry.ts:31-40`）：只信 RateLimit/ProviderInternal 两类的 `retryAfterMs`；与本地退避取 `max`（provider 说等 3 分钟就等 3 分钟）；但被 `RETRY_AFTER_MAX = 15 minutes` 封顶——`:31` 注释原文："Bound provider-requested delays so a hostile or buggy retry-after cannot stall a session for hours"（恶意或 bug 的 retry-after 头不能拖死会话）。上游解析器（`ai/route/executor.ts:24-37`）接受 `retry-after-ms` 毫秒、`retry-after` 秒数或 HTTP-date 三种格式。
4. **等待可观测且精确**：`wait`（`retry.ts:97-113`）先发 `SessionEvent.RetryScheduled`（带 `at: scheduled + delay` 未来时间戳）再 sleep 剩余量——UI 能显示"下次重试在几点"，且 publish 与 sleep 之间的时钟流逝被减掉，不多等。

---

## 5. 去重器家族：重复检测的三种形态

"同一个东西来了两次"在仓库里有三个家族成员，形态各异但同构（都是"已知状态 + 比对新来者"）：

### 5.1 TUI fragment 对账器（流式文本去重）

`tui/src/mini/stream-v2.fragment.ts`（83 行，纯函数 + Map 状态）。键 `messageID\0partID`；每个 fragment 存 `{text, projected}` 双轨（累计全文 / 尚待消费的投影余量）。重连回放时 `restore`（`:65-78`）做前缀比对三分类：

```typescript
if (text.startsWith(current.text))  return { type: "append", suffix }  // 新的更长 → 只补后缀
if (current.text.startsWith(text))  return { type: "covered" }         // 旧的更长 → 全是重复，丢弃
return { type: "conflict" }                                             // 分叉 → 交上层裁决
```

`delta`（`:48-63`）更精细：流式增量先在 `projected` 里 `indexOf` 找"被覆盖的块"（注释原文："Replay may start after an unseen prefix, so consume a covered chunk from anywhere in the remaining projection rather than only its start"）——重放起点不齐时也能对齐。

### 5.2 CLI renderedText 前缀比对（非交互输出去重）

`cli/src/run/noninteractive.ts:578-601`：`renderedText: Map<key, string>` 记每个 `(messageID, ordinal)` 已打印的文本；轮询投影时 `item.text.startsWith(rendered)` 才切出 `slice(rendered.length)` 增量打印，`rendered === item.text`（完全一致）或前缀不匹配（倒退/分叉）都跳过。与 5.1 同一算法，但**无 conflict 上报**——不匹配就静默丢弃增量，CLI 场景宁缺勿乱。

### 5.3 projector onConflictDoNothing（落库去重）

`core/src/session/projector.ts:183, 468`（session 表 insert）+ `inbox.ts:312`（inbox 行）：SQLite `INSERT ... ON CONFLICT DO NOTHING` + `.returning()` 空即 `SessionAlreadyProjected` die——**数据库主键做最终裁决者**。事件重放/双写竞态下，第二份静默不存在，事务回滚不产生半行。

三形态对照：

| | fragment 对账器 | renderedText | onConflictDoNothing |
|---|---|---|---|
| 层 | 显示（内存） | 显示（内存） | 持久化（SQLite） |
| 键 | msgID+partID | msgID+ordinal | 表主键（msg id 等） |
| 分叉处理 | conflict 上报 | 静默丢弃 | die → 事务回滚 |
| 语义 | **前缀去重**（流式） | 前缀去重（终态） | **整键去重**（集合） |

为什么前缀去重不能用整键去重替代：流式场景下"重复"不是整条文本重复，而是**前缀重叠**——重连回放会重发已见的前 500 字再加新 200 字，整键比对只能全收或全拒，前缀比对才能只补 200 字。反过来，落库场景下行是原子的，整键判定最便宜且由 SQLite 主键免费提供。选型随数据形态，不随偏好。

### 5.4 幂等面在投影里的最后一道

`projectAdmitted`（`inbox.ts:294-316`）值得单独看它的防御顺序：insert 前先查 `SessionMessageTable` 是否已有同 id 消息（`:287-293`，有则 die LifecycleConflict——消息都投递了还想排 inbox 行？）；insert 用 `onConflictDoNothing().returning()`，空返回同样 die（`:316`）。两道检查 + 主键约束，投影层的重复事件（重放/双订阅）没有生存空间。这与 4_session §4 的"投影幂等三层防线"（1_durability 已引）构成纵深：**准入幂等在 reconcile，投递幂等在投影，各管一段**。

---

## 6. 失效场景矩阵

| # | 失效场景 | 撞上的防线 | 残余风险 |
|---|---|---|---|
| 1 | 网络超时 → 客户端重发 prompt | 铸 id + 双层 reconcile 首准入获胜（§2） | 无——第二次发送必返回第一次的结果 |
| 2 | 重试携带编辑后的 payload | 负载被忽略（inbox.ts:154） | **编辑静默丢失**——坑面：客户端复用旧 id（§2 推论） |
| 3 | 并发双 admit 同 id | projector onConflictDoNothing + admit 的 catchDefect 回读（§5.3） | 无 |
| 4 | 多 session 同时 429 → 重试风暴 | jittered 指数退避 + retry-after 取 max（§4） | 全局无并发预算池——靠"每处局部限速足够小"（5_resource-limits §3） |
| 5 | 恶意/buggy retry-after: 86400 | RETRY_AFTER_MAX 15min 封顶（retry.ts:31-32） | 无 |
| 6 | 死 provider 每次超时几分钟 | 超时专项预算 3 次（retry.ts:58-60） | 最多约 3×超时+退避后放弃 |
| 7 | 崩溃后恢复重执行已跑工具 | **无防线**——at-least-once（restart.ts:52-53 原文："local coordination prevents concurrent drains, not repeated external side effects after a crash"） | 工具副作用可能重复；缓解仅 CONTINUE_AFTER_SERVER_RESTART 指令（restart.ts:15-16 "without repeating completed work"——对模型是约束不是保证） |
| 8 | 崩溃恢复自身 crash-loop | per-turn durable 预算 10 次，耗尽 terminalize（restart.ts:77-91） | 无——"the budget cannot be dodged"（:78-79） |
| 9 | 重试掩盖确定性 bug | isRetryable：Auth/Quota/ContentPolicy/InvalidRequest/Timeout 一票不重试（provider-error.ts:104-110）；但 UnknownProvider 重试（"transient failures are exactly the ones… no classifier anticipates"，:97-100） | 未分类的新错误形态会被重试 10 次——换取"未知瞬态能自愈" |
| 10 | 幂等键跨会话/跨类型碰撞 | LifecycleConflict → PromptConflictError（显式报错） | 无——不静默 |
| 11 | 断线重连回放旧流式文本 | fragment restore 三分类 / renderedText 前缀比对（§5） | conflict 分叉需上层裁决；CLI 侧静默丢增量 |

矩阵读法：**准入路径（1-3,10）是硬幂等；执行路径（7）诚实地停在 at-least-once**；中间层（4-6,8-9,11）各有限速/封顶/分类闸。这条断层线与 [1_durability.md](1_durability.md) §4 的结论完全一致。

值得展开的三个"明知不防"：

**#7 的工具双执行**是框架性取舍。工具调用（尤其 shell/写文件）没有跨进程幂等令牌——模型发的 tool call id 在崩溃窗口内可能已 spawn 了进程。opencode 不给工具结果做去重键，而是把恢复语义交给两层：durably 已结算的工具结果在下次模型工作前 settle 成终态（1_durability §3 注入点）；未结算的只能当作没发生，靠 CONTINUE_AFTER_SERVER_RESTART 提示模型自己判断。付费 API 类工具的重放成本由用户感知，不由系统兜底。

**#9 的 UnknownProvider 重试**是偏执的另一极。注释原文（provider-error.ts:97-100）给出理由："transient failures are exactly the ones that arrive in shapes no classifier anticipates"——分类器没见过的错误形态恰恰最可能是瞬态的，宁可重试 10 次也不把可自愈的故障当死刑。代价是新型确定性错误会被重试到预算耗尽（约 84-101s）才报错——用户多等一分半，换来的是"未知网络抖动自动恢复"的常见案例。这是重试策略的经典权衡，opencode 选了可用性侧。

**#2 的静默丢编辑**没有任何报错通道。防线完全在客户端纪律：任何"用户可编辑后重发"的 UI 必须在编辑时换新 id（或 cancel 旧项再发新项——inbox 的 cancel/steer/queue 就是为此准备的 mutation 通道，inbox.ts:230-260）。服务端对此无能为力，因为它无法区分"网络重试"与"恶意/buggy 重发改写"——首准入获胜下两者不可分辨，选边站了重试。

---

## 7. 测试覆盖实测

```
防线                        测试文件（实测存在）
────────────────────────────────────────────────────────────
首准入获胜语义              core/test/session-prompt.test.ts（1217 行）
  ├ :622  已投递项从投影 reconcile（无准入历史）
  ├ :679  同 id 不同 prompt → 第一条获胜，admittedCount===1
  ├ :703  delivery 模式被忽略
  ├ :940  metadata 被忽略
  └ :1005 synthetic 重试不看 payload
Step 重试策略                core/test/session-runner-retry.test.ts（47 行）
  ├ "stops retrying transport timeouts after three attempts"
  │   → [true,true,true,false]（超时预算 3 的直接断言）
  └ "keeps the general allowance for other transient failures"
      → 10 次全 true（预算不误伤）
util retry（瞬态匹配）      util/src/retry.test.ts（5 个 test：次数/cause 链/
                            非瞬态不重试/自定义条件）
TUI 重试 provider UI        tui/test/component/retry-provider.test.tsx
崩溃恢复预算                core/test/session-execution.test.ts（5_resource-limits §3 已引）
```

值得单记：`session-prompt.test.ts:679` 的断言方式是**同时检查三面**——返回值等于第一次、`retried.payload.text` 是旧文本、`admittedCount === 1`。它钉住的不只是"没有第二条"，还有"重试方拿回的是已准入真相"——这正是客户端敢无脑重试的契约基础。

`session-runner-retry.test.ts` 的构造也值得学：它不 mock provider，而是直接对 `SessionRunnerRetry.policy` 喂手造的 `AIError`（`TransportError{code:"Timeout"}` 与 `ProviderInternalError`），循环调用 4 次断言 `[true,true,true,false]`——把"预算"从集成行为降维成纯策略函数的单元事实，47 行覆盖两条最重要的预算线。TUI 侧的 `retry-provider.test.tsx` 则从 UI 角度钉 RetryScheduled 事件的渲染。

---

## 演进注记

对照旧卷：4_session §4（准入时序）、5_inbox-coordination（inbox 全景含 admitCompaction 特化）、3_model-protocol §8（isRetryable 分类表）、5_prompt-chain §1（客户端铸 id 与 outbox 乐观层）、5_resource-limits §3（重试预算三本账：step 10 次/超时 3 次/恢复 10 次）各有其篇；本篇为**收口层**——首次把铸 id（identifier.ts 的 ascending 实现）、双层 reconcile、六形态重试矩阵、退避组合子、三形态去重器放进"幂等键 → 对账 → 退避 → 预算"统一框架，并推演"第一次准入获胜"的静默丢编辑坑面。v1 时代的差异：submit.ts 的 `.catch` 单次重发与 data.ts 的 outbox 集合是 v2 客户端乐观层的产物；identifier 的 counter 重置逻辑（同毫秒内计数、跨毫秒清零）意味着跨进程同毫秒只靠随机段防碰撞——单机单进程模型下的够用设计。

## 相关篇

- [../6_modules/4_session.md](../6_modules/4_session.md) §4 —— 准入流与事件序（本篇 §2 的宿主）
- [../6_modules/5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) —— inbox 全景、admitCompaction 特化冲突规则
- [../6_modules/3_model-protocol.md](../6_modules/3_model-protocol.md) §8 —— isRetryable 错误分类表（本篇 §3 #1 的闸门）
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §1 —— 客户端铸 id、outbox 乐观准入、`.catch` 重发的时序位置
- [5_resource-limits.md](5_resource-limits.md) §3 —— 重试预算三本账与超时全景表
- [1_durability.md](1_durability.md) —— 崩溃安全：claim、启动 sweep 与 at-least-once 边界（本篇 §6 #7 的另一面）
