# 压缩详解（摘要替换前缀历史以释放上下文窗口）

> 一句话：**压缩 = 用一份结构化摘要（checkpoint）替换前缀历史，把模型可见窗口从"全量 transcript"切换到"摘要 + 保留的近期消息"；本篇下钻 compaction.ts 的三路触发、预算/缩减算法、原生 provider 窗口。**
> 代码基线：v2.0.30，dev 分支（e25b7abf89）；行号实测（2026-10-19）。
> 权威出处：`specs/v2/session.md`（"Compaction Rebuilds Active History" L98-104、投递词汇 L14-23、指令 epoch L96）+ 仓库根 `AGENTS.md`（Session Core 节）。
> [返回 README](README.md) | 前置 [4_session.md](4_session.md) §7（触发面总表，本篇不重复） | 控制项投递语义见规划中的 5_inbox-coordination.md（本篇只引用）

---

## 0.5 通俗完整描述（先读这段再看细节）

对话越长，窗口越满——满了之后模型既装不进新内容、注意力和缓存也在退化。**压缩就是把前面的历史折叠成一份摘要**，让对话"带着总结继续"：摘要替换旧前缀，最近的对话保留原文，模型从摘要处接着干。（revert/move/transfer/diff 四个会话操作已拆至 [7.1_session-operations.md](7.1_session-operations.md)——它们与压缩零代码耦合，2026-10 拆分归位）

三种触发方式：用户手动 `/compact`；自动——用量逼近阈值时在安全边界顺手压缩（阈值=窗口减去预留）；被动——模型请求超长被 provider 拒了，现场压缩重建同一步再试。压缩本身也是一次 LLM 调用（专门的角色、专门的 prompt："把以上对话总结为……"），输出流式实时可见（你看到的"压缩中"进度）。

两个精妙处：①**压缩是投递硬边界**——它以控制项身份进 inbox 队列，批提升到它必须停刀（压缩后说的话属于新历史，不能混进旧前缀的批次）；②**压缩推进指令纪元**——压缩后指令基线重新快照，旧的"环境变更公告"折叠进摘要，模型的世界观被整理归零。失败的压缩零副作用（历史未动，标记失败即止）；崩溃时进行到一半的压缩在重启时被判失败收场。

顺带澄清两个易混操作：**revert**（还原到某条消息——物理回退历史，暂存/确认两阶段）；**move**（会话搬家换目录——执行环境切换）。它们与压缩同住一个文件族但语义各异。

**"前缀"的精确语义**：前缀 = 消息序列（按 seq 单调编号）的早期部分——即口语的"历史对话前半段"。但"摘要替换前缀"**不是删除**：完整 transcript 永远 durable 存在（spec 原话 "The full transcript remains durable"），压缩改变的只是**模型可见历史的读取起点**——history.ts:83-91 加载时 `WHERE seq >= 压缩点seq`，被跳过的那一段就是"前缀"；压缩点本身展开成 `<conversation-checkpoint>`（摘要+近期上下文）。用"前缀"这个词还承接了**前缀缓存**的术语：请求前缀与上次一致才命中缓存（1/10 价），"保留近期、折叠早期"正是为了前缀稳定——与指令基线（6_context）"稳定在前、变化在后"同一逻辑。

```
压缩前（模型看到的窗口）                压缩后（模型看到的窗口）
┌────────────────────────────┐      ┌────────────────────────────┐
│ seq1 用户A                  │      │ <conversation-checkpoint>  │
│ seq2 模型a                  │      │   <summary> 把 seq1-5 的    │
│ seq3 工具t   ← 早期消息      │ 折叠 │   要点总结成一份摘要         │
│ seq4 模型b     （前缀）      │ ───► │   <recent-context> …</>    │
│ seq5 用户B                  │      ├────────────────────────────┤
├────────────────────────────┤      │ seq6 模型c   ← 近期消息      │
│ seq6 模型c   ← 近期保留      │ 保留 │ seq7 用户C     原样保留      │
│ seq7 用户C                  │      │ seq8 模型d                  │
│ seq8 模型d                  │      │        （下次读取起点=压缩点）│
└────────────────────────────┘      └────────────────────────────┘
       完整 seq1-8 永远在数据库（UI 可回放全部历史）——
       变的只是"模型从哪读起"（读取起点平移到压缩点）
```

