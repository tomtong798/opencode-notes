# Session 聚合详解（inbox 准入 → drain 执行 → 投影读模型的事件溯源聚合根）

> 一句话：**Session 是 opencode 的编排核心——一个以 sessionID 为聚合 ID 的事件溯源聚合根，所有输入先以 inbox 项的形式 durable 准入（admit），再由进程本地的执行协调器在安全边界提升（promote）为可见消息并驱动 LLM Step，读模型完全由事件投影而来。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：`specs/v2/session.md`
> [返回 README](README.md) | 前置 [1_data-layer.md](1_data-layer.md) | 关联 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)（执行链时序）

---

## 0. 模块地图

`packages/core/src/session/`（43 文件，11,159 行）+ 门面 `packages/core/src/session.ts`（495 行）。公共形状在 `packages/schema/src/`（session.ts 66 行 / session-message.ts 322 行 / session-inbox.ts 78 行 / session-event.ts 723 行）。

```
packages/core/src/
├─ session.ts ★门面（Effect 服务，495 行）
│   ├─ create / fork / remove / list / log / diff / background / generate / command
│   └─ 其余操作委托给 Session.make() 的 ID 绑定 Handle
└─ session/（聚合内部）
    ├─ 【聚合根】
    │   ├─ session.ts ★ Session.make()/forSession() ID 绑定操作集（426 行）
    │   ├─ schema.ts   纯 re-export @opencode/schema/session（9 行）
    │   ├─ event.ts    纯 re-export session-event（2 行）
    │   └─ message.ts  纯 re-export session-message（2 行）
    ├─ 【准入】
    │   ├─ inbox.ts  ★ SessionInbox：admit/reconcile/promote/cancel/steer/queue + 投影（550 行）
    │   └─ prompt.ts SessionPrompt.prepare：hooks→附件→skill→UserPayload（231 行）
    ├─ 【执行协调】
    │   ├─ execution.ts      ★ SessionExecution：进程全局调度 + claim 生命周期（195 行）
    │   ├─ run-coordinator.ts ★ 泛型 Coordinator：wake 门铃/interrupt/awaitIdle（196 行）
    │   └─ execution/
    │       └─ restart.ts    ★ 崩溃恢复：resumeSuspendedSessions 扫描（240 行）
    ├─ 【读取路径】
    │   ├─ store.ts     ★ SessionStore：session 行 CRUD + claim + 游标分页（256 行）
    │   ├─ history.ts   ★ SessionHistory：compaction 边界裁剪加载（181 行）
    │   ├─ projector.ts ★ SessionProjector：全部 durable 事件→表投影（810 行）
    │   ├─ info.ts      行→Session.Info 解码（64 行）
    │   └─ stats.ts     跨会话用量/工具统计（436 行）
    ├─ 【会话操作】
    │   ├─ revert.ts    ★ SessionRevert stage/clear/commit（114 行）
    │   ├─ move.ts      ★ SessionMove：move 控制项准入 + 源失联直移（179 行）
    │   ├─ transfer.ts  SessionTransfer export/import（339 行）
    │   ├─ diff.ts      SessionDiff.turn：按 turn 取文件 diff（138 行）
    │   ├─ command.ts   /command 展开（35 行）
    │   └─ compaction.ts ★ SessionCompaction：auto/manual/overflow 压缩（988 行）
    ├─ 【运行器（Location 作用域）】runner/
    │   ├─ index.ts Interface + DrainResult（42 行）  ├─ llm.ts ★ drain/Step 循环/孤儿结算（408 行）
    │   ├─ step.ts 一次 Physical Attempt（321 行）    ├─ retry.ts 重试决策（115 行）
    │   ├─ model.ts 模型解析（96 行）                 ├─ max-steps.ts（16 行）
    │   ├─ to-llm-message.ts（334 行）                └─ publish-llm-event.ts（680 行）
    ├─ 【上下文组装】
    │   ├─ context.ts         SessionContext select/load（208 行）
    │   ├─ model-request.ts   请求组装（439 行）      ├─ model-transport.ts 流传输（571 行）
    │   ├─ message-updater.ts 事件→消息态机（488 行） ├─ provider-context.ts 原生检查点（84 行）
    │   └─ system-prompt.ts（28 行）
    ├─ 【指令子系统】
    │   ├─ instruction-state.ts epoch/baseline（274 行） ├─ instruction-entry.ts API 管理条目（181 行）
    │   └─ instructions.ts 源组合（125 行）
    ├─ 【生命周期与派生】
    │   ├─ shell.ts              后台 shell 启动（27 行）
    │   ├─ skill.ts              skill 激活（16 行）    ├─ environment.ts 会话环境变量（33 行）
    │   ├─ subagent-job.ts       后台子代理 Job（76 行）├─ subagent-completion.ts 完成通知（46 行）
    │   ├─ title.ts              标题生成（157 行）     ├─ generate.ts 免准入生成（73 行）
    │   └─ usage.ts              token/cost 计算（56 行）
    └─ 【杂项】
        ├─ error.ts 错误类型（90 行） ├─ to-session-error.ts（105 行）├─ affinity.ts（8 行）
        └─ sql.ts ★ 全部表定义（176 行）
```

> 防重复约定：runner/step/publish-llm-event 的流式与工具时序归 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)；system-context/instructions 代数归未来 6_context 篇；compaction 摘要算法归未来 7_compaction 篇。

## 1. 心智模型

三个概念撑起整个聚合：

1. **聚合根（Aggregate Root）**：`sessionID` 既是数据库外键中心，也是事件总线的 aggregateID。所有 durable 事件按 seq 单调编号，投影在事件事务内同步落表——**事件是真相源，表是折叠结果**（AGENTS.md Session Core 第 1 条）。
2. **inbox 准入（Admission precedes execution）**：`prompt()` 永远先发布 `session.inbox.enqueued`（投影插入一行 `session_inbox`），然后才"建议性"地 `wake()` 执行。inbox 项在投递前**不属于模型可见历史**（spec L7-9）。
3. **投影（Projection）**：`session.inbox.delivered` 的投影在同一事务里"删 inbox 行 + 插可见消息"；compaction/move 控制项则只删行不成为消息（projector.ts:618-648）。

