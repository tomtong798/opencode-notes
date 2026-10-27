# 并发正确（Concurrency）

> 一句话：**opencode 的并发正确性不是一把大锁，而是一张"每层只管一件事"的串行化层次——同聚合事件用 KeyedMutex 排队、同会话 drain 用门铃协调器合并、跨会话天然并行、工具 fiber 并行干活但发布器串行记账、进程边界靠托管协议 + SQLite WAL 排除而不是防御——所有原语都是 Effect 的（fiber/Deferred/Latch），没有一行手写锁。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，sed 抽查 ≥18 处）。
> 权威出处：`packages/core/src/session/run-coordinator.ts` 头注释（"Serializes execution for each key while allowing different keys to run concurrently"）+ `packages/core/src/session/execution/restart.ts:46-61` 头注释（托管协议三保证）。
> [返回 README](README.md) | 关联 [../6_modules/0_effect-infra.md](../6_modules/0_effect-infra.md) §6（KeyedMutex 本体）/ [../6_modules/5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) §5（run-coordinator 全图）/ [5_resource-limits.md](5_resource-limits.md) §2.3（并发三层体系）/ [../6_modules/9_tool.md](../6_modules/9_tool.md)（工具执行管线）
> 本篇是**并发视角的收口**：各篇已写的机制只引用不重写，这里回答"它们合起来为什么没有竞态、剩下哪些窗口是明知留给协议的"。

---

## 0. 恐惧清单（并发视角怕什么）

单线程世界不会有这篇文章。opencode 是单进程多 fiber 的架构（Effect runtime），所有"并发"都是 green-thread 并发——共享内存、可抢占、无编译器帮检查数据竞争。恐惧按层排列：

```
opencode = 单进程多 fiber + 多进程同库
├─ 恐惧 1：双 fiber 同写一个 inbox 行
│    场景：用户连续两次提交 / steer 变更撞上投递
│    后果：丢事件、重复投递、enqueued_seq 乱序
│    防线：SessionInbox.serialized = KeyedMutex(sessionID)（§2 层2）
├─ 恐惧 2：同会话双 drain
│    场景：wake 撞上 resume、后台 shell 完成唤醒撞上用户发消息
│    后果：同一个 turn 被两个 fiber 各跑一遍、事件交错、LLM 双计费
│    防线：run-coordinator 的 Map<key, Execution> + 门铃合并（§3）
├─ 恐惧 3：事件乱序投影
│    场景：两个 fiber 同时向同一聚合 publish durable 事件
│    后果：seq 分配与投影事务交错，事件序列不再反映因果
│    防线：Bus.durableLocks = KeyedMutex(aggregateID)（§2 层1）
├─ 恐惧 4：工具结果与 Step 终态乱序
│    场景：并行工具 fiber 完成时，Tool.Success 晚于 Step.Finished 落库
│    后果：转录里出现"孤儿工具结果"，重放投影对不上
│    防线：forkScoped 并行执行 + publisher 串行结算 + Fiber.awaitAll 收口（§4）
├─ 恐惧 5：双进程同库
│    场景：两个 server 实例指向同一个 SQLite 文件
│    后果：同会话双 drain（跨进程版）、claim 互踩
│    防线：**不防御，协议排除**——托管注册锁 + sweep 假设 owner 已死 + busy_timeout 兜底（§2 层4）
└─ 公共前提：全部原语来自 Effect（fiber/Semaphore/Deferred/Latch/Queue），
     单线程事件循环内调度——不存在真并行内存写，竞态窗口全部是
     "await 点之间的交错"，这让"锁内不可中断"成为可实现的纪律。
```

关键认知：**Effect fiber 不是线程**。Bun/Node 单线程 runtime 里，fiber 交错只发生在异步边界；一段不 yield 的同步代码天然原子。opencode 的并发正确策略因此高度一致：把"检查-决策-提交"放进锁内或 uninterruptible 区域，把可能长的 I/O 留在锁外。

---

## 1. 并发原语地图（全仓 grep 实测）