一句话：**窗口治理=摘要替换前缀；三种触发、控制项身份、纪元推进——"让长对话可持续"的核心机制**。
---

## 0. 模块地图

五个文件都在 `packages/core/src/session/` 下，共 1,758 行：

```
packages/core/src/session/
├─ compaction.ts ★ 压缩引擎（988 行，本篇主角）
│   ├─ 1-102    类型与常量：Settings(41) / Trigger(53) / Outcome(59)
│   │           SHRINK_STEPS(95) / RESERVE_MIN(97) / UNKNOWN_WINDOW(99)
│   ├─ 104-182  摘要 prompt：SUMMARY_TEMPLATE(104) / SUMMARY_RULES(137)
│   │           buildPrompt(145, 首次/更新/legacy 三态) / NUDGE(178) / LEGACY_HEADING(182)
│   ├─ 184-681  Layer 主体（Effect 服务）
│   │   ├─ state(195)         Settings 状态机（auto=true, keep=15000 初值）
│   │   ├─ compact(205)       ★ 统一入口：门槛判定→选模型→分流
│   │   ├─ due(242)           auto 触发阈值判定
│   │   ├─ summarize(272)     文本摘要路径（自己的 LLM 调用）
│   │   ├─ compactNatively(330) 原生 provider 窗口路径（endpoint/trigger 两机制）
│   │   ├─ deliver(390)       ★ 发送循环：预算适配 + 拒绝缩减 + 重试
│   │   ├─ flattenAndDropOldest(471) 装不下时拍平成文本并丢最老 exchange
│   │   ├─ stream(512)        llm.stream 折叠：delta→bus、finish 分类
│   │   ├─ prepare(581)       经 hooks 的请求组装（SessionModelRequest.compaction）
│   │   ├─ fromHook(601)      hook 直接供给摘要（不调模型）
│   │   ├─ spend(621)         计费累计（UsageRecorded, source="compaction"）
│   │   ├─ publish(629)       Ended/Failed 事件发布
│   │   └─ Service.of(666)    + onInterrupt 中断语义 + spent 清理
│   └─ 683-988 模块级纯函数
│       ├─ previousCompaction(690) / splitConversation(708) / recentStart(722)
│       ├─ oldestToDrop(746)  预算装填核心（两处复用）
│       ├─ messageToText(759) 消息→recent 文本（含 skill/文件/工具截断）
│       ├─ truncateToolOutput(813, 1250 字符) / flattenMessage(829)
│       ├─ recentUserMessages(860) 原生路径保留的整条用户消息
│       ├─ estimateContext(874) / estimatePrompt(880) 上下文估算
│       └─ calculateCeiling(911) / fitOutput(939) / estimate*(950-988)
├─ transfer.ts  ★ 会话导出/导入（339 行）：export(57) / import(64)
│               isSettled(165) / sanitize(180) / sanitizeMessage(208-310)
└─ diff.ts      ★ turn 级文件 diff（138 行）：TurnRangeError(15) / turn(38-137)
```

外围协作者（本篇引用行号均实测）：

- `runner/llm.ts`：三路触发调用点（91/220/265）+ `settleStaleCompactions`(332-360)
- `inbox.ts`：`admitCompaction`(204-227)、`promote` 的控制项提队规则（492-527，注释在 493）
- `history.ts`：`Boundary` 三态(21) / `latestCompaction`(28-60) / `messageEntries`(78-108)
- `projector.ts`：Compaction 三事件投影 + epoch 推进（729-736）、RevertEvent 投影（737 起）、Moved 投影（475-493）
- `message-updater.ts`：compaction 消息三种状态落库（414-484）
- `schema/session-event.ts`：Compaction 命名空间四事件（582-640）
- `config/plugin/compaction.ts`：配置文档 → `editor.configure` 的桥（34-45 行附近）
- `model-request.ts`：`SUMMARY_OUTPUT_MAX=32000`(53)、`outputLimit` 的 compaction 分支(88-97)

---

## 1. 心智模型