```
用户/hook/通知
   │ prompt/synthetic/compact/move
   ▼
┌─────────────── 门面 session.ts（Effect 服务）───────────────┐
│  Session.make().forSession(id) → ID 绑定操作集               │
└──────────────┬──────────────────────────────────────────────┘
               ▼
┌────────── 准入（durable，先于执行）──────────┐
│ SessionInbox.admit                          │
│  ├ reconcile：幂等检查（inbox 表 + 已投影消息）│
│  └ publish session.inbox.enqueued            │
│      └ 投影 → INSERT session_inbox 行        │
└──────────────┬───────────────────────────────┘
               │ resume!==false 时 execution.wake(sessionID)
               ▼
┌────────── 执行协调（进程本地）───────────────┐
│ SessionRunCoordinator（门铃合并）            │
│  wake ─→ start/drain ⟲ ─→ settle            │
│  started→Execution.Started + ★claim(write-ahead)│
│  settled→Succeeded/Failed/Interrupted + release│
└──────────────┬───────────────────────────────┘
               ▼
┌────────── runner.drain（Location 作用域，止于模块边界）┐
│ nextPromotable → promote（InboxDelivered 投影出消息）  │
│ → SessionStep（llm.stream 一次 = 一个 Step）           │
│ → 事件流（Text/Tool/Reasoning/...）                   │
└──────────────┬────────────────────────────────────────┘
               ▼ 投影（bus.project，同事务）
   session / session_message 读模型 ← list/messages/context/log
```

关键分界：**准入可以独立于执行随时调用**（spec L60）；执行是"建议性"的——wake 只是按门铃，真正决定是否推进的是 drain 入口的 `nextPromotable` 检查（llm.ts:64-69（nextPromotable 在 65 行））。

## 2. 数据模型

### 2.1 session_v2 表（sql.ts:22-77）

| 列 | 类型 | 说明 |
|---|---|---|
| id | text PK | `SessionSchema.ID` |
| project_id | text → project.id, cascade | 归属 Project（创建时 resolve 一次，Session 不重复写） |
| workspace_id | text? | 显式 workspace（空 = 隐式本地放置） |
| parent_id / fork_session_id / fork_boundary | text? / json | 子会话父链 / fork 来源与边界 |
| slug / directory / path | text | 目录定位 + 项目内子路径 |
| title / version / share_url | text? | |
| summary_* / cost / tokens_* | int/real | 用量聚合（投影累加，projector.ts:58-73） |
| metadata / permission / agent / model | json | 会话级配置 |
| time_created/updated（Timestamps）| int | time_updated 是 list 排序键 |
| time_idle / time_viewed / idle_outcome | int?/text | 空闲边界 + 未读水位（单调 max 更新） |
| time_compacting / time_archived | int? | |
| time_suspended | int? | ★执行 claim 时间戳（历史列名，见 store.claim） |
| resume_attempts | int | 崩溃恢复计数（仅 terminal 清零） |

索引：project/workspace/parent + `time_suspended is not null` 部分索引（恢复扫描用，sql.ts:73-75）。

### 2.2 session_message 表（sql.ts:79-98）

| 列 | 类型 | 说明 |
|---|---|---|
| id | text PK | `msg_` 前缀；用户消息 ID = 原 inbox ID |
| session_id | text → session_v2, cascade | |
| type | text | tagged union 判别子 |
| seq | int | ★聚合内单调序，唯一索引 (session_id, seq) |
| time_created/updated | int | |
| data | json | 编码后的判别联合剩余字段（去掉 id/type） |

消息写入只有两条路：inbox 投影 `insertMessage`（projector.ts:395-410，seq = 事件 seq）与 runner 事件经 `SessionMessageUpdater` 的 upsert。

### 2.3 session_inbox 表（sql.ts:125-145（表名见 126 行））——只存未消费工作

| 列 | 类型 | 说明 |
|---|---|---|
| id | text PK | = 将来消息的 ID |
| session_id | text, cascade | |
| type | text | user / synthetic / compaction / move |
| payload | json | UserPayload/SyntheticPayload/CompactionPayload/MovePayload |
| delivery | text | "steer" \| "queue" |
| enqueued_seq | int | ★准入序 = enqueued 事件的聚合 seq（唯一索引） |
| time_created | int | |

> 旧表 `session_pending`（sql.ts:100-123，带 `session_pending_session_compaction_idx` 每会话至多一个 compaction 的部分唯一索引）仍在定义中，现码活跃路径全部走 `session_inbox`（演进注记 §11.2）。

### 2.4 SessionMessage tagged union（schema/session-message.ts:297-311）

| 分支 | 行 | 要点 |
|---|---|---|
| AgentSelected / ModelSelected | 44/52 | 切换事件落成消息 |
| LocationSwitched | 60 | move 后 |
| **User** | 75 | text/files/agents/skills（来自 inbox UserPayload 还原） |
| **Synthetic** | 85 | text + description（shell/subagent 通知、续写指令） |
| **System** | 93 | ★指令更新冻结文本（frozen text）+ 摘要 description |
| Skill | 103 | 激活的 skill 内容 |
| Shell | 111 | shellID/status/exit/output，started+ended 合一 entry |
| **Assistant** | 213 | content=[text/reasoning/tool]，finish，retry，time.streamed/completed |
| Compaction | 246-289 | status: running/completed/failed 子联合 |
| Idle | 291 | ★turn 边界标记：outcome + 可选 error |

### 2.5 durable 事件清单（@opencode/schema/session-event.ts，723 行）