口径：`grep -rn "KeyedMutex|Semaphore\.|forkScoped|Queue\.|Deferred\.|Latch\.|FiberMap|FiberSet"`，剔除测试。使用点清单：

```
原语                    定义/典型落点（实测）
──────────────────────────────────────────────────────────────
Effect fiber
  fork / forkScoped     core/src 40+ 处（config/plugin/* 全家、shell 清理、
                        location-watcher、model-transport drain……）
  FiberSet.makeRuntime  run-coordinator.ts:72 / mcp/index.ts:164 /
                        workspace.ts:92 —— "受 scope 管的 fire-and-forget"
  FiberMap              llm.ts:55（标题生成 per sessionID 去重）/
                        config.ts:300（config watcher per key）
  Effect.fork           step.ts:76 快照捕获与 provider 请求重叠
Semaphore
  KeyedMutex 封装       8 个文件（下节全列）
  裸 Semaphore(1)       20+ 处：provider loading 锁（openai.ts:237、
                        chatgpt.ts:214、azure.ts:74…）、config.ts:106-107
                        reload/update 双锁、vcs.ts:68 refreshLock、
                        mcp/index.ts:536 reconcileLock、
                        model-transport.ts:136 per-session 通道锁、
                        tool/mcp.ts:33 / websearch.ts:14 / codemode/tool.ts:85
  database 按文件名     database.ts:57-71 locks Map（§2 层4）
Queue                   job.ts（事件分发）、shell.ts:324、event-feed.ts:76
                        Queue.dropping(4096)、permission.ts、form.ts
Deferred                job.ts:55-85 done/backgrounded 双票据、
                        run-coordinator.ts:44 done、workspace.ts:90 attempts
Latch                   plugin.ts:27 ready、mcp/index.ts:81 startup、
                        shell.ts:324 outputDone
EffectFlock（OS flock） util/effect-flock.ts —— 唯一跨进程原语，
                        用于 repository-cache.ts:133 与 plugin update
```

值得记的一条纪律：**裸 `Semaphore.makeUnsafe(1)` 与 `KeyedMutex` 的分工**——单例服务（config、vcs、一个 provider 的加载）用前者；按业务键（path/sessionID/aggregateID/serverName/workspaceID/repository）排队用后者。没有一处用 `Map<K, Promise>` 手搓锁。

---

## 2. 串行化层次（核心交付）

一张大图。每层回答两个问题：**什么被串行、什么仍并行**。

```
┌─────────────────────────────────────────────────────────────────────┐
│ 层1  同聚合串行 —— KeyedMutex(aggregateID)                           │
│ 落点：bus.ts:200 durableLocks，包住三处（:469 单事件、:575 批量、     │
│       :681 replay）                                                   │
│ 串行：同一 aggregate 的 durable 事件提交（seq 分配+投影事务+通知）      │
│ 并行：不同 aggregate 的事件流水线完全独立                              │
│ 保证：事件 seq 全序 = 因果序（seq 分配与 INSERT 同事务，锁内进行）     │
│ 测试：bus.test.ts "does not interleave a concurrent publish with      │
│       batch notifications" / "increments durable event seq per        │
│       aggregate"                                                      │
├─────────────────────────────────────────────────────────────────────┤
│ 层2  同会话/同键串行 —— 两把互补的锁                                  │
│ (a) inbox 串行：inbox.ts:59 inboxLocks = KeyedMutex(sessionID)，      │
│     serialized() 导出（:62）；admit/steer/queue/cancel/promote 全过   │
│     （:209/:452/:502）；llm.ts:83 边界循环也在锁内读 nextPromotable    │
│ (b) 执行串行：run-coordinator.ts:71 Map<Key, Execution> —— 同 key     │
│     至多一个 owner fiber，期间到达的唤醒进 pendingWake 合并            │
│ 串行：准入决策、投递事务、（每会话）drain 主循环                        │
│ 并行：inbox 事务（短）与 drain（长）可以重叠——锁只包决策不包执行；      │
│       同会话的标题生成 fiber、后台 shell、subagent 子会话各自跑         │
├─────────────────────────────────────────────────────────────────────┤
│ 层3  跨会话并行 —— 无锁，结构保证                                      │
│ 落点：execution.ts 的 coordinator 是进程全局服务，Map 天然按 key 分离； │
│       test "runs different keys concurrently" 钉住                   │
│ 串行：无（这正是设计）                                                 │
│ 并行：不同 Session 的 drain、各自 Location 的服务图、各自的 WS 通道     │
├─────────────────────────────────────────────────────────────────────┤
│ 层4  进程边界 —— 协议排除 + SQLite 兜底                                │
│ (a) 协议：restart.ts:55-61 三保证（头注释 :46 起）——托管注册锁一次一个 managed server； │
│     successor 只在前任确认死亡后拉起（client service kill/evict 轮询   │
│     PID）；未注册进程永不 sweep                                       │
│ (b) 数据库：database.ts:57-71 同文件名共享 Semaphore(1)（进程内跨      │
│     Layer 装配）；PRAGMA busy_timeout=5000（:42）+ WAL（:40）跨进程    │
│     写冲突时等待而非立即失败；迁移层 migration.ts:20-21 注释明说锁在    │
│     Database layer 手里                                                │
│ 串行：同库写事务（最终由 SQLite 单写者模型串行化）                      │
│ 并行：读（WAL）；**但两个未托管进程同时 drain 同一会话不被防御**——     │
│     AGENTS.md："Keep local Session drains process-local until          │
│     clustering is implemented"，claim 是恢复标记不是围栏               │
└─────────────────────────────────────────────────────────────────────┘
```