**压缩 = 用摘要替换前缀历史**。完整 transcript 永远 durable 存在（spec L102 "The full transcript remains durable"）；压缩改变的只是"模型可见历史从哪读起"。压缩完成后，history 加载从最新 completed compaction 开始（history.ts:83-91 的 `gte(seq, compaction.seq)`），边界上的那条 compaction 消息在发给模型时被 `to-llm-message.ts:306-325` 展开为一条 `<conversation-checkpoint>` user 消息（`<summary>` + `<recent-context>`），之后的消息原样跟上。

```
事件流（durable，seq 递增）                     模型可见历史（投影）
════════════════════════════════════════════════════════════════════

压缩前：
  [U1][A1][U2][A2][U3][A3][U4][A4]              [sys][U1][A1][U2][A2][U3][A3][U4][A4]
  ─────────────────────────────────              （全量，越来越贵）

auto 压缩触发（估算 ≥ ceiling）：
  [U1][A1][U2][A2][U3][A3][U4][A4]
      └─ summarize(U1..A3) ──┐
                            ▼
  [C1=compaction(completed, summary+recent)]
                              [U4][A4]          [sys][<checkpoint>S1+E4</checkpoint>][U4][A4]
  ─────────────────────────────────              （窗口被"截断"到边界之后）

再次压缩（叠加）：
  [C1][U4][A4][U5][A5][U6]
                            ▼
  [C2=compaction]                            [sys][<checkpoint>S2+E6</checkpoint>]
                                              S2 覆盖 S1+E4+E5（update prompt，非增量拼接）

原生 provider 窗口（native）：
  [C1'=compaction(providerContext=加密窗口)]    [sys][窗口内消息(U U)][新消息…]
                                              provider 决定保留什么；本地不可读
```

**图例（上图的缩写）**：`U/A` = User/Assistant 消息；`[sys]` = system 段（模板+工具指导）；`C1/C2` = 第 1/2 次 compaction 的 durable 消息；`S1/S2` = **Summary**（摘要——summarize 的产物，第 1/2 版）；`E4/E5/E6` = **Entries/epoch 基线**（[6_context.md](6_context.md) §5 的指令快照——压缩推进纪元后基线重新渲染，编号即 epoch 版本号）。两个易读错点：① **指令不在 checkpoint 里**——checkpoint 展开 only `<summary>+<recent-context>`（to-llm-message.ts:316-321 实测），指令基线在 system[1] 段；`E4` 标在 checkpoint 旁表达的是"压缩点之后模型的环境认知 = 新基线 E4"——S1 管"聊过什么"、E4 管"环境是什么"，合起来是压缩后世界观的两半。② `S2 覆盖 S1+E4+E5（非增量拼接）`——第二次压缩用 update 模式的 prompt **重新总结**（上次的摘要+后续对话再压一遍），不是把 S1 原文拼进 S2——防"摘要的摘要"层层失真。

还原（revert）则是**回退到某消息**：durable 层删除边界之后的所有消息行（projector.ts:759-775），并把边界之后 assistant 触碰过的文件从快照恢复。两者都是"重写模型可见历史"的操作，但方向相反：压缩是无损摘要（transcript 保留），revert 是真删。

---

## 2. 压缩触发三路

三路都汇到 `SessionCompaction.compact(trigger)`（compaction.ts:205-240）。`Trigger`（:53-57）就是三种 reason：`auto | overflow | manual`。触发面总表（谁在哪调用）见 4_session.md §7.4，本篇只讲判定逻辑。

### 2.1 手动（manual）

`Session.compact`（session/session.ts:246）→ `admitCompaction`（inbox.ts:204-227）把一个 **compaction 控制项**放入 inbox（delivery 默认 steer；幂等：同 ID 已是 compaction 项则复用 :214，已有 pending compaction 直接返回它 :219-220——一个会话同时最多一个待执行压缩项）。runner 在安全边界处理它（runner/llm.ts:91-163）：先原子发布 `InboxDelivered + Compaction.Started(reason=manual, inputID)`（:92-95），再用 `SessionHistory.preview` 组装上下文调 `compact({reason:"manual", inputID})`（:127-138）。