Session 聚合的真相源。按功能分组（durable = 进 `log` 回放；其余为 live-only）：

| 分组 | 事件 | 投影落点（projector.ts） |
|---|---|---|
| 生命周期 | Created / Forked / Deleted / Moved | INSERT session 行 / fork 复制 / DELETE / UPDATE location |
| 准入 | InboxEnqueued / InboxDelivered / InboxCancelled / InboxDeliveryChanged | inbox 行四态（§4.2） |
| 执行 | Execution.Started / Succeeded / Failed / Interrupted | claim 写入 / projectIdle + claim 释放 |
| Step | Step.Started / Streamed / Ended / Failed | SessionMessageUpdater upsert + usage 累加 |
| 内容流 | Text/Reasoning.{Started,Ended}、Tool.Input.{Started,Ended} | assistant 消息内容增量 |
| 工具 | Tool.Called / Success / Failed | 工具终态 |
| 重试 | RetryScheduled | 投影重试态（后续活动清除） |
| 压缩 | Compaction.Started / Ended / Failed | compaction 消息 + epoch 前移 |
| 指令 | InstructionsUpdated | System 冻结文本 + InstructionState.apply |
| 杂项 | Synthetic / Skill.Activated / Shell.{Started,Ended} / Renamed / MetadataUpdated / Permissions / Viewed / AgentSelected / ModelSelected / MessageContentUpdated / UsageRecorded | 各自 upsert |

AGENTS.md 的"事件最小化"原则在此体现：usage 只记增量（UsageRecorded），累计量由投影折叠；Idle 的 turn 语义由 Execution terminal 派生。

## 3. 聚合根接口走查

两层结构（spec L44-58）：

- **门面** `session.ts`（Effect 服务 `@opencode/Session`，Interface 见 session.ts:118-237）：持有集合级操作（create/fork/list/remove/log/diff/generate/command/background）+ 全局 deps；其余逐 ID 委托。
- **ID 绑定值** `session/session.ts`：`Session.make()` 在宿主 Scope 构建一次（session/session.ts:42），`forSession(id)` 只是 `bind` 出共享实现的操作集（370-420）。**Handle 不缓存投影、不持有 runner**；获取/丢弃 Handle 不启动也不中断执行。

逐方法（session/session.ts 行号）：

| 方法 | 行 | 语义要点 |
|---|---|---|
| prompt | 145-178（uninterruptibleMask 起于 146 行）| `uninterruptibleMask`：先 `admission.reconcile`（幂等），未命中才 `SessionPrompt.prepare`（restore 可中断段）→ 有 staged revert 先 commit → `admission.admit` → `resume!==false` 才 `execution.wake`。LifecycleConflict → `PromptConflictError` |
| synthetic | 273-314 | 直接构造 SyntheticPayload admit；有 staged revert 时**不 wake**（310） |
| shell | 179-222 | 立即后台启动（`forkIn(scope, startImmediately)` + join）；启动失败/完成都以 `synthetic(resume:false)` 通知，不打断 prompt 唤醒 |
| skill | 223-245 | 发布 Skill.Activated 后 fork `execution.resume` |
| compact | 246-264 | 先 commit staged revert，`admitCompaction`，必然 wake |
| interrupt | 315-318 | 包 `execution.interrupt`，uninterruptible |
| wait / resume | 265-272 | awaitIdle / coordinator.run |
| cancelInbox/steerInbox/queueInbox | 130-144 | 包 `mutatePending`；**只有 steer 后 wake**（139），queue/cancel 不调度（spec L76） |
| revert.stage/clear/commit | 319-345 | 活跃时 BusyError；clear 后 wake |
| view | 61-70 | 单调 viewed 水位 |

门面特有（session.ts 行号）：`create` 262-318（复用 ID = 采纳既有会话；并发投影竞态由 `SessionAlreadyProjected` defect 捕获回读，305-313）；`fork` 319-359（边界消息定位 + 事务读 instruction 状态，fork **采纳父会话最新指令值**而非边界时值，349-351）；`remove` 367-377（interrupt→awaitIdle→transport.close→递归删子→Deleted 事件）；`log` 405-415（`bus.log` 独占 seq 游标 + Synced 水位 + follow）；`background` 442-459（jobs.backgroundAll + synthetic 说明）。

## 4. 准入子系统（prompt.ts + inbox.ts）

### 4.1 SessionPrompt.prepare（prompt.ts:29-91）

把提交输入变成 user inbox item，**不 admit、不 commit revert、不 wake**（spec L66）。流程：`Plugin.awaitActivation` → `session.prompt` hook（structuredClone 隔离）→ 附件物化（data URL / file:// / 目录列表，上限 20MB，prompt.ts:203）→ skill 解析（SkillNotFoundError）→ `Prompt.make` + metadata → `instances.provide(session)` 在会话 Location 实例内执行。**重试 reconcile 发生在 prepare 之前**（session/session.ts:151-157），已准入输入跳过 hooks 与附件解析（spec L64）。

### 4.2 状态机

```
                admit / admitCompaction
   (无) ───────────────────────────────────▶ session_inbox 行 (pending)
                                                 │ delivery = steer|queue
              steer/queue（mutatePending）        │
   pending ──────────────────────────────────────┤ delivery 互换（projectDeliveryChanged
                                                 │  ：CAS on 旧值，inbox.ts:357-384）
   pending ──── cancel ─────────────────────────▶ (行删除，projectCancelled 只允许
                                                 │  steer/queue 状态，inbox.ts:338-355)
   pending ──── promote（runner 在边界调用）────▶ (行删除 + 可见消息插入，同事务)
              │                                   projectDelivered inbox.ts:324-336
              └─ compaction/move：行删除但不成为消息（projector.ts:624）
```