注意层 4 的诚实边界：[5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) §9.3 已写"双进程同会话的竞态防护几乎没有，靠协议排除"。本篇补充实证：跨进程互斥的真实原语只有 `EffectFlock`（OS 文件锁），且只用在 repository-cache 与 plugin update 这两个"文件系统副作用"场景——数据库写入本身不需要它，SQLite 的 WAL + busy_timeout 已经是仲裁者。

另一个容易误解的点：`BEGIN IMMEDIATE`。@effect/sql-sqlite-bun 客户端的确默认 `beginTransaction: "BEGIN IMMEDIATE"`（SqliteClient.js:174，注释 ：7 "Explicit transactions on writable connections use `BEGIN IMMEDIATE`"），效果是写事务持有写锁全程。但 opencode 的进程内正确性**不依赖**这个——同文件 Semaphore(1)（database.ts:57-71）先于 SQL 层把事务串行化了；busy_timeout 只在跨进程时才真正上岗。

---

## 3. 门铃合并深读：竞态窗口视角

[5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) §5 已画全状态机；本节只下钻一个问题：**pendingWake 置位与 loop 结束之间的窗口，为什么安全？**

危险窗口的直觉版本：

```
fiber A（owner drain）：            fiber B（唤醒方，如 steer 准入）：
  最后一次资格检查（锁内 promote→0）
                                    wake(sessionID, "steer")
                                      executions.get 命中 → 置 pendingWake
  loop 检查 pendingWake → 有 → 再 drain一圈
```

若 B 的置位发生在 A 的"最后一次检查"**之后**、loop 结束**之前**，看起来会不会漏？不会——因为顺序只有两种，两种都安全：

1. **置位在 loop 检查之前**（Effect 单线程调度保证两个 sync 块有序）：loop 的 `Effect.suspend` 读到 pendingWake，吸收为 scope、蹦床再转一圈（run-coordinator.ts:77-83）。安全。
2. **置位在 loop 决定结束之后**：此时 owner 已退出，`settle`（:116-119）在 executions.delete(key) **之前**检查 `execution.pendingWake`——幸存的门铃直接 `start(key, false, pendingWake)` 起后继执行（:114 注释原文："A doorbell that survives the execution loop … starts a fresh execution for the remaining work"）。安全。

窗口不存在的根本原因：**wake 与 settle 都是同步代码块**（wake 是 `Effect.sync`，settle 的判断在 onExit 的 sync 段），在单线程 runtime 里不可能互相插入异步间隙。真正的微妙点不在"漏"，而在**结算窗口的 interrupt**（:152-156 注释 "Settlement window: the owner exited but the settled hook has not finished"）：此时终态已定，interrupt 返回 false 但没收 pendingWake——防的就是"死意图的后继在 settle 时复活"。测试逐一钉住：`a settlement-window wake starts a fresh execution with its own scope` / `an interrupt during terminal settlement claims the recorded wake`（session-run-coordinator.test.ts:523 起）。