投递顺序（specs/v2/session.md L18 "Delivery stops before a compaction or move control item"）：`SessionInbox.promote`（inbox.ts:492-527）里 `steers.findIndex(type==="compaction"||"move")`——**steered compaction 提队到所有 pending prompt 之前，但不会跨过一个排在前面的 steered move**（control === 0 时返回 undefined 让 runner 先处理 move；:505-509）。这正是 spec L23 "control item forms a delivery boundary" 的实现。

### 2.2 自动（auto, due）

每个 Step 开跑前：runner/llm.ts:220 `compaction.compact({reason:"auto", context: loaded})`。判定在 compact 入口 + `due()`（compaction.ts:211, 242-251）：

- `settings.auto === false` → skipped（:210，**overflow 也算自动，关掉就只剩手动**）。
- `due()` 三道闸（:242-251）：
  1. 最后一条消息已是 completed compaction → false（刚压完，让 runner 从新边界重组请求）；
  2. **最近一次实测 prompt（provider 报过 token 数的 assistant）必须晚于最近一个原生检查点**——加密原生窗口本地估算为 0，必须等一次真实响应才能测量（:247-249）；
  3. `estimateContext(context) >= ceiling` 才 due。

阈值来源（全在源码，无魔法数）：`calculateCeiling`（:911-918）= `window - max(window*10%, RESERVE_MIN=16000 if window≥32000)`；配置可覆盖——`Settings.buffer`（:44）直接减固定 token 数。Settings 由 `config/plugin/compaction.ts` 从配置文档（`compaction.auto/buffer/keep.tokens`）经 `editor.configure` 注入，热更新（`compaction.reload`）。

### 2.3 溢出恢复（overflow）

provider 在 durable 输出之前拒绝为 context overflow 时：step.ts:158-166 检测 `isContextOverflowFailure` 且 `!outputStarted`，调 `input.recoverOverflow` → runner/llm.ts:265-271 `compact({reason:"overflow"})`，成功则 `Outcome.Compacted()` 重建同一逻辑 Step。**一次性**：llm.ts:320-323 `Compacted` 分支把 `recoverOverflow=false`；第二次 overflow 终态（spec L104 "A second overflow ... is terminal"）。overflow 触发时首个预算直接打到 `estimateContext × 0.7`（compaction.ts:226-228 + `SHRINK_STEPS[0]`，:95）——因为 estimate 已被证明偏低。

### 2.4 三路时序对照

```
manual（用户敲 /compact）
  TUI/API → Session.compact(session.ts:246)
          → admitCompaction(inbox.ts:204)            [session_inbox 行]
          → execution.wake
  runner drain → advanceToStep → nextPromotable(llm.ts:86)
          → 是 compaction 项？原子发布 [InboxDelivered + Started(manual)](llm.ts:92)
          → SessionHistory.preview(不准入新输入, llm.ts:121-126)
          → compaction.compact({manual, inputID})(llm.ts:127)
          → Delta*(ephemeral) → Ended|Failed(durable) → force=false, continue(llm.ts:161)

auto（每 Step 前）
  runStep 循环体(llm.ts:216-229)
          → prepareContext + context.load（重载历史）
          → compaction.compact({auto, loaded})(llm.ts:220)
          → 入口自判：!settings.auto 或 !due → skipped，直接进正常 Step
          → due → Started(auto) 由 deliver 自己发(compaction.ts:400)
          → completed → assistantMessageID 换新 + continue（重组请求再来）

overflow（provider 拒绝）
  steps.attempt 流内(step.ts:158-165)
          → isContextOverflowFailure && !outputStarted
          → recoverOverflow → compact({overflow, loaded})(llm.ts:265-271)
          → Outcome.Compacted(step.ts:166) → recoverOverflow=false(llm.ts:321)
          → 回到 runStep 循环重建同一逻辑 Step
```

三路的差异本质上只有两点：谁发 Started（runner vs 服务自身）与预算起点（overflow 首发即 ×0.7）。

### 2.5 阈值算例