- **reconcile**（inbox.ts:155-167）：`find`（inbox 表）?? `promotedFromMessage`（已投递则从 session_message 反解，126-143 要求 sessionID 与 user/synthetic 类型匹配）。命中即"第一次准入获胜"，重试的 payload/metadata/delivery 全部忽略（spec L11）；跨会话或跨类型 → `LifecycleConflict`。compaction 不参与 reconcile 幂等（164）。
- **admit**（169-202）：先 reconcile，未命中发布 `InboxEnqueued`；投影竞态 defect 也走 find 回读（191-197）。
- **admitCompaction**（204-228）：`serialized` 锁内——精确命中同会话 compaction 返回；同 ID 已成其他消息/promoted → 冲突；已有 pending compaction → 返回那个（一会话一个待处理 compaction）；否则 admit。
- **promote**（496-531，核心投递规则）：
  - 有 steer 或 scope==="steer"：取 `pendingSteers`（533-550，**排序修正**：steer 队列里若有 compaction 控制项且不在队首，把它提到队首——"compact before promoting even earlier steers so their text stays verbatim after the checkpoint"；move 不提，因为移动改变 Location）。`control===0`（队首就是控制项）返回 `undefined` 交还 runner 处理；否则发布队首到控制项之前的所有 steer。
  - 无 steer 且 scope==="input"（空闲边界）：取**一个** queued 项；若是控制项返回 undefined；否则投递它，再补投期间新到的 steers（521-528）。
- **steer/queue 边界规则**（对照 spec "Delivery is explicit"，L15-19）：steer 按准入序在下一个安全步边界投递，遇 compaction/move 控制项停止；queue 在会话可继续时保持 pending，空闲边界 steers 优先，否则恰好一个 queued 项投递后 runner 重新评估是否续跑。

取消/变更的并发安全：所有变更经 `KeyedMutex` 的 `serialized(sessionID)` 串行（inbox.ts:59-63）；promote/nextPromotable 也在锁内或由 drain 的 `SessionInbox.serialized` 包裹（llm.ts:83-108）。

### 4.3 投递顺序示例（pendingSteers 的 compaction 提队）

设 inbox 按准入序为 `[U1(steer), U2(steer), C(compaction,steer), U3(steer), Q1(queue)]`：

- `pendingSteers` 排序修正后：`[C, U1, U2, U3]`——C 被提到队首（inbox.ts:546-548）。理由（543-545 注释）：move 改变 Location，compaction 绝不跨 move；但在 move 边界内，先 compact 再提升更早的 steer，能让这些 steer 的原文在检查点之后逐字保留，而不是被卷进摘要。
- promote("steer") → 控制项在队首（control===0）→ 返回 undefined，由 runner 在 `advanceToStep` 锁内单独处理：原子发布 `InboxDelivered + Compaction.Started`（llm.ts:91-95）。
- compaction 完成后 runner `continue` 重新进边界，此时提升 `[U1, U2, U3]`（一批，step 只重置一次），Q1 仍留队。

若顺序改为 `[U1, M(move,steer), U2]`：promote 只投递 `[U1]`，止于 M；M 由 runner 处理时关闭 transport 并发布 `InboxDelivered + Moved`（llm.ts:96-105），drain 返回 `Moved`，execution 层换 Location 后带着 continuation 重入（execution.ts:107）。

### 4.4 与 spec 投递词汇的逐条对照

| spec 表述（行） | 代码 |
|---|---|
| "steer is the default"（L19） | `input.delivery ?? "steer"`（session/session.ts:155、prompt 事件默认 prompt.ts:50） |
| "Steers deliver in enqueue order at the next Safe Step Boundary"（L19-20） | enqueued_seq 排序（inbox.ts:391/419/516/538）+ 边界点 = advanceToStep 循环头（llm.ts:83-108） |
| "Delivery stops before a compaction or move control item"（L20） | promote 的 `findIndex(control)` + slice（inbox.ts:507-509） |
| "queue remains pending while the Session can continue"（L21） | llm.ts:68：steer-scope drain 遇 queued 项直接 Complete |
| "at an idle boundary, steers still take priority; otherwise one queued item delivers, followed by any steers that arrived during delivery"（L21-22） | inbox.ts:512-528 |
| "The runner then reevaluates continuation before another queued item"（L22-23） | 第二个 queued 项永远要等下一轮 drain（promote 每次 input-scope 至多取一个 queued） |
| "Inbox items may be cancelled or changed between queue and steer before delivery"（AGENTS.md） | cancel 的 WHERE 限定 delivery in (steer,queue)（inbox.ts:348） |

## 5. 执行协调（execution.ts + run-coordinator.ts + execution/restart.ts）

### 5.1 SessionRunCoordinator（泛型，run-coordinator.ts）


进程内 `Map<Key, Execution>`，每 key 一个 fiber 的 busy period。核心是**门铃（doorbell）合并**：

```
            wake | run                 （run-coordinator.ts:50-58 注释图）
    idle ──────────────▶ execution (one fiber)
                           drain ⟲ doorbell rung mid-drain
                           │ exit (settled hook runs)
    doorbell quiet ◀───────┴───────▶ doorbell rung
    idle, waiters get exit          successor execution
```

- `loop`（74-85）：drain 结束后若 `pendingWake` 存在，吸收为新的 scope 并蹦床续跑（`Effect.yieldNow` 防栈增长）——**重复 wake 合并为一次后续 drain**。
- `wake`（136-145）：活跃时只置 `pendingWake`（**合并取最宽 scope**：input ⊃ steer，141）；空闲才 start。
- `run`（124-134）：显式 resume 加入当前执行并等其 exit；stopping 的执行拒绝 joiner，等清理完重跑。
- `interrupt`（147-167）：置 stopping、**没收已记录的 wake**（属于被打断的意图，161）、fire-and-forget 中断 fiber、立即返回 true；结算窗口（owner 已退出、settled 未完）返回 false 但同样清 wake。
- `settle`（116-120）：exit 时若门铃又被按 → 开后继执行，否则删 key；`done` 通知所有 joiner。

