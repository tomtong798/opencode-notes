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

## 0.5 服务端全生命周期：从进程启动到处理完成

> §0 总图从"回车"开始；本节补上它的前奏与底层——**服务进程自己怎么活着**。四幕叙事，每步标注深入小节。这几轮校准的关键结论都嵌在这里：启动时 Map 为空、没有常驻消费者、执行权进程级而执行能力目录级、真相永远先落库。

### 第一幕：进程启动（一次性的准备）

```
① OS 启动 opencode serve（或被桌面/TUI spawn）
   └ server 装配全局服务图（进程级单例）：
     Database → Bus（事件总线+投影注册）→ Job →
     ★SessionExecution（内含 coordinator，★此刻 Map 为空）
     ★恢复子系统 restart、SessionStore、LocationServiceMap（空表）

② 数据库就绪：WAL 打开、迁移补齐
   event / event_sequence / session / session_inbox / session_message
   ——上次崩溃留下的 durable 事实全在这里

③ HTTP 端口绑定，status.ready
   ★此刻：没有任何 Location 图、没有任何执行 fiber、Map 空
```

### 第二幕：启动恢复（一次性扫描，跑完即止）

```
④ resumeSuspendedSessions（restart.ts:193）扫两处：
   ├ session 表里带未释放 claim 的会话
   │   （claim = 执行前写的"我正在跑"durable 标记；
   │     正常完成/失败/用户中断会释放，崩溃/被杀才残留）
   └ Job 表里 running 状态的后台任务（shell/subagent）

⑤ 恢复顺序（:212-218）：先收 shell（补录结果）→ 再收 subagent
   （releaseChildClaims）→ 最后逐个处理带 claim 的顶层会话：
   ├ countResume 查预算（>10 次 → Failed 终结，放弃）
   ├ 准入 synthetic"继续"消息（durable 先行）
   └ execution.resume → wake ——进入第三幕通用流程
   ★除此之外没有常驻消费者——扫描完，这个 fiber 也散了
```

### 第三幕：稳态运行（每个请求的通用流程）

前半段秒回（细节 → §2）：

```
⑥ POST session.prompt（客户端已铸 id）
   └ reconcile 幂等 → prepare →
     bus.publish(inbox.enqueued)★同事务插 session_inbox 行
     （输入已不可丢失）→ wake —— HTTP 秒回

⑦ wake（run-coordinator.ts:136）：
   ├ Map 有条目 → 只置 pendingWake（现有循环会再跑一轮）
   └ 没有 → start：登记 + 发 Started + ★写 claim +
     fork drain fiber ←★消费者此刻才诞生（每会话一个）
```

后半段排空（细节 → §3/§4/§5）：

```
⑧ drain：按 sessionID 查 store 得 location →
   从 LocationServiceMap 借该目录的图
   （首次访问 → 现场编译 53 服务，之后缓存）
   └ 在借来的 runner 里循环：

⑨ 循环体（llm.ts:74-199）：
   ├ 门检查：无可提升 → Complete 跳出
   ├ ★promote：steer 批止于控制项；空闲放行一个 queue；
   │   删行+插消息同事务；提升成功 → 步数预算重置
   ├ runStep：组装请求 → llm.stream → 逐事件发布+投影
   │   ├ 工具：每 tool-call 一个专属 fiber 并行，结果先落盘再发布
   │   ├ 模型还要工具 → 回填 → ★回到 ⑨ 再 runStep
   │   │   （一次排空可含多次 LLM 调用）
   │   └ 不再要工具 → Step.Ended（usage 落账）
   └ 本轮结束：pendingWake 响过？→ 蹦床再跑一轮
      否则 settled：发终态 + ★释放 claim（同事务）+
      Map 移除条目 + fiber 自我终结（★消费者消亡）

⑩ 与 ⑧⑨ 并行：每个事件经 Bus → SSE → client/solid 投影
   → 组件重渲染（→ §6 回程七跳）
```

### 第四幕：异常路径（同一骨架的变奏）