```
窗口 200k, 无配置覆盖：
  ceiling = 200_000 - max(20_000, 16_000) = 180_000（compaction.ts:917）
窗口 400k：400_000 - max(40_000, 16_000) = 360_000
窗口 30k（< 2×RESERVE_MIN）：30_000 - max(3_000, 0) = 27_000（保 10%）
配置 buffer: 50_000：ceiling = window - 50_000（:416，不再取 max）
窗口未知（limit.input=0 且 limit.context=0）：ceiling = +Infinity（:915）
  → auto 永不触发；但手动/overflow 仍可压，目标按 UNKNOWN_WINDOW=200k 算（:223-225）
```

---

## 3. 压缩执行流

`compact`（:205-240）的分流：`context.model.compaction?.type === "native"` 走 `compactNatively`，否则走 `summarize`。模型选择（:213-220）：文本路径解析 **`compaction` 专属 agent** 的 model（`Agent.ID.make("compaction")`），解析不了回落会话当前模型。预算 cap：窗口未知时按 `UNKNOWN_WINDOW=200_000` 假装（:222-225）。

### 3.1 文本摘要路径 summarize（:272-316）

源码头部注释（:253-271）就是最好的算法图：`[S1+"E9 E10"][E11…E18][E19] → [S2+"E19"]`，S2 覆盖 S1 和其后全部。

1. `splitConversation`（:708-720）+ `recentStart`（:722-744）：把消息切成 `older`（被摘要）与 `recent`（`keep=15000` token 内的最新消息，转成文本保真保留）。切点三规则：装得下就从 user 消息切（tool call/result 不拆开，:733-735）；只装得下一轮就保最新一轮；只剩一轮时全摘要——除非上次摘要留过 recent，那就只摘要"摘要的摘要"（:741-743）。
2. `prepare`（:581-598）：经 `SessionModelRequest.compaction` 组装请求（hooks 可改/可拦截，model-request.ts:428）。输出上限 `min(model, SUMMARY_OUTPUT_MAX=32000)` 再按窗口余量压（model-request.ts:88-97 + compaction.ts:939-948 `fitOutput`）。
3. **prompt**（:145-176 `buildPrompt`）：追加为最后一条 user 消息（:289-294）。三态——首次（"You MUST summarize … given to another agent to resume"）、更新（"Update the existing checkpoint … Newer history always takes precedence"）、legacy（旧模板摘要含 `## Additional Context`，要求整体重写降温，:166-170，注释注明 #48058 后移除）。模板是固定七节：Objective / Requirements / Decisions / Work State(Completed/Active/Blocked) / Next Move / Relevant Files(≤15) / Important Context。**规则明示不算 todo 系统，只是 prompt 文案**。
4. 模板校验 + 一次 nudge：回复必须含模板的某个 `##` 标题（:283-284）；没有则追加 `NUDGE`（:178-179）重试一次，仍没有 → `compaction.failed`（:304-311）。
5. hook 直接供摘要：`prepared.event.result` 存在时 `fromHook`（:601-614）完全不调模型。

### 3.2 原生路径 compactNatively（:330-378）

头部注释（:318-329）：`[window] → [U U][item]`，provider 自己挑保留什么，产出**加密的 compaction item**（只在制造它的 endpoint 上可重放）。两机制：`trigger`（请求里带 compaction_trigger，provider 返回 checkpoint 消息，本地再补 `keep` 内最新整条 user 消息，:361-367，`recentUserMessages` :860-872 只保留"完整的真实 user 消息"）与 `endpoint`（整窗口送 endpoint 压缩，provider 返回 replacement 消息组，:369-373）。前置校验 provenance 稳定（:342-349）——hook 改写了 model 路由就直接 unsupported。

### 3.3 deliver 发送循环（:390-461）——压缩的"重试大脑"

头部注释（:380-389）完整描述策略，实现逐条对应：

```
请求 ≤ target？ ──是──> 原样发
    │否
flattenAndDropOldest（拍平+丢最老 exchange）──仍不行且没被拒过──> 无限预算再拍平
    │仍不行（且被拒过）──> fail "cannot be reduced further"（:419-427）
发 send()
    ├─ 成功 → 返回 Result
    ├─ Failure（模板不合规等）→ 直接 fail（不重试，:432）
    ├─ payload-too-large 且是未改请求 → asText=true 重来（不计次，:436-439）
    ├─ too-large / context-overflow → SHRINK_STEPS=[0.7,0.5,0.35] 逐级缩目标
    │     （以首次被拒请求的估算为基数，:441-448；三级用尽 → fail）
    └─ 其他 provider 错误 → 会话 retry policy（:450-459）
```