### 5.2 SessionExecution（execution.ts）

进程全局、仅按 sessionID 键控。`drain`（87-110）在每次排水开始时 `store.get` 重读会话、`instances.provide(session)` 进入其 Location 取 runner——**放置发现推迟到 drain 开始**（AGENTS.md）。DrainResult 三态：`Complete` / `Moved`（换 Location 续跑）/ `Reloaded`（Location 关闭重入）。

**write-ahead claim**（spec L86-92 的代码落点）：

- `started` 钩子发布 `Execution.Started`，`claimOnCommit`（81-83）使 claim 写入与事件**同事务**：`store.claim` 置 `time_suspended`（store.ts:205-216，NULL 守卫使重复 claim 为零行 no-op；`time_updated` 自赋值，claim 不算用户活动）。
- `settled` 钩子（119-147）按 terminal 分类：Succeeded / Failed / 非 shutdown 的 Interrupted 都 `releaseOnCommit`（清 claim + 重置 resume_attempts）；**shutdown 中断保留 claim**（134），供下次启动续跑。中断还会 `jobs.cancel`（130）。
- 注释明言（78-80）：claim 是恢复标记而非所有权/围栏——"recovery is a property of the database, never of a shutdown hook"。

`interrupt` 的 resume 选项（153-170）：中断后若 `nextPromotable("input")` 命中 steer 或控制项，则以 "steer" scope 再按门铃——queued 的下一 turn prompt 留在原地。

### 5.2.1 runner.drain 主循环细读（llm.ts:57-199，结构层面）

```
drain(input: {sessionID, force, continuation?, promotable?})
│
├─ 入口资格检查（64-69）：!force 且非续跑时 nextPromotable("input")
│    无 pending → Complete；steer-scope 遇 queued 普通 prompt → Complete
├─ plugins.awaitActivation → settleStaleCompactions → settleStaleToolCalls（71-72）
│    ★孤儿结算先于一切模型工作（spec L133-135）
├─ advanceToStep（74-189）：uninterruptibleMask 内的边界循环
│    ├─ lifecycle.isClosed → 关 transport → Reloaded（换 Location 后重入）
│    ├─ SessionInbox.serialized 锁内 nextPromotable（83-108）：
│    │    ├─ compaction → 原子发布 InboxDelivered + Compaction.Started
│    │    └─ move      → 关 transport + 原子发布 InboxDelivered + Moved
│    ├─ 非 steer 输入重置 step=1 / entering（109-112）
│    ├─ move → 返回 Moved（113-114）；compaction → 执行压缩后 continue（115-163）
│    ├─ 资格再检查（164-165）：!force && !continuing && 无 pending → Complete
│    └─ ready：prepareContext（指令 baseline）+ promote + 标题生成 + context.load（166-184）
│         promote 返回 undefined = 边界被新到的控制项占领 → 重查（175-176）
└─ while(true)：runStep(context, step) → step++ / force=false / entering=false（191-198）
```

runStep（209-330）是"一个逻辑 Step、多个 Physical Attempt"的落点：`while(true)` 内按 Outcome 分派——Completed 返回 needsContinuation；Retry 退避等待（保留 assistantMessageID）；Continue（不完整流）发布 Synthetic 续写指令并**换新 assistantMessageID**（287-289）；OutputLimit 连续 3 次 `finish=length` 才终态（306-315）；Compacted 消耗一次性 overflow 恢复（320-323）；RecoverFull 消耗一次性全量重建（324-326）。流式/工具时序细节见 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)。

### 5.3 崩溃恢复（execution/restart.ts）

`resumeSuspendedSessions`（193-231）托管服务器启动时调用：

1. 收集 suspended 集合 = `store.listSuspended()`（顶层 + time_suspended 非空，store.ts:194-204）∪ 运行中的 subagent 后台 job 子会话，排除本进程活跃的。
2. `releaseChildClaims(children)`（store.ts:225-238，可恢复 subagent 除外）。
3. **先投递 shell/subagent 通知再恢复子会话**（208 排序：shell 先），恢复通知以 `synthetic(resume:false)` 准入。
4. 对每个仍 suspended 的会话：`prepareResume`（77-99）先 `countResume`（**durable 先行**，崩溃在恢复路径中也会被下一次 sweep 计数）→ 超 `DEFAULT_MAX_ATTEMPTS=10`（33）则发布 `Execution.Failed`（RESUME_EXHAUSTED）并 release 终结；否则发布 restart 续写 Synthetic 指令再 `execution.resume`。

at-least-once 语义在头注释（52-53）明确："local coordination prevents concurrent drains, not repeated external side effects"。

## 6. 读取路径（store.ts / history.ts / projector.ts）

### 6.1 SessionStore（store.ts）

- `list`（99-139）：`time_updated+id` 键值游标（anchor direction=previous 时翻转排序再 reverse 回来）；支持 directory/project+subpath/workspaceID/search/parentID 过滤。
- `messages`（140-178）：游标分页——cursor.id 先解析为 seq，再 `seq > / < anchor.seq` 严格排他边界，order 与 direction 组合决定翻转；type 可过滤。**语义：独占游标，cursor 指向的消息本身不返回**。
- `context`（179）：`SessionHistory.load(db, id, "latest")`。
- claim/release/countResume：见 §5.2。

### 6.2 SessionHistory（history.ts）

`Boundary`（21）：`"latest"`（模型中立读者，越过一切原生检查点）/ `"local"`（原始转录，不重放原生窗口）/ `Provenance`（具体模型的兼容性判定）。`messageEntries`（78-108）：定位最近 completed compaction（`latestCompaction` 28-60，对原生窗口附加 provenance JSON 匹配）→ 只载 `seq >= compaction.seq` → 过滤：原生检查点之前的 system 冻结文本被 baseline 取代（104-107 注释）+ 不兼容检查点剔除。`entriesForRunner`（118-135）在事务里合并 initial 指令基线；`preview`（137-162）截到 settled 前缀（未完成 assistant 之前）。

