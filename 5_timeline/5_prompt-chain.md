# 提问执行链：从回车到流式渲染

> 一句话：**回车 → 乐观入 store → `POST session.prompt` → 服务端幂等准入（durable `session.inbox.enqueued` + `session_inbox` 行）→ `wake`（门铃合并）→ drain 循环提升 inbox → Step：一次 `llm.stream` 流式 + 工具并行 fiber → 事件经 Bus→SSE→本地总线→store 投影驱动渲染。**
> 代码基线：v2.0.30，`dev` 分支；行号实测（2026-10-16）。
> [返回 README](README.md) | 上一篇 [4_renderer-boot.md](4_renderer-boot.md)

---

## 0. 全链路总图

```
[浏览器]                          [服务端]                       [持久层]
回车 submit.ts:57
 ├ clonePrompt/校验/会话解析
 ├ 乐观 admitLocal() ──────────────┐
 └ POST session.prompt ────────────┼─► handlers/session.ts:308
                                   │   Session.prompt (session.ts:145)
                                   │    ├ reconcile 幂等对账
                                   │    ├ SessionPrompt.prepare (prompt.ts:29)
                                   │    │   插件钩子 ∥ 附件物化(8并发,20MB) ∥ 技能解析
                                   │    ├ revert commit（若有暂存还原）
                                   │    ├ admission.admit (inbox.ts:169)
                                   │    │   └ bus.publish(InboxEnqueued)
                                   │    │       ├ 投影: session_inbox 插行（同事务）
                                   │    │       └ ──────────────────────► [SQLite]
                                   │    └ execution.wake (run-coordinator.ts:136)
                                   │        门铃合并；空闲则 start(fiber)
                                   │            └ loop → drain (execution.ts:87)
                                   │                └ SessionRunner.drain (runner/llm.ts:57)
                                   │                    ├ nextPromotable 门检查
                                   │                    ├ settle 陈旧压缩/工具
                                   │                    ├ advanceToStep：
                                   │                    │   prepareContext →
                                   │                    │   SessionInbox.promote ──► InboxDelivered
                                   │                    │   （删 inbox 行+插消息，同事务）
                                   │                    └ while: runStep (L209)
                                   │                        ├ 组装请求（transcript+tools）
                                   │                        └ SessionStep.attempt (step.ts:73)
                                   │                            ├ createLLMEventPublisher
                                   │                            ├ llm.stream ──► [LLM Provider]
                                   │                            ├ Stream.runForEach 逐事件:
                                   │                            │   publish(event)（delta 100ms 批）
                                   │                            │   tool-call → forkScoped 工具 fiber
                                   │                            └ Step.Ended（usage/finish）
◀── text.delta ───────────────────────── bus.publish → EventFeed(SSE帧) ─┤
SSE 收 → 10ms 批 → 本地总线.publish                                        │
 → handleEvent: store.produce(text += delta)                              │
 → Solid 细粒度响应 → 渲染 ◀──────────────────────────────────────────────┘
```

---

## 1. 浏览器侧：提交

**`packages/app/src/composer/submit.ts`** —— `createComposerSubmit`（L56，返回 `{submit, stop}`）的提交流：

```
submit()
│
├─ L60      clonePrompt(adapter.state.current())（克隆输入）
│
├─ L62-88   客户端命令（如 /btw）分流处理
│
├─ L102-108 readSubmission() 校验
│            （文本/图片/附件齐备；模型 + agent 已选）
│
├─ L111-115 shell 命令不能 queue（delivery 校验）
│
├─ L136-139 ★ 会话解析
│            ├ 活跃会话 → 直接使用
│            └ 新会话   → adapter.start()（先建 session）
│
├─ L163-165 乐观置 running（若当前空闲）
│
└─ L167-170 sendPrompt(session, value, track)
```

**`sendPrompt`（L493-546）**：

```
sendPrompt
│
├─ L498      buildSubmissionRequest（text/files/agents/skills）
│
├─ L507-542  组装 admission
│             （★客户端铸造的 id、delivery、
│              prepare: () => commit()）
│
└─ L545      session.data.session.prompt(admission) → data 层
```

**`packages/client/src/solid/data.ts:1611-1655` `session.prompt`** —— 乐观准入流：

```
session.prompt(admission)
│
├─ L1613-1641 ★ 乐观准入
│             ├ 新 id 入 outbox 集合（未确认跟踪）
│             └ admitLocal() → 界面立即渲染
│
└─ L1642-1654 sendAdmission
              ├ ★同会话串行（sending Map L370——保准入顺序）
              ├ api().session.prompt(POST)
              └ 失败且服务端从未确认 → 回滚乐观行
                （服务端已确认过的不回滚——outbox 的职责）
```