`flattenAndDropOldest`（:471-510）把 `[上次摘要][之后的对话]` 拍成一条文本 user 消息：上一次摘要永远置首（:479-484），transcript 丢 reasoning/media/指令更新、工具输出截 1250 字符（`flattenMessage` :829-849 + `truncateToolOutput` :813-826），仍超则整轮 exchange 最老先丢（:505），头部插 `[N older exchanges omitted]` 注记（:498-499）。

### 3.4 流式与计费

`stream`（:512-571）折叠 `llm.stream`：`textDelta` → `bus.publish(Compaction.Delta)`（**:531-535，ephemeral 事件**，session-event.ts:601-608 无 replay 保证）；`stepFinish` 记 providerState + `spend` 计费（:537-541）；finish 按 normalized reason 分类，`length`（摘要顶到输出上限）与 `content-filter` 直接不可用失败（:558-564）。**每次调用完成即计费**（:616-618 注释：失败/中断的压缩也计费），`UsageRecorded(source="compaction")` 累计，Ended/Failed 事件携带总额（session-event.ts:617-618/631-632 注释：内部 usage 事件不出客户端，压缩事件重复数字供展示）。

### 3.5 事件面

```
manual:   InboxDelivered+Compaction.Started 原子对（llm.ts:92-95）→ Delta* → Ended|Failed
auto/overflow: Compaction.Started(reason="auto")（compaction.ts:399-401，manual 不发——
              runner 已在投递时发过）→ Delta* → Ended|Failed
落库：message-updater.ts 414-484
  started → append CompactionRunning 消息（id=inputID 或从事件 ID 派生）
  ended   → 找 running 更新为 completed（写入 summary/recent/providerState/providerContext）
  failed  → running 转 failed；无 running 则追加
```

### 3.6 estimatePrompt：measured/estimated 双轨锚点（:880-904）

上下文估算不是全量本地估——它以**最近一次 provider 实测的 assistant**（`hasMeasuredPrompt` :924-929：同 provider、无 error、tokens.input+cache>0）为锚点：

```
measured = anchor.tokens.input + cache.read + cache.write + output + reasoning（:899-901）
estimated = 锚点之后的消息本地估算之和（:902）
estimateContext = measured + estimated（:874-877）
```

细节：锚点自身的 output 计入 measured 但其本地工具结果不计（provider 没见过，:890-891）；跨 provider 的 token 数不可信（加密原生窗口的计数不含其代表的历史，:920-923 注释）——这正是 `due()` 测量闸（:247-249）的动机。:884-886 的 TODO 承认 image 预算只看锚点之后的图，图片密集的会话可能估高。

---

## 4. 压缩后历史视图

`history.ts` 是边界语义的家：

- `Boundary`（:21）三态：`"latest"`（模型中立读者：所有完成压缩都算边界）/ `"local"`（原始 transcript：原生窗口不算边界，本地摘要算）/ `Provenance`（具体模型：只算它可重放的原生窗口，:23-26 `replayable` + `compatible`）。
- `latestCompaction`（:28-60）：SQL 找最新 `status='completed'` 的 compaction 行；boundary 条件用 `json_extract` 拼出 `providerContext is null`（本地摘要）或 provenance 逐字段相等（:41-53）。
- `messageEntries`（:78-108）：`seq >= 边界` 加载；两条过滤——native checkpoint 之前的 system 消息丢弃（epoch 已重置，:102-106），不可重放的 native checkpoint 丢弃（:106）。注释（:99-101）解释 fork 跨检查点时 baseline 依然生效。
- 发给模型时 compaction 消息的展开在 `to-llm-message.ts:306-325`：本地摘要 → `<conversation-checkpoint>` 包裹的 user 消息；原生 → `SessionProviderContext.decode` 直接还原 provider 消息组。`status !== "completed"` 一律跳过（:308）。
- `previousCompaction`（compaction.ts:690-693）依赖同一不变量："history 从最新完成压缩加载，所以上一条压缩永远是第一条消息"。