| 事故 | 发生什么 |
|---|---|
| 用户 Esc | interrupt 沿 fiber 树传播；工具记账段不可中断保完整；Interrupted + claim 释放（pending inbox 项不删） |
| 会话迁移 | move 控制项消费 → drain 返回 Moved → ★借新目录 runner 带状态重入（执行权进程级不动，执行能力换目录） |
| LLM 限流 | retry 分类退避（预算内），耗尽才 Failed |
| ⑨ 中途 kill -9 | claim 残留（故意保留）→ 下次启动回到第二幕 ⑤ |
| 目录长期不用 | 什么都不发生（图永不过期，仅显式 invalidate） |

**一段话总结**：启动只建基座（Map 空、无消费者）→ 一次性恢复扫描（claim 残留者重新 wake）→ 稳态由 wake 点火：准入落库秒回、fork 该会话专属 fiber、借它所在目录的 runner、循环"提升 inbox → Step（LLM+工具并行）→ 事件投影"直到无活、门铃没响就释放 claim 并解散 fiber——**没有常驻调度者，一切按需诞生、用完即散；真相永远先落库，内存随便丢**。

---

## 1. 浏览器侧：提交

### 1.0 前置：composer 与 adapter

**词义**：composer（作曲家/创作器）是 LLM 应用社区从邮件客户端借来的惯用词——**消息创作区**。应用界面里"读"的部分叫 timeline（对话时间线），"写"的部分就叫 composer：文本编辑、附件拖拽、@引用、agent/model 切换都在这块。`packages/app/src/composer/` 即此概念的代码统称。

**`composer/adapter.ts`（120 行纯类型，零逻辑）**——composer 与两种会话环境之间的端口适配层，让 `submit()` 写一次服务两个场景：

```
ComposerAdapter = ActiveComposerAdapter ∪ NewSessionComposerAdapter
                  （判别字段 kind）
│
├─ 公共底座（:96-102）
│    state ——输入框状态（草稿文本/附件，可捕获快照）
│    ready / working / controls / submitted
│
├─ ActiveComposerAdapter（:104-109，活跃会话页）
│    kind: "active-session"
│    session() ——ComposerSession（:52-94）能力包：
│      api（command/shell/switchAgent/switchModel/revert）
│      data（session.prompt 乐观准入、setStatus）
│      handoff / current / admitted
│    interrupt() ——中断当前 turn
│
└─ NewSessionComposerAdapter（:111-118，新会话草稿页）
     kind: "new-session"
     start() ——首次提交的"建会话"入口：
       传入选择+提交状态+用户消息，
       返回 {session, cleanupReady, complete?}
       ——成功后才切换成 active 形态
```

两个实现与一个消费者：

| 角色 | 位置 | 职责 |
|---|---|---|
| 实现 A | `session/composer/adapter.ts` `createActiveComposerAdapter` | 从已存在 sessionID 组装（working = `data.session.status(id)==="running"`） |
| 实现 B | `new-session/composer-adapter.ts` `createNewSessionComposerAdapter` | draft 页组装（含 worktree/branch/MCP 控件），`start()` 里建会话 |
| 消费者 | `composer/submit.ts` | :73 典型判别消费：`adapter.kind === "active-session"` 才走 steer 追踪分支 |

`submit.test.ts:55/69` 的 `active()`/`fresh()` 两个构造器靠此端口注入假环境——adapter 同时是天然的测试边界。

### 1.1 `createComposerSubmit` 的提交流

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
              （.catch 重试一次——幂等靠铸 id + 服务端 reconcile）
```

**"app 怎么调到 client"——不是运行时转发，是四步依赖注入**。`session.data.session.prompt` 这个方法本来就长在 client/solid 的 Data 对象上；app 启动时把 HTTP 客户端注入给它，再经引用传递到 submit.ts 手里：

```
① 每服务器一次的装配            app/runtime/server/runtime.tsx
│
├─ L172  sdk = createServerSdkContext(conn, scope)
│        （内部 OpenCode.make({baseUrl, headers: Basic})）
│
└─ L174-185 createData({           ←client/solid 工厂
      api: () => sdk.api,          ★promise 客户端注入数据层
      event / connection })
      产出 Data 对象（含本节的 session.prompt 方法）

② Solid Context 下发             app/runtime/server/current.tsx
│
└─ <context.provider ctx={{data}}>
   useData() = useServer().ctx.data（:50-53）——组件树内随处可取