配套微纪律（[5_inbox §5.3] 已写，本篇只点竞态含义）：loop 与 start 的 leading `yieldNow`（:82/:94-96 注释）不是性能装饰——它保证 owner 赋值先于任何可能同步完成的 drain settle，否则 `interrupt` 会读到一个 `owner === undefined` 的"假结算窗口"。

---

## 4. 工具并行结算：并行干活、串行记账

[9_tool.md](../6_modules/9_tool.md) 已写执行管线；竞态视角只看一件事：**并行执行的工具 fiber 如何保证事件顺序正确？**

```
Step.attempt（step.ts:57 起）
├─ providerStream = Stream.runForEach（单消费 fiber，顺序天然）
│    每个 tool-call 事件：
│    ① publisher.publish(Tool.Called)     ← 同步在流消费内：先记账后启动
│    ② executeTool(call) 包 uninterruptibleMask + forkScoped（:125-133）
│         → 并行干活：N 个工具 fiber 同时跑（权限、I/O、子会话……）
├─ 工具 fiber 完成 → publisher.toolExecution(id, name, outcome)
│    （publish-llm-event.ts:617 起）
│    守卫三连（:619-623，实测）：called 必须存在 / name 必须一致 /
│    settled 幂等（"Duplicate tool execution" defect）
└─ 收口：Fiber.awaitAll(toolRuns)（step.ts:151）→ classifyToolExits
     → 终态事件（Step.Finished/Failed/Aborted）在 awaitAll 之后发布
     （:140 注释："Keep the final tool and Step events uninterruptible,
      even when the work itself is cancelled"）
```

三个设计决定，每个都是竞态防御：

1. **"per-source order without a shared writer queue"**（step.ts:103 注释原文）：provider 事件流和每个工具 fiber 各自保序，发布共用 bus 但**没有中央写队列**——顺序由结构（谁在哪个 fiber 上完成）+ Bus 的 per-aggregate 锁（§2 层1）保证，避免了"队列排序器"这个新竞争点。
2. **记账串行在发布器里**：`toolExecution` 的守卫状态（called/settled）是发布器闭包内的可变量，只在 bus.publish 的同 fiber 段读写；并行的工具 fiber 在此处天然汇入串行发布。双完成、晚完成的乱序尝试都会 die 而不是悄悄写脏。
3. **终态不可中断**：awaitAll 之后、终态发布之前是 uninterruptible 区——中断只能取消"活"，不能取消"记账"。被中断的工具由 `TOOLS_INTERRUPTED`/`failUnsettledTools` 补结算，转录永远闭合。