指令 epoch 联动：`Compaction.Ended` 投影顺手 `InstructionState.advanceEpoch`（projector.ts:730-734），把 epoch 起点挪到压缩事件精确 seq（spec L96 "An instruction epoch spans completed compactions"）。

### 4.1 三种读者各取所需

```
读者                    Boundary           效果
runner（发给模型）       provenance ?? local  原生窗口只有目标模型可重放时才算边界
SessionHistory.preview  同上                 手动压缩前组装上下文（llm.ts:125）
模型中立视图（UI 等）    "latest"            所有完成压缩都算边界 → 最小可见历史
原始 transcript 工具     "local"             本地摘要算边界、原生窗口不算
                                              → 尽量还原完整对话（compactNatively
                                                 里 llm.ts:363 用 "local" 重读原始历史）
```

`entriesForRunner`（:118-135）在事务里同时取 `InstructionState.initial` + 消息，保证指令 baseline 与边界一致。

### 4.2 摘要叠加的收敛性

每次 update 压缩都重写整份摘要（非拼接），且 prompt 明令"Newer history always takes precedence … you may remove it"（:171）。legacy 检测（`summary.includes("## Additional Context")`，:282 + :182）让 #48058 之前的臃肿摘要在下一次压缩时被一次性重写到新粒度（:161-165 注释给出完整迁移说明与删除条件）。这保证了 checkpoint 尺寸 O(1)，不会随压缩次数增长。

---


---

## 5. 不变量与设计取舍

1. **压缩是控制项而非普通消息**：`/compact` 经 inbox 控制项投递（spec L23），投递即 `InboxDelivered` 消费 inbox 行但**不产生 transcript 消息**（spec L7 "consumed without becoming transcript messages"；inbox.ts:624 投影跳过）。为什么：压缩要在"安全边界 + 无人插队"下重组历史，若作为普通 prompt，后续 steers 可能落进即将被摘要掉的前缀；控制项的 delivery boundary 语义（inbox.ts:507 findIndex）保证边界干净。
2. **摘要是 durable 事件**：`Compaction.Ended` 是 durable 事件（session-event.ts:609），投影落成 completed compaction 消息行——它既是模型上下文的一部分（checkpoint 展开），又是历史加载边界（history.ts:90），又推进指令 epoch（projector.ts:733）。Delta 则是 ephemeral（:601）——流式过程不进 replay（spec L110）。
3. **transcript 无损**：压缩从不删旧消息行，只移动读取起点。revert 才删。两者互补成"可压缩、可回退"的历史。
4. **一个会话同时最多一个压缩**：pending 项去重（inbox.ts:219-220）+ 注释明示（compaction.ts:618 "A session never runs two compactions at once"）。
5. **估算可以错，拒绝兜底**：本地 token 估算只是启发式（4 字符/token，model-request.ts:54），真正的裁决者是 provider 拒绝——所以才有 SHRINK_STEPS 的"以首次被拒尺寸为基数逐级缩"（:441-448），而不是相信自己的估算重算。
6. **原生窗口不跨边界**：spec L102 "provider-native continuation state does not cross that boundary"；provenance 校验（:342-349）+ Boundary 三态过滤（history.ts:23-26）+ `due()` 的测量闸（:247-249）三处共同保证不会把 A 家的加密窗口喂给 B 家。
7. **计费诚实**：失败/中断的压缩也计费（:616-618），费用随 Ended/Failed 事件暴露给客户端。

---

## 6. 失败与恢复