③ adapter 装配（每会话）          app/session/composer/adapter.ts
│
└─ L21  data = useData() → :38 塞进 ComposerSession.data

④ 提交终点                       app/composer/submit.ts
│
└─ L545  session.data.session.prompt(admission)
         真正发 HTTP 的在其内部的
         api().session.prompt(...)（data.ts:1646）
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

### 3.0 是什么、为什么（三问校准）

**wake（唤醒）= 按门铃**——告诉协调器"这个会话有新活儿了"；**drain（排空）= 把活干完**——把该会话积压的待办（inbox 输入、待续 Step）一轮处理到没有可做的为止。准入与执行分离后需要一座从"事实落库"到"开始消费"的桥，wake 就是其最小（advisory）形态。

**为什么是这个形状**——五个不可选约束的联立解：

```
要崩溃安全   → 准入/执行分离（durable 事实 vs 消费）── spec:11
要 HTTP 秒回 → wake 仅建议性触发，不等结果（advisory）
要会话内顺序 → 同 key 串行；多 wake 折叠成"再跑一轮"── spec:60
要堵竞态缝   → pendingWake 门铃：drain 的"最后一次检查"与
               "宣布空闲"非原子（run-coordinator.ts:31-37
               注释原话 "closes the gap …cannot be one
               atomic step"），门铃补上这个窗口
要 steer 批  → drain 排空到自然空闲，期间到的 steer 收进
               下一批；蹦床循环（L81）不涨栈
```

**与生产者/消费者模式的辨析**（骨架像，三处不同）：

| 维度 | 经典 MQ | 这里 |
|---|---|---|
| 取的规则 | 消费者全取/逐条 | **投递词汇**：steer 批到首个控制项边界；queue 空闲时**恰好一个**再重评估（spec 原话 "exactly one queued item delivers"） |
| 消费单位 | 一条消息一次处理 | 一次提升触发 **Step 链**（内层 LLM 步进循环，工具结果回填续跑，不碰 inbox） |
| 消费者生命周期 | worker 常驻 | **drain 完即散，wake 再点火**（executions Map 是易失的） |

**并发粒度（三层）**：session 间并行（一 session 一条 `executions Map` 条目 = 一个 drain fiber + 各自门铃，spec:61 "Different Sessions run concurrently"）；session 内串行（drain→Step 链按序）；Step 内工具并行（forkScoped）+ 发布串行。

**inbox 在哪**：**持久化**——`session_inbox` 表（sql.ts:125-143），入队/消费与事件同事务。内存中的 pending 镜像、fiber 注册表全是易失派生（崩溃后靠表行 + claim 恢复续跑）；若 inbox 在内存，"秒回后 kill -9 消息蒸发"将是最恶劣的失败形态——持久化是第一性决策（详见 [../6_modules/1_data-layer.md](../6_modules/1_data-layer.md) §9.5 分界表）。

### 3.1 门铃模式的运行流

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

一次 `text.delta` 从 Bus 到屏幕的七跳管道（跨服务进程 → renderer 进程边界）：

```
【服务进程】
│
├─ ① core/src/bus.ts
│    publish = durable 追加 + 投影事务 + 订阅分发
│
├─ ② server/src/event-feed.ts:33-81
│    Bus.listen → SSE 帧（data: {json}\n\n，L30）
│    → 订阅者队列
│
├─ ③ handlers/event.ts:8-36
│    SSE 端点：connected 先发 + 15s 心跳合并
│
│    ════════ HTTP/SSE（跨进程）════════
│
【renderer 进程】
│
├─ ④ client/solid/connection.ts:76-145
│    迭代读取 → touch()（喂 watchdog）
│    → publish() 10ms 批量 flush
│
├─ ⑤ app/runtime/server/client.tsx:89-105
│    events.publish → createGlobalEmitter（本地总线）
│
├─ ⑥ client/solid/data.ts:2007-2011
│    event.listen → handleEvent
│    → setStore(produce(...))
│
└─ ⑦ Solid 细粒度响应
     读 store.session.message[id] 的组件自动重渲染
     （text.delta：editText(t => t.text += delta)，L952）
     ——组件只跑一次，信号级精确更新 → 像素
```

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