对照阅读：[../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §5 的时序图是这套机制的纵向切片。

---

## 5. 竞态窗口清单（代码注释自认的窗口）

grep `race|window|serializ|concurrent` 注释，源码自己承认的窗口/防御，按"已关闭/明知开放"分栏：

| # | 位置 | 窗口 | 处置 |
|---|---|---|---|
| 1 | config/plugin/entry-observer.ts:22 | 首读与订阅建立之间 config 可能变更 | 已关闭：订阅先建、再重读一次（"Close the race…"） |
| 2 | config/plugin/agent.ts:59 / command.ts:48 | 配置扫描与变更通知并发 | 已关闭：单 trigger feed 串行化 + 共享 debounce 窗口，" subscribing before the initial scan means updates racing the scan still trigger a rebuild" |
| 3 | location-services.ts:39-42 | boot 失败驱逐撞上并发 invalidate | 已关闭：build 身份比对（"An explicitly invalidated build must not evict its replacement"）+ "Evict once per failed build, before its result reaches borrowers"；在飞的 detached boot 走自己的 shutdown（:33） |
| 4 | workspace.ts:320-326 | destroy 与 provision 竞态 | **明知开放**：先结算共享 Deferred 取消 racing provision，注释自认"a provision racing this window may briefly succeed before teardown destroys its fresh binding"——接受 |
| 5 | cold wake| cold wake 与 janitor | TODO 自认："make cold wake interruptible with a re-pin loop against janitor races" |
| 6 | session.ts:305 | 并发创建同 ID 会话 | 已关闭："Concurrent creation lost the projection race. The existing Session identity wins."（投影冲突即输） |
| 7 | plugin/update.ts:50-51 | 并发更新清掉彼此的 in-progress 标志 | 已关闭：per-target KeyedMutex（"Serialize per target…"） |
| 8 | run-coordinator.ts:152 | 结算窗口 interrupt | 已关闭（§3 本篇） |
| 9 | mcp/index.ts:167 | 持生命周期锁时关闭 scope 触发 onClose 回调 | 已关闭且有纪律：从连接回调拿锁的必须 fork（注释原文），防锁内重入死锁 |
| 10 | file-mutation.ts:63-65 | 进程内串行 vs 外部写 | **明知开放**："Mutation locking is process-local…external writes can still race" |
| 11 | model-transport.ts:455 | WS 错误后重试撞握手 | 已关闭：错误即弃连接，"retries never race that handshake" |
| 12 | restart.ts:46-61 | 双进程同库 | **明知开放**（§2 层4）：at-least-once、"local coordination prevents concurrent drains, not repeated external side effects" |
| 13 | database.ts:30-34 | workerd 多 Durable Object 共享模块级 semaphore | 已关闭：bootstrap 锁 scope 到 database 而非模块（注释解释跨对象 I/O 拒绝问题） |

读法：opencode 不假装消灭所有窗口——**能用身份比对（#3/#6）或协议排除（#12）解决的绝不上重锁；解决不了的在注释里写明接受的代价（#4/#10）**。这是全仓最值得抄的并发文化。

---

## 6. 失效场景矩阵

| # | 失效场景 | 撞上的防线 | 残余风险 |
|---|---|---|---|
| 1 | 双 fiber 同时 admit 同一 inbox ID | serialized 锁 + reconcile 幂等（first admission wins） | 无 |
| 2 | wake 风暴（shell 完成+用户输入+subagent 同时唤醒） | 门铃合并取最宽 scope（run-coordinator.ts:141） | 无——一次 drain 覆盖 |
| 3 | 工具 fiber A 成功、B 失败、C 被中断 | awaitAll + classifyToolExits 三分类 | 无——终态闭合 |
| 4 | 并行工具重复发布同一 call 结果 | toolExecution 的 settled 守卫 → defect | 无 |
| 5 | drain 中进程被 SIGKILL | claim 留库即崩溃签名；下次托管 sweep 恢复 | 外部副作用可能重放（at-least-once，明知） |
| 6 | 两个未托管进程同库同会话 | **无防线**——事件 seq 仍全序（SQLite 串行化写入）但 drain 会交错 | 官方排除：等 clustering |
| 7 | 迁移与业务写并发 | 同文件 Semaphore 跨 Layer 共享 + 迁移在锁内 | 无 |
| 8 | 配置热重载与首次扫描并发 | 单 feed 串行 + debounce（agent/command 同款） | debounce 窗口内多次变更合并为一次重建（预期行为） |
| 9 | WS 通道检查点与连接轮换竞态 | per-session Semaphore(1)（model-transport.ts:136,:522）+ token 身份比对（complete :501-504） | 无——旧通道的 complete 落空 no-op |
| 10 | 并发 OAuth refresh 同一 token | KeyedMutex(url+client+token)（mcp/oauth.ts）共享同一次刷新 | 无（测试 "shares concurrent refreshes for the same token"） |
| 11 | 并发 npm 更新同一插件目录 | per-target KeyedMutex + EffectFlock 跨进程 | 无 |
| 12 | 慢 SSE 消费者 | Queue.dropping(4096) 牺牲慢者，重连重取 | 该订阅者断连一次（设计取舍） |

---

## 7. 测试覆盖（并发测试实测）

```
防线                        测试（实测存在）
────────────────────────────────────────────────────────────
run-coordinator             core/test/session-run-coordinator.test.ts
                            28 个 it.effect，686 行。并发面逐条：
                            joins concurrent resumes / coalesces wakes /
                            runs different keys concurrently /
                            a settlement-window wake starts a fresh
                            execution / an interrupt during terminal
                            settlement claims the recorded wake /
                            starts one follow-up when a wake races with
                            failure / trampolines synchronous
                            self-waking execution …
执行/恢复（claim 生命周期）   core/test/session-execution.test.ts
                            "starts every claimed execution without
                            waiting for earlier drains to finish" /
                            "resumes each claimed Session at most once" /
                            "counts every resume durably and never
                            consumes the claim it recovers" /
                            "resumes a background subagent and notifies
                            its parent exactly once"
KeyedMutex 本体              core/test/effect/keyed-mutex.test.ts
                            serializes same key / different keys
                            independent / interrupted waiter 不掉锁
inbox/事件聚合               bus.test.ts "does not interleave a
                            concurrent publish with batch notifications"
                            session-create.test.ts "returns one recorded
                            session to concurrent exact retries"
                            session-compact.test.ts "coalesces concurrent
                            manual compaction"
文件事务                    file-mutation.test.ts:113 "serializes
                            concurrent writes to the same absolute
                            target"（it.live）
工具事件序                  session-runner-tool-events.test.ts 全套
                            （批 delta、终态前 flush、守卫）
WS 通道                     session-model-transport.test.ts "serializes
                            concurrent calls for one Session" /
                            "isolates connections and permits
                            concurrency across Sessions"
workspace                   workspace.test.ts "shares provisioning
                            between concurrent first spawns" /
                            "interrupts in-flight provisioning on
                            destroy…"
MCP                         mcp.test.ts "serializes concurrent MCP
                            lifecycle operations" / "joins concurrent
                            stdio transport closes"
OAuth                       mcp-oauth.test.ts "shares concurrent
                            refreshes for the same token"
repository-cache            repository-cache.test.ts "serializes
                            concurrent materialization for the same
                            checkout"
```

测试风格值得一提：并发测试几乎全部用 **Deferred 门闩手工制造交错**（先放行 fiber A 到指定点，再触发 B，断言合并/串行结果）——不是概率性压力测试，是把竞态窗口钉成确定时序的编排测试。

---

## 演进注记

对照旧卷：KeyedMutex 机制归 [0_effect-infra.md](../6_modules/0_effect-infra.md) §6、门铃状态机归 [5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) §5、资源限额归 [5_resource-limits.md](5_resource-limits.md) §2.3——本篇为**新增域**：首次把散落的原语按"串行化层次"收口成一张总图，并引入此前未成文的实码：Bus durableLocks 三落点（bus.ts:469/575/681）、location-services 的 build 身份比对驱逐（:39-42）、workspace destroy/provision 竞态的自认取舍（:319-326）、step.ts 的 per-source order 注释（:103）、BEGIN IMMEDIATE 的真实归属（@effect/sql-sqlite-bun 而非 opencode 自己的代码）、EffectFlock 作为唯一跨进程原语。v1 时代无 run-coordinator（执行协调内联在 session 里），门铃合并与结算窗口语义是随泛型化重构进入的。

## 相关篇

- [../6_modules/0_effect-infra.md](../6_modules/0_effect-infra.md) §6 — KeyedMutex 本体实现（本篇 §1/§3 只列落点）
- [../6_modules/5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) §5/§9 — run-coordinator 全图与跨进程诚实边界（本篇 §3 下钻竞态窗口）
- [5_resource-limits.md](5_resource-limits.md) §2.3 — 并发三层体系（数值并发/会话串行/键级互斥，本篇换"串行化层次"视角重组）
- [../6_modules/9_tool.md](../6_modules/9_tool.md) — 工具执行管线（本篇 §4 只写竞态视角）
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §5 — 工具 fiber 并行执行时序（§4 的纵向切片）
- [6_security.md](6_security.md) §3.4 — 数据库 0600（同为 database.ts 的另一道纪律）