- **压缩中断（aborted）**：manual 压缩被 interrupt → `onInterrupt` 刻意不发事件（compaction.ts:672-675，runner 拥有该消息的生命周期）；runner 侧捕到 `Cause.hasInterruptsOnly` → 发 `Compaction.Failed(error={type:"aborted"})`（llm.ts:150-158）。auto/overflow 中断则由 onInterrupt 直接发 `compaction.interrupted`（:673-675）。
- **崩溃恢复 `settleStaleCompactions`**（runner/llm.ts:332-360，注意：在 `session/runner/llm.ts` 而非 `llm.ts`）：进程死亡跳过了压缩的 finalizer，留下 `status='running'` 的孤儿 compaction 行。每次 drain 入口（:71）扫描它们，**从最新往最旧**（匹配事件投影顺序）逐个发 `Compaction.Failed(compaction.interrupted)`——message-updater 把 running 转 failed（:465-484）。注释（:335-336）点名"包括藏在已完成检查点后面的孤儿"。
- **失败后的会话状态**：失败的 compaction 消息 `status='failed'`，`to-llm-message.ts:308` 直接跳过（不进模型上下文），history 加载也不受影响（latestCompaction 只找 completed）——**失败压缩零副作用于后续请求**。auto 失败会让 Step 直接 `StepFailedError`（llm.ts:221）；overflow 失败同理（recoverOverflow 返回 false → 维持 overflow 终态）。
- **不可再缩**：deliver 的 "cannot be reduced further" 分支（:419-427）——保最新 exchange 与 checkpoint 的底线不可破。
- **move 的恢复路径**：源失联紧急发布 Moved（move.ts:140-147）+ drain 入口 `lifecycle.isClosed` 时关 transport 并 Reloaded（llm.ts:78-81）。

---

## 演进注记

- **旧笔记 4.3 §9 的 `compactIfNeeded` / `compactAfterOverflow` 已不存在**（全仓 grep 零命中）。现结构是单一 `compact(trigger)` 入口 + reason 判定：`compactIfNeeded` 的职责化为 `due()`（compaction.ts:242-251）+ runner 每 Step 的无差别调用（llm.ts:220，由 compact 内部自判 skipped）；`compactAfterOverflow` 化为 step.ts:158-166 的 `recoverOverflow` 回调 + llm.ts:320-323 的一次性闸门。旧笔记时代的 compaction 约 241 行，现 988 行：新增了原生 provider 窗口（compactNatively + providerContext/provenance 体系）、SHRINK_STEPS 预算循环、flattenAndDropOldest 拍平、update/legacy 双态摘要 prompt、fitOutput、估算锚点（estimatePrompt 的 measured/estimated 双轨）。
- **compaction.ts 里的 "todo"**：:74 与 :76 是两处普通代码注释（`// todo: this should become a msg reference, we shouldnt be stringifying the recent history`、`// todo: consolidate these 2`），与已死的 Todo 功能（旧笔记 4.14）**毫无关系**——只是作者承认两个技术债：① recent 历史被 stringify 成文本而非消息引用（注释自注"confuses some models"）；② providerState/providerContext 两个出口待合并。类似的还有 :480 的 `TODO: Read the previous summary from the hooked request`。
- **revert/move/transfer/diff 已拆至 [7.1_session-operations.md](7.1_session-operations.md)**（2026-10 拆分，零代码耦合实证）；4_session.md §7 的操作面总表仍覆盖全部四操作入口。
- **transfer 曾被误记为"父会话转手"**：实测是 export/import 序列化搬运（[7.1](7.1_session-operations.md) §3），parentID 仅为导入完整性校验。
- **手动压缩的事件归属曾分裂**：现规则是 manual 的 Started 由 runner 在投递时发（llm.ts:92-95），auto 的 Started 由 compaction 服务自己在 deliver 里发（compaction.ts:399-401），Failed 两边都可能发——onInterrupt 的 manual 分支不发（:672-675）是这套归属划分的关键细节。

## 相关篇

- [4_session.md](4_session.md)：§7 触发面总表（本篇 §2 的入口行号出处）、§5 执行协调（drain/边界）、§6 读取路径（projector 投影事务）
- [1_data-layer.md](1_data-layer.md)：durable/ephemeral 事件、Bus.reserveSequence、投影事务——理解 Ended 事件三重身份的基础
- 规划中 5_inbox-coordination.md：控制项投递全语义（steer/queue、delivery boundary）——本篇 §2.1/§5.1 只引用
- 未来 instructions 篇：InstructionState.advanceEpoch 与 epoch 三事件（Ended/Moved/Committed）