## 2. 服务端准入

**`packages/server/src/handlers/session.ts:308-343`** `session.prompt` handler → 错误映射（NotFound/PromptConflict/Attachment/SkillNotFound/location errors）。

**`packages/core/src/session/session.ts:145-178` `Session.prompt`**（uninterruptibleMask 内）的准入流：

```
Session.prompt(sessionID, input)
│
├─ L148   get(sessionID)（无则 NotFoundError）
├─ L149   messageID = input.id ?? create()
│         （★客户端铸 id 上门）
│
├─ L151-157 ★ admission.reconcile —— 幂等对账
│            同 id + Session + type 已存在 → 直接返回已有
│            （重试幂等：第一次准入获胜，负载/投递模式被忽略）
│
├─ L158-163 restore(SessionPrompt.prepare(...))
│            （可中断段；注入 Instance / FSUtil）
│
├─ L165   有暂存 revert → 先 SessionRevert.commit
│
├─ L166-170 admission.admit(...)
│            └ LifecycleConflict → PromptConflictError
│
└─ L174   ★ input.resume !== false 才 execution.wake(sessionID)
```

**`SessionPrompt.prepare`（`session/prompt.ts:29-91`）**：L38 等插件激活 → L39-51 `session.prompt` 钩子 → L53-55 `materializeAttachment` **并发 8、单件 20MB** → L56-76 技能解析 → 产出 `SessionInbox.Item{type:"user"}`。

**`SessionInbox.admit`（`session/inbox.ts:169-202`）**：L174 再对账 → L176-198 `bus.publish(SessionEvent.InboxEnqueued)` → 投影 `projectAdmitted`（L277-317）：`session_inbox` 插行（`onConflictDoNothing`），**与事件发布同事务**。

## 3. 唤醒与 drain

**`SessionExecution.wake`（`session/execution.ts:150-174`）** → 委托协调器。

**`SessionRunCoordinator`（`session/run-coordinator.ts`）**——门铃模式的运行流：

```
wake(key, scope)（L136-145）
│
├─ 已有执行在跑？
│   ├ 是 → 合并进 pendingWake（input 吸纳 steer）
│   │       ★多次唤醒折叠为"再跑一轮"标记
│   └ 否 → start(key, false, scope)
│
└─ start（L87-112）
    └ fork fiber：
        started 钩子（发 Execution.Started）
        → loop（L74-85）
            │
            ├ drain()（一轮排空）
            │
            └ drain 结束后：
                ├ pendingWake 置位？
                │   ├ 是 → 更新 scope → 再跑一轮 loop ★门铃
                │   └ 否 → 结束（settled 钩子）
```

**`drain`（`execution.ts:87-110`）**：`SessionRunner.Service.use(runner.drain)` 并 `instances.provide(session)`（按会话 Location 提供实例）；结果 match：`Complete` 止 / `Moved` 换 Location 递归 / `Reloaded` 带标记重跑。

## 4. Runner：提升与步进

**`SessionRunner.drain`（`session/runner/llm.ts:57-199`）** —— 排空一个会话的执行流：

```
SessionRunner.drain({sessionID, force, continuation, promotable})
│
├─ L57-69   ★ 门检查（不满足直接 Complete）
│            ├ 无可提升项 → Complete
│            └ promotable==="steer" 且下一项是 queue
│              （非控制项）→ Complete
│              （steer 优先、queue 等空闲）
│
├─ L70-72   收陈旧（上进程遗留的 running 投影判失败
│            ——崩溃恢复的一部分）
│            ├ settleStaleCompactions
│            └ settleStaleToolCalls
│
└─ L191-199 ★ 主循环 while(true)
             │
             └ advanceToStep()（L74-189）
                 │
                 ├ 控制项优先：
                 │   ├ compaction → 直接执行（发 Started）
                 │   └ move → 关 model transport → 发 Moved
                 │
                 ├ 否则提升输入：
                 │   ├ prepareContext(sessionID)
                 │   └ ★ SessionInbox.promote（inbox.ts:496-531）
                 │       ├ steer 批提升至首个控制项边界
                 │       ├ 空闲边界：提一个 queue
                 │       │ 再收新到 steer
                 │       └ 每行发 InboxDelivered + 删行（同事务）
                 │
                 ├ 提升了输入且标题未起
                 │   → fork 标题生成（FiberMap onlyIfMissing）
                 │
                 └► Ready → runStep(context, step)
                     │
                     └ L209-330 runStep
                         ├ L211 铸 assistant 消息 id
                         ├ L212 Retry 协调器
                         └ 循环：
                             L220 溢出检查
                             （可触发压缩重建同一步）
                             → L226-247 baseTranscript
                               + context.request.primary() 组装
                             → L248-272 steps.attempt（§5）
```