### 6.3 SessionProjector（projector.ts）

一个 `Layer.effectDiscard` 注册约 40 个 `bus.project`。关键落点：

| 事件 | 行 | 投影动作 |
|---|---|---|
| Created | 447 | INSERT session 行（冲突 → `SessionAlreadyProjected` defect） |
| InboxEnqueued | 650 | `projectAdmitted`（inbox.ts:277-317，消息已存在/插入冲突 → die）+ touch time_updated |
| InboxDelivered | 618 | `projectDelivered` 删行 → user/synthetic 才 `insertMessage`（seq=事件 seq） |
| InboxCancelled / DeliveryChanged | 667/673 | 删行（仅 steer/queue 态）/ CAS 更新 delivery |
| Execution.{Succeeded,Failed,Interrupted} | 680-682 | `projectIdle`（412-441）：time_idle 用 `max(time, coalesce(old+1))` **强单调**（同毫秒 terminal 也要推进未读水位）+ idle_outcome |
| UsageRecorded | 616 | SQL 自增累加 cost/tokens |
| Forked | 617→115 | 批量复制 settled 消息（500/批，运行中的 assistant/shell/compaction 不复制，203-205）+ `Bus.reserveSequence` 保持 seq 连续 |
| RevertEvent.Committed | 759 | ★删边界及以上消息 + **删 enqueued_seq ≥ 边界的 inbox 行**（777-786）+ `InstructionState.reset` |
| Compaction.Ended | 730 | run + `InstructionState.advanceEpoch`（epoch 前移到事件 seq） |
| Step.Ended/Failed、Usage | 796-806 | 订阅（非投影）→ 回读行汇总发布 `UsageUpdated` |

`log`（门面 session.ts:405-415）：durable 事件回放 + `EventLog.Synced` 水位标记 + follow 订阅；live-only 增量（text/reasoning/tool-input 流 delta）**有意不回放**（spec L102）。

### 6.4 游标分页语义示例

`messages({cursor: {id: m5, direction: "next"}, order: "asc", limit: 3})`，会话消息 seq 为 m1..m9：

1. 解析 cursor：m5 → seq 5（store.ts:144-154）；解析失败返回空（154）。
2. 边界：order=asc + next → `seq > 5`（157）。
3. 返回 `[m6, m7, m8]`。翻页把 cursor 换成 m8（返回列表末元素）继续。

`direction: "previous"` 时：内部 order 翻转为 desc、取 `seq < 5` 的 `[m4, m3, m2]`，再 `toReversed()` 还原为时间序 `[m2, m3, m4]`（171-177）——**返回值永远保持用户请求的方向，翻转只发生在 SQL 层**。`list` 的 anchor 语义相同（99-139），游标键是 `(time_updated, id)` 复合比较，避免同毫秒排序不稳定。

另一个容易忽略的点：`log` 的 Synced 标记 seq **可能大于**最后一条发出事件的 seq，因为同一聚合的 durable 总线里还有非 Session 事件共享序列空间（门面注释 session.ts:163-168）。

## 7. 会话操作

### 7.1 revert（revert.ts + session/session.ts:319-345）

- `stage`（21-57）：活跃时 BusyError；捕获（或复用已有）快照 → `plan`（84-114）扫边界之后 assistant 消息的 `snapshot.files` 得到文件→快照映射 → **立即物理还原文件** → 发布 Staged（投影写 session.revert）。
- `clear`（59-74）：按 staged 记录把文件还原回原始快照，发布 Cleared。
- `commit`（76-82）：仅发布 Committed；重活全在投影（§6.3）：截断消息、清 inbox 残留、重置指令状态。**新 prompt 准入会自动 commit staged revert**（session/session.ts:164-165）。

### 7.2 move（move.ts）