## 5. Step：一次 LLM 调用

**`SessionStep.attempt`（`session/runner/step.ts:73-286`）** —— 一次 LLM 调用的执行流：

```
SessionStep.attempt(input)
│
├─ L76-85   createLLMEventPublisher
│           （含 pendingStartSnapshot：流开始前的文件快照 fiber）
│
├─ L107-138 ★ llm.stream(request).pipe(
│             Stream.runForEach(event => …),
│             Effect.ensuring(publisher.flush()))
│             │
│             └ 逐事件处理：
│                 ├ publish(event)（LLM→Session 事件映射，
│                 │               delta 100ms 批量）
│                 └ tool-call（非 provider 执行）→
│                     Effect.uninterruptibleMask(
│                       restore(执行工具)
│                       → 截断 → publisher 发布
│                       → catchTag Tool.Error → failTool
│                     ).pipe(forkScoped)
│                     ★本地工具并行 fiber、结果经 publisher
│                       串行发布（规范：每个本地调用先持久化
│                       再副作用；终端结果在 Step 单一
│                       ended/failed 之前落盘）
│
└─ L141-285 流后处理
             ├ finish reason 解析
             ├ 重试判定（runner/retry.ts：限流/传输/
             │ 不完整流；初始 + 至多 4 次退避）
             ├ 溢出恢复（可触发压缩重建同一步）
             └ Step.Ended（usage）
```

**publisher（`runner/publish-llm-event.ts:105-680`）**：LLM 事件 → Session 事件的映射中枢——text/reasoning/tool-input 三类 delta **100ms 批量**（L106）；`step-finish` 冲刷并记 finish；本地工具结果走 `toolExecution`（L617-634）发 `Tool.Success`。

## 6. 回程：事件到像素

| 跳 | 位置 | 机制 |
|---|---|---|
| 1 | `core/src/bus.ts` | publish = durable 追加 + 投影事务 + 订阅分发 |
| 2 | `server/src/event-feed.ts:33-81` | Bus.listen → SSE 帧（`data: {json}\n\n`，L30）→ 订阅者队列 |
| 3 | `handlers/event.ts:8-36` | SSE 端点：connected 先发 + 15s 心跳合并 |
| 4 | `client/solid/connection.ts:76-145` | 迭代读取 → `touch()` → `publish()` 10ms 批 |
| 5 | `app/runtime/server/client.tsx:89-105` | `events.publish` → createGlobalEmitter |
| 6 | `client/solid/data.ts:2007-2011` | `event.listen → handleEvent` → `setStore(produce)` |
| 7 | Solid 细粒度响应 | 读 `store.session.message[id]` 的组件自动重渲（`text.delta`：`editText(t => t.text += delta)`，L952） |

## 7. 关键不变量（对照 specs/v2）

| 不变量 | 落点 |
|---|---|
| **准入与执行分离** | prompt 只写 inbox 行 + advisory wake；`resume:false` 仅准入 |
| **幂等** | 客户端铸 id + 双层 reconcile（session.ts L151 / inbox.ts L174）；重试负载被忽略 |
| **一步一流式调用** | 每个 Step 一次 `llm.stream`，从重载历史继续；**无内存 tool loop**（工具结果 → durable → 下一 Step 重载） |
| **durable 先于副作用** | 工具调用先落盘再执行；Step 单一终端事件 |
| **中断语义** | Effect interruption 贯穿（工具 uninterruptibleMask 只护发布段）；interrupt 删 pending 输入之外的状态 |
| **投递词汇** | steer 安全边界插入/queue 空闲投递；控制项（压缩/move）是投递边界 |

## 相关篇

- [6_tui-cli-prompt-chain.md](6_tui-cli-prompt-chain.md)——同一后端契约的 TUI/CLI 侧（串行队列、session.wait idle fence、CLI 投影对账）
- [4_renderer-boot.md](4_renderer-boot.md)——store 与连接层（本篇的回程基础设施）
- [2_server-init.md](2_server-init.md)/[3_location-lazy.md](3_location-lazy.md)——服务端前置
- [../1_architecture/1_overview.md](../1_architecture/1_overview.md) 附录 A——本链的服务端半部概览
- `specs/v2/session.md`——全部语义的权威出处