`move`（119-158）：`resolveDestination`（74-102，~ 展开/校验目录/resolve project/**探测目标 Location 可用性**）→ `sourceUnavailable`（104-116，源目录失联探测）→ `serialized` 锁内二选一：源失联且未活跃 → 直接取消既有 move 项 + 发布 `Moved`（紧急路径，140-147）；否则以**新 inbox ID** admit move 控制项（148-154）→ wake。move 控制项的投递由 runner 在边界处理：关闭 transport → InboxDelivered + Moved 原子发布（llm.ts:96-105）→ drain 返回 `Moved`，execution 层换 Location 续跑（execution.ts:107）。

### 7.3 transfer（transfer.ts）

`export`（57-63）：info + settled 消息（可选 sanitize）。`import`（64-）：sessionID 冲突 → `ImportConflictError`；复用 `SessionEvent.Created` + 自定义 `commit` 钩子在事件事务内插消息、`reserveSequence`、回填用量（108-121）——**导入也走事件通道**，投影无特例。

### 7.4 compaction 触发面（算法归 7_compaction 篇）

三个入口汇到 `SessionCompaction.compact(trigger)`（compaction.ts:205-231）：

| 触发 | 位置 | 说明 |
|---|---|---|
| manual | inbox 控制项，llm.ts:91-159 | 投递 compaction 项 + Compaction.Started 原子发布；失败发 Compaction.Failed |
| auto | 每 Step 前，llm.ts:220 | `due()`（232-243）估算超 ceiling 才动 |
| overflow | provider 拒绝后，llm.ts:265-271 | `recoverOverflow` 一次性（第二次 overflow 终态，spec L122） |

## 8. 生命周期与派生（shell/command/subagent 派生面）

- **shell**：用户 shell 立即后台执行（不打断模型执行、不被 caller 取消影响记录，spec L60-63 的落点在 session/session.ts:185-221 的 `forkIn(hostScope)`）；Started/Ended 事件合成一条 shell entry；完成/启动失败以 `synthetic(resume:false)` 准入。崩溃后由 restart.ts:101-136 `recoverShell` 补通知。
- **command**（command.ts:11）：`/command` 展开为 prompt 参数后走 `session.prompt`，可选自定义 delivery。
- **subagent**（派生会话）：subagent 工具以 `parentID: context.sessionID` `Session.create` 子会话（tool/plugin/subagent.ts:189）并 `execution.resume` 驱动；前台等待结果，background=true 时经 `SubagentJob.background` 注册 Job 并立即返回。完成通知统一 `SubagentCompletion.deliver`（subagent-completion.ts:22-46）：向父会话 `synthetic` 一条 `<subagent sessionID=... state=...>` 消息。崩溃恢复走 restart.ts:138-190：子会话有自己的 claim，独立 `prepareResume`，父会话是否 resume 由 suspended 集合决定（154）。
- **affinity.ts**（8 行）：按 id/parentID/fork 判会话家族亲和。

### 8.1 派生关系图

```
                    ┌────────────── 顶层会话（parent_id = NULL）──────────────┐
                    │  prompt → inbox → drain → Step（LLM 调用）              │
                    └──────┬──────────────────────────────────┬───────────────┘
                           │ Session.create({parentID})       │ shell（不派生会话）
                           ▼                                  ▼
                 ┌─ 子会话（subagent 工具）─┐          ┌─ 后台 shell Job ─┐
                 │  独立聚合、独立 claim     │          │ 启动：Session.shell │
                 │  前台：execution.resume  │          │  fork 到宿主 Scope │
                 │        + 等待最后 assistant│         │ 完成：synthetic     │
                 │  后台：SubagentJob.back- │          │  (resume:false)    │
                 │        ground + Job 表    │          └───────────────────┘
                 └────────┬─────────────────┘
                          │ SubagentCompletion.deliver
                          ▼
                 父会话 synthetic <subagent sessionID state=...>
                 （前台 resume 未指定，默认可唤醒父会话继续）
```

fork 与子会话的区别：fork 复制消息到新聚合（fork_session_id 记录来源，parent_id 为空）；子会话共享 parent 链但消息独立。两者都继承 metadata/permissions（create 时 parent 回填，session.ts:285-288 注释"family uniformly"）。

### 8.2 各派生面的崩溃恢复矩阵

| 派生物 | 崩溃时状态 | 恢复路径 |
|---|---|---|
| 顶层执行 | claim 保留 | restart.ts §5.3 扫描 + 续写指令 |
| 前台 subagent | 父 claim + 子 claim 都可能悬挂 | 子会话独立走 prepareResume；父恢复后从历史续跑 |
| 后台 subagent | Job.recovery{kind:"subagent"} | restart.ts:138-190 重建 Job 并 resume 子会话 |
| 后台 shell | Job.recovery{kind:"shell"} | restart.ts:101-136 按 Job 终态发通知（running → cancelled） |
| staged revert | session.revert 持久 | 无自动动作；下次 prompt/clear/commit 处置 |

## 9. 不变量与设计取舍（对照 specs/v2/session.md）

| spec 条款（行） | 代码落点 |
|---|---|
| "Prompt admission precedes execution"（L7-9） | session/session.ts:145-178 顺序：admit 完成 → 才 wake；projector.ts:650/618 双投影 |
| "inbox 项在投递前不可见"（L8） | session_inbox 表独立于 session_message；history.ts 读取只碰后者 |
| "delivery 原子消费 inbox 行并插入消息"（L9） | projector.ts:617-648 同一事件事务：projectDelivered 删行 + insertMessage |
| "复用 Session ID 采纳既有会话"（L11） | session.ts:263-265 `store.get` 命中即返回 |
| "user/synthetic 项 ID 幂等，首准入获胜；投递后按投影消息 reconcile"（L11-13） | inbox.ts:155-167 `reconcile` + `promotedFromMessage`（113-143） |
| "resume 控调度不控持久"（L15-17） | session/session.ts:174 唯一的 wake 条件；admit 无条件 |
| "steer 默认；安全边界按序；止于控制项"（L19-20） | inbox.ts:496-531 promote + pendingSteers 533-550 |
| "空闲边界 steer 优先，否则一个 queue 项后重评估"（L21-22） | inbox.ts:512-528；llm.ts:164-165 `!force && !continuing && !pending → Complete` |
| "提升新输入重置 step 配额；一批 steer 只重置一次"（L24-25） | llm.ts:181 `if (promoted > 0) step = 1`（promote 返回计数，批内只调一次） |
| "手动 compaction 与 move 走同一 inbox，控制项形成投递边界"（L27-29） | compact 走 admitCompaction；move.ts:148-154；promote 的 control 检查 |
| "Session 操作拥有准入策略；门面供惰性 Location 服务"（L44-58） | session/session.ts 无 Database/LocationServiceMap 依赖（deps 仅 Bus/Store/Instance/Execution/Inbox/FSUtil/Scope） |
| "shell 立即启动、fork 到宿主 Scope、通知 resume:false"（L60-63） | session/session.ts:185-221；restart.ts:129 |
| "reconcile 先于 prepare，跳过 hooks/附件"（L64-66） | session/session.ts:151-157 |
| "SessionExecution 进程全局、按 ID 键控、drain 时才发现放置"（L70-72） | execution.ts:87-110 + SessionRunCoordinator |
| "显式 resume 加入活跃执行；wake 合并；跨会话并发；中断不删 pending"（L74-79） | run-coordinator.ts:124-145 / 147-167（没收 wake 但不碰 inbox） |
| "interrupt：未知会话 NotFound；空闲/非本地持有 no-op"（L81-83） | 验证在 server 层 `get`；本地无执行返回 false（run-coordinator.ts:150） |
| "busy period 提交 write-ahead claim；shutdown/崩溃保留"（L86-92） | execution.ts:81-86/111-147；store.ts:205-224 |
| "恢复有界、at-least-once"（L92-93） | restart.ts:33/77-99/52-53 |
| "durable 事件会话作用域；log 独占 seq 游标 + Synced"（L96-101） | session.ts:405-415；AGENTS.md "minimal durable events" → 投影累加派生量（projector applyUsage） |
| "恢复边界显式：wake ≠ crash recovery"（L107-111） | claim 扫描独立于 wake 路径；settleStale* 先于模型工作 |

AGENTS.md "Session Core" 补充的三条硬规则：事件最小化（不重复可折叠状态）、一步一次显式 `llm.stream`（禁止内存工具循环，llm.ts:191-198 的 while + 每轮 `prepareContext` 重载即落点）、事件回放所有权与执行所有权分离。

**取舍点评**：(1) inbox 行删除 + 消息插入的原子性依赖事件事务而非两阶段——简单，但要求投影 defect 即回滚（inbox.ts:454-456 注释）。(2) claim 用 `time_suspended` 旧列名复用而非新表——少一次迁移，代价是语义错位需注释弥合（sql.ts:66）。(3) 门铃合并把"检查-空闲"竞态关进进程内 Map，分布式放置留给未来 clustering（spec L113-115）。

## 10. 失败与恢复

1. **崩溃重启 claim 恢复**：见 §5.3。要点：claim 是唯一恢复依据（进程死亡/SIGKILL/优雅 shutdown 留下同样签名）；恢复预算 per-turn（terminal 清零，store.ts:220）；预算 durable 先行计数（restart.ts:79-81）。
2. **settleStaleCompactions**（llm.ts:332-360）：drain 入口扫 `status='running'` 的 compaction 消息，逐个发布 `Compaction.Failed(compaction.interrupted)`——**从新到旧**结算以匹配投影顺序。
3. **settleStaleToolCalls**（llm.ts:362-385）：扫投影历史里仍 streaming/running 的工具调用，发布 `Tool.Failed(aborted)`，保留原 assistant 归属（`assistantMessageID`），subagent 工具附 child sessionID；**不重放副作用**（spec L133-135 "never directly replays ambiguous side effects"）。二者都在任何模型工作之前执行（llm.ts:71-72）。
4. **inbox 残留语义**：inbox 行是纯 pending 状态，崩溃后天然保留（wake 只是建议）——重启后若无人 wake，行留待用户下次操作；恢复路径的 synthetic 通知会触发 wake。**revert commit 是唯一批量清 inbox 的非投递路径**（enqueued_seq ≥ 边界全删，projector.ts:777-786）。
5. **投递/准入竞态**：admit 与 promote/cancel 共享 `KeyedMutex` 串行；admit 的投影竞态用 `onConflictDoNothing` + 回读兜底（inbox.ts:191-197、312-316）。
6. **move 源失联**：紧急直移路径跳过投递边界（move.ts:134-147），因为源 Location 的 runner 已无法在边界处理控制项。

## 演进注记（对照旧笔记 4.3，2026-08-03 基线）

### 11.1 漂移总览
1. **session_input → session_inbox**：旧笔记的 `session/input.ts`（288 行 SessionInput）已拆为 `inbox.ts`（550 行）；表从 `session_input` 演进为 `session_pending`（sql.ts:100 遗留定义 + 20260603/20260604 迁移）再到现行 `session_inbox`（`admitted_seq` 更名 `enqueued_seq`，payload 四类型显式化）。旧笔记的 admit/promote 状态机骨架仍在，但 steer/queue 投递词汇、控制项（compaction/move）入 inbox、pendingSteers 的 compaction 提队规则都是新增。
2. **context-epoch.ts 已删除**：旧笔记的 Context Epoch（174 行独立文件）被 `instruction-state.ts`（274 行）+ `instruction_state`/`instruction_blob`/`instruction_entry` 三表取代；epoch 推进改由 `Compaction.Ended` 投影驱动（projector.ts:733），revert commit 重置状态（793）。
3. **事件重命名**：`PromptAdmitted` → `session.inbox.enqueued`；`SessionV2` 命名消失，门面重构为 `Session.make()/forSession()` 双层（旧为单层 SessionV2 + 静态函数）。
4. **执行协调**：旧笔记的 `execution/local.ts` 不复存在；现行 `execution.ts` 是进程全局服务，`run-coordinator.ts` 的门铃/settlement 语义（pendingWake 没收、successor execution）为全新设计。write-ahead claim（time_suspended 复用）与 `execution/restart.ts` 恢复扫也是旧基线没有的。
5. **消息模型**：新增 `Idle`（turn 边界标记）、`System`（指令冻结文本）、`LocationSwitched`；Compaction 拆出 providerContext（原生窗口检查点）。旧笔记 719 行中的 "System Context 代数" 章节整体迁往指令子系统。
6. **projector 从 458 行涨到 810 行**：吸收了 usage 累加、fork 批复制、worktree 归属迁移（projector.ts:495-551）、revert 截断等原分散逻辑。
7. **runner 命名与形态**：旧笔记的 `runner.run` 现为 `runner.drain`（返回 DrainResult 三态而非布尔）；`execution/local.ts` 的"定位 runner"职责上移到 execution.ts 的 `instances.provide(session)`。compaction 从 241 行膨胀到 988 行（原生 provider 窗口、overflow 恢复、SHRINK_STEPS 预算）。
8. **读取路径增强**：旧 store.ts 仅 63 行（get/list）；现行 256 行，新增 claim/release/countResume/listSuspended 恢复面与游标分页。history.ts 从 101 行到 181 行（Boundary 三态 + 原生检查点过滤）。

## 相关篇

- [1_data-layer.md](1_data-layer.md)：事件表/Bus/seq 分配与投影事务机制
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)：一条 prompt 的端到端时序（step/工具/流式细节）
- 未来 6_context 篇：instructions 子系统与请求组装（model-request.ts / to-llm-message.ts）
- 未来 7_compaction 篇：摘要算法、原生窗口、overflow 恢复
