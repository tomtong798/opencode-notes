# 崩溃安全（Durability / Crash Safety）

> 一句话：**opencode 对"进程任意时刻死掉"的回答是一套三层协议——WAL+单事务底座保证磁盘上永远没有半提交状态、write-ahead claim 把"执行到一半"变成 durable 事实、启动 sweep 以 at-least-once 语义续跑——官方明言这不是 exactly-once：崩溃前已发出的副作用可能重放，幂等责任一部分交给"投影幂等三层防线"、一部分交给恢复指令里的模型行为约束。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，sed 抽查 ≥10 处）。
> 权威出处：`AGENTS.md:184`（write-ahead execution claim 原话）+ `packages/core/src/session/execution/restart.ts` 头注释。
> [返回 README](README.md) | 前置 [../6_modules/1_data-layer.md](../6_modules/1_data-layer.md) §4/§9/§9.5 / [../6_modules/4_session.md](../6_modules/4_session.md) §10 / [../6_modules/5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) §6-§7
> 本篇是**崩溃安全视角的收口**：事务序、claim、恢复子系统的机制细节已被上述三篇深覆盖，这里只引用不重写。本篇的新增价值是**崩溃注入点枚举**——沿着一次会话的生命周期，把所有"事务边界之间的窗口"逐一列出：每个窗口崩了、重启后看到什么、恢复动作是什么、残余风险剩什么。

---

## 0. 恐惧清单（这个产品怕什么）

```
恐惧源：kill -9 / 断电 / OOM / isolate eviction —— 共同点是"没有 teardown"
├─ 恐惧 1：磁盘上的状态半写（事件记了、投影没写；或反之）
│    └─ 防线：单 SQLite 事务（投影+seq+事件行同事务，WAL 崩溃恢复粒度=一事务）
│         → [1_data-layer §4](../6_modules/1_data-layer.md)，本篇 §1 一句带过
├─ 恐惧 2：执行到一半的 turn 从此无人知晓——用户回来会话"卡死"
│    └─ 防线：write-ahead claim（busy period 开始时同事务落 time_suspended）
│         + 启动 sweep（本篇 §2/§3）
├─ 恐惧 3：崩溃后重启，模型看到"工具在跑但永远没有结果"的僵尸历史
│    └─ 防线：settleStaleCompactions / settleStaleToolCalls
│         —— 任何模型工作之前把 running 投影判死（本篇 §3 注入点⑤⑥）
├─ 恐惧 4：崩溃恢复本身陷入 crash-loop（恢复路径上再崩，无限拉起）
│    └─ 防线：per-turn 恢复预算 10 次，计数 durable 先行
│         "the budget cannot be dodged"（restart.ts:78-79，本篇 §3）
├─ 恐惧 5：崩溃前后同一副作用执行两次（工具/付费 API 调用）
│    └─ 诚实的答案：不防。at-least-once，非 exactly-once（本篇 §4）
└─ 恐惧 6：两进程共享库并发 drain 同一会话 → 交错事件序列
     └─ 不防，靠协议排除（托管注册锁 + 未注册进程不 sweep）
          → [5_inbox-coordination §7](../6_modules/5_inbox-coordination.md)
```

每个阶段"丢什么/不能丢什么"的统一答案来自 [1_data-layer §9.5](../6_modules/1_data-layer.md) 的分界线：**内存 = 派生缓存（服务图/Location 图/运行 fiber，随进程消失、可重建），磁盘 = 事实记录（event/event_sequence/session_inbox/claim，永久）**。"durable 先于副作用"不变量保证磁盘上的事实记录永远领先于内存状态——内存里没来得及提交的最多是"尚未发生过的事实"，丢了不构成损坏。

用恐惧源本身收一遍（kill -9 / 断电 / OOM 的物理差异 vs 协议视角的无差异）：

```
物理视角                         协议视角
kill -9  ：内核立即回收进程       三者在数据库层面不可区分——
断电     ：页缓存都没刷          都没有 teardown、都留下同样的
OOM      ：SIGKILL by memory     durable 残迹（§2 末的统一签名表）。
                                 唯一的物理差异在 SQLite 之下：
                                 WAL + synchronous=NORMAL 下，
                                 断电可能丢"已返回成功但未 checkpoint
                                 的事务"（fsync 粒度），kill -9 不会
                                 （OS 页缓存仍会刷盘）。
```

这个差异被静默接受（[1_data-layer §2.2](../6_modules/1_data-layer.md) 的吞吐/耐久折中）：断电窗口内丢失的事务对上层表现为注入点①——原子消失、用户重发——协议不需要为它单写分支。

---

## 1. WAL 与事务底座（一句带过）

崩溃恢复的第一层是存储底座：`journal_mode=WAL` + `synchronous=NORMAL`（`database.ts:40-41`），未完成事务在 WAL 恢复时整体消失；每条 durable publish 是一个 `{behavior:"immediate"}` 事务，投影、commit 钩子、seq 推进、事件行追加**同事务提交或整体回滚**。崩溃恢复三条 + 投影幂等三层防线的完整展开在 **[1_data-layer §9](../6_modules/1_data-layer.md)**，本篇不重复。以下所有注入点分析都站在这个底座上：**凡是在事务内的，崩溃 = 整体没发生；凡是在事务之间的窗口，才是本篇的对象。**

---

## 2. 崩溃注入点枚举（本篇灵魂）

一次会话生命周期，`═` 表示事务边界（崩溃在此 = 原子回滚），`▲` 表示注入点（崩溃在此 = 留下跨事务状态）：

```
用户输入
  │
  ▲ 注入点①  准入事务提交【前】
  │   ════ 事务 T1：session.inbox.enqueued 事件 + session_inbox 行插入（同事务）════
  ▲ 注入点①' 准入事务提交【后】、drain 开始【前】
  │
  │  claim：busy period 开始
  │   ════ 事务 T2：Execution.Started 事件 + commit 钩子里 store.claim(time_suspended=now) ════
  ▲ 注入点②  T2 提交后、drain fiber 真正跑起来前
  │
  │  drain 入口（llm.ts:71-72）：settleStaleCompactions → settleStaleToolCalls
  ▲ 注入点③  settle 事务之间/之中（本进程无 stale 时为空转）
  │
  │  ┌─ Step 循环（每 step 一次模型请求 + 若干工具）────────────────────┐
  │  │   ════ T-step：Step.Started 事件 ════                            │
  │  │ ▲ 注入点④  Step 执行中：LLM 流式返回中 / 工具副作用已发生、       │
  │  │    Tool.Completed 未落盘                                          │
  │  │   ════ T-tool：Tool.Completed/Failed + 投影 ════                  │
  │  └───────────────────────────────────────────────────────────────────┘
  │  ┌─ 压缩进行中（compaction 作为控制项入 inbox、被投递后执行）────────┐
  │  │ ▲ 注入点⑤  compaction 消息 status='running'、尚未终态            │
  │  └───────────────────────────────────────────────────────────────────┘
  │  ┌─ 工具 fiber 运行中（后台 shell / subagent 子会话）────────────────┐
  │  │ ▲ 注入点⑥  Job 记录 status='running'、子会话/进程还活着或已死    │
  │  └───────────────────────────────────────────────────────────────────┘
  │
  │  busy period 终态
  │   ════ 事务 Tn：Succeeded/Failed/Interrupted 事件 + commit 钩子 store.release ════
  ▲ 注入点⑦  Tn 提交后——claim 已清、无残余（安全窗口）
  │
会话空闲
```

逐一展开：崩溃 → 重启后看到什么 → 恢复动作 → 出处。先给一张速查表，细节在后：

| # | 窗口 | 崩溃留下的 durable 残迹 | 谁负责恢复 |
|---|---|---|---|
| ① | 准入事务内 | 无（原子回滚） | 无需 |
| ①' | 准入提交后、drain 前 | session_inbox 行 | wake（sweep 或用户操作触发） |
| ② | claim 事务后、drain 前 | time_suspended ≠ NULL | claim sweep + Synthetic 续跑 |
| ③ | settle 事务之间 | 部分 stale 已判死 | 下次 drain 重扫（幂等） |
| ④ | Step 执行中 | running 工具投影 | settleStaleToolCalls + 续跑 |
| ⑤ | 压缩进行中 | status='running' compaction | settleStaleCompactions |
| ⑥ | 工具 fiber 中 | Job running + 孤儿子 claim | releaseChildClaims + recover* |
| ⑦ | 终态事务后 | 无（已 release） | 无需 |

### 注入点①：准入事务提交前/后（inbox 行与事件的原子性）

- **崩溃在 T1 内**：整体回滚。inbox 行、事件、`enqueued_seq` 推进都没有——用户输入凭空消失，但**库内无任何残迹**（原子性）。用户重发即可。
- **崩溃在 T1 后、drain 前（①'）**：`session_inbox` 行是 durable 事实，静静躺着。重启后若无人 wake，行留待用户下次操作；托管服务器的恢复 sweep 发布 Synthetic 通知时本身会触发 wake（[5_inbox §6](../6_modules/5_inbox-coordination.md) §6.1 步骤 8）。inbox 行无 TTL、无清理任务——唯一批量删除的非投递路径是 revert commit 的边界删除（`projector.ts:768-786`：`enqueued_seq >= boundary` 全删，语义正确的"撤销准入"）。
- **一个反直觉点**：① 崩溃丢的是**用户输入**而非系统状态。系统对"输入丢失"没有防线（为什么要防？发送方——TUI/客户端——能感知连接断开并提示重发）；系统防的是"半条输入"。原子性把损坏降级为丢失，这是所有注入点的共同模式：**宁可丢，不可坏**。
- **机制出处**：admit 的幂等性 = `session_inbox` 插入 `onConflictDoNothing`（`inbox.ts:299-317` 区域）+ 冲突回读兜底（`inbox.ts:191-197`）；投递投影 `projectDelivered` 删行与消息表插入同事务（`inbox.ts:324-336`）——**inbox 行与事件的原子性完全依赖事件事务，没有两阶段**。深读：[5_inbox-coordination §2-§3](../6_modules/5_inbox-coordination.md)。

### 注入点②：claim 写入与 drain 开始之间

- claim 在 **Execution.Started 事件的同一事务**里由 commit 钩子写入（`execution.ts:76-85`，`claimOnCommit` 挂 `store.claim(sessionID)`；started 钩子 `execution.ts:112-115`）。这是"write-ahead"的字面含义：**durable 的"我要开始忙了"先于任何实际工作落盘**。
- **崩溃在 T2 后、drain 前**：重启后 `listSuspended`（`store.ts:194`，`time_suspended NOT NULL AND parent_id IS NULL`）扫到该会话——sweep 会恢复一个"其实还没开始跑"的 turn：发布 `Synthetic(CONTINUE_AFTER_SERVER_RESTART)` 让模型续跑。代价是模型可能凭空收到一条"继续"指令；收益是协议**不需要区分"claim 了但没跑"与"跑到一半"**——同一签名，同一路径。
- **claim 的 NULL 守卫**：`store.claim`（`store.ts:205-215`）`WHERE time_suspended IS NULL`，重复 claim 零行 no-op——恢复的 turn 经同一 started 钩子再 claim 天然幂等。且 `time_updated` 自赋值：claim **不算用户活动**，不会把会话顶到列表前面。

### 注入点③：drain 入口的 stale 结算（衔接点）

drain 的**第一件事**、先于任何模型工作：`llm.ts:71-72` 依序 `settleStaleCompactions` → `settleStaleToolCalls`。这两个函数是注入点⑤⑥的恢复动作，也是"本进程内上一次异常"的兜底——不假设只有重启才需要结算。崩溃发生在结算事务之间：部分 stale 已判死、部分仍是 running——**下一次 drain 再扫一遍即可**（结算本身是发布 Failed 事件，幂等靠投影 `onConflictDoNothing`）。

### 注入点④：Step 执行中（本篇最微妙的注入点）

窗口内容：LLM 流式返回进行中，或**工具副作用已发生但 Tool.Completed 未落盘**。

- 崩在此处，重启后看到：Step.Started 已在库里（含 assistant 占位），工具在投影历史里 `status: "streaming" | "running"`，但永远等不到结果。
- 恢复动作分两半：
  - **投影侧**：`settleStaleToolCalls`（`llm.ts:362-385`）扫出所有 streaming/running 工具，发布 `Tool.Failed(error: {type:"aborted", message:"Tool execution interrupted: ..."})`，保留原 assistant 归属（`assistantMessageID`），subagent 工具附 child sessionID。**不重放副作用**——spec 明言 "never directly replays ambiguous side effects"。
  - **执行侧**：claim sweep 恢复 turn，Synthetic 指令含 "Continue from where you left off without repeating completed work"（restart.ts:15-16）——把"哪些副作用算已完成"的判断交给模型读历史。
- **"durable 先于副作用"如何约束这里**：见 §4.2——真相源记录的是工具**结果**而非副作用本身，约束方向是反的（副作用可以发生在落盘前），所以此窗口的残余风险是**重复执行**而非状态损坏。

### 注入点⑤：压缩进行中

- 崩在此处，重启后看到：`session_message` 里有 `type='compaction', status='running'` 的行——进程死亡跳过了 compaction finalizer。
- 恢复动作：`settleStaleCompactions`（`llm.ts:332-360`）扫出所有 running 的 compaction 消息，逐个发布 `Compaction.Failed(compaction.interrupted)`，**从新到旧**（`orderBy desc`）结算以匹配事件投影顺序；注释明言包含"藏在已完成 checkpoint 后面的孤儿"。测试实证：`session-runner.test.ts:4432-4435` 的断言注释——"These starts have no terminal events, as after SIGKILL. The older orphan is outside model-visible history; recovery must settle it as well."
- 深读（含 checkpoint 语义）：[4_session §10.2](../6_modules/4_session.md)、[7_compaction](../6_modules/7_compaction.md)。

### 注入点⑥：工具 fiber 运行中（后台 shell / subagent）

- 崩在此处，重启后看到：Job 记录 `status='running'`（`jobs.pendingBackground`），subagent 的**子会话可能仍有自己的 claim**。
- 恢复动作（sweep 的前半段，restart.ts:193-231；序出处在 [5_inbox §6.1](../6_modules/5_inbox-coordination.md)）：
  1. `releaseChildClaims(children)` 先清孤儿子 claim（可恢复的 subagent 除外，`store.ts:225`）；
  2. 后台 job 逐个恢复，**shell 排在 subagent 前**（restart.ts:207-208 注释："Admit shell outcomes before a recovered child can start its first model request"——shell 结果是环境事实，必须先于任何模型调用可见）；
  3. `recoverShell`：running → cancelled（"Command cancelled because the server restarted"），通知走 `synthetic(resume:false)` **不得唤醒**空闲的 long-lived shell 拥有者；
  4. `recoverSubagent`：验父子关系（失联直接丢弃）、非 running 补发结果、child 需恢复则 `jobs.start` 包装 `execution.resume(child)`，完成后取最后一条无错 assistant 消息作结论交 `SubagentCompletion.deliver`。
- 深读：[5_inbox §6.3](../6_modules/5_inbox-coordination.md)。

### 注入点⑦：终态事务之后

`settled` 钩子（`execution.ts:119-144`）按终态分派：Succeeded/Failed 带 `releaseOnCommit`；用户/inactivity 中断释放；**shutdown 中断特意不挂 release**（`execution.ts:134`，`outcome.reason === "shutdown" ? undefined : releaseOnCommit`）——优雅关停与 kill -9 留下同样的 claim 签名，下次启动都续跑。Tn 提交后崩溃：claim 已清、计数已零，安全。

### 三种死亡的统一签名（本小节的设计要点）

把上表收拢成一句话：**crash、SIGKILL、isolate eviction、优雅 shutdown——四种死亡方式在数据库里留下完全相同的签名**（restart.ts:47-49 头注释："crash, SIGKILL, isolate eviction, and graceful restart all leave the same durable signature"）。这不是巧合而是约束：

```
死亡方式            teardown 跑了吗   claim 终态        数据库签名
─────────────────────────────────────────────────────────────
正常完成            是（settled 钩子） released          time_suspended = NULL
用户中断            是                released          同上
shutdown 中断       是                preserved（刻意）  time_suspended ≠ NULL
kill -9 / 断电/OOM  否                preserved（没跑）  time_suspended ≠ NULL
```

注意第三、四行的签名**相同**——设计者选择让优雅关停也走恢复路径（而不是在关停时抢着把 turn 跑完或标记成失败），因为"shutdown 钩子可能根本没机会跑"（`execution.ts:79-80` 注释："recovery is a property of the database, never of a shutdown hook that may not run"）。恢复协议因此只需要识别**一种**孤儿状态，而不是为每种死亡方式写一条分支。副作用是：一次例行 `opencode serve` 重启会把所有执行中的 turn 走一遍 Synthetic 续跑——这是托管服务器滚动升级能工作的前提。

---

## 3. 恢复 sweep 全景与防 crash-loop

sweep 入口 `resumeSuspendedSessions`（restart.ts:193-231），九步顺序的"为什么是这个序"在 [5_inbox §6.1](../6_modules/5_inbox-coordination.md)，此处只补崩溃安全视角的三条。

**sweep 自己也可能死**。整个函数没有任何"先标记、后执行"的协议状态——每一步都是可重入的：`listSuspended` 是纯读；`releaseChildClaims` 幂等（再跑一遍还是清同样的行）；后台恢复经 `jobs.get` 查重（restart.ts:213，已非 running 则跳过）；正式恢复前的 `execution.active` 重读（restart.ts:220，注释："Background completion can wake a parent, so inspect local ownership only after recovery"）兜住"后台恢复唤醒了父会话、父已在本进程活跃"的竞态。sweep 在任何一步崩掉，下一次启动从头再跑——正确性不依赖完整执行，依赖每步幂等。

**恢复预算 durable 先行**（restart.ts:77-99）：`prepareResume` 先 `countResume`（`store.ts:239`，`resume_attempts+1 RETURNING`，读改写一条语句原子完成）**再**决定是否恢复。注释（restart.ts:78-79）："Durable before the resume runs, so a crash inside the resumed turn is counted by the next sweep and **the budget cannot be dodged**"。超限（默认 10，restart.ts:33 `DEFAULT_MAX_ATTEMPTS`）则发布 `Execution.Failed(RESUME_EXHAUSTED)` 并在 commit 钩子里 `release`（清 claim + 清零计数同一条 UPDATE，`store.ts:217-224`——不存在"放了 claim 但计数残留"的中间态）。预算是 **per-turn**：只有 terminal 事件清零，正常完成的 turn 永不累积（restart.ts:25-30 接口注释）——这是恐惧 4（crash-loop）的完整防线：一个反复在恢复路径上死掉的 turn 最多被自动拉起 10 次，第 11 次被 `RESUME_EXHAUSTED`（type: "aborted"）终结。

**claim 从不在 sweep 里清**（restart.ts:42-46 Interface 注释："The claim is never cleared here: only a terminal event releases it, so a death anywhere in the resume path leaves the same orphaned claim for the next boot"）。恢复路径上任何一点再死，同一个孤儿 claim 原样留给下一次启动——预算计数保证这不会无限循环。

---

## 4. write-ahead claim 语义深读：为什么诚实地不是 exactly-once

`AGENTS.md:184` 原话：

> "A write-ahead execution claim marks a process-local busy period for restart recovery… **The claim is a recovery marker, not clustered ownership, fencing, or an exactly-once guarantee.**"

这句话的自我限定分三层，每层都对应一个真实的重复执行窗口：

### 4.1 claim 标记的是什么

"process-local busy period"——一个进程内、按会话键控的执行期标记。它写给**确认前任已死的下一次启动**看（托管协议三保证，restart.ts:55-61 头注释：successor 只在前任 PID 确认死亡后拉起——client service `kill`/`evict` 轮询 PID；注册锁串行化；未注册进程永不 sweep）。它不是写给"另一个活着的进程"看的：`store.claim` 的 NULL 守卫不拒绝他人，因为它根本不是锁。

值得对比 write-ahead 这个词的两处用法，避免混淆：存储层的 WAL 是"日志先于数据页"，执行层的 claim 是"意图先于工作"——同构不等价。WAL 保证已提交事务不丢；claim 保证**未完成的工作可发现**。前者是耐久性，后者是可恢复性，中间隔着注入点④那扇关不上的窗（副作用不进任何日志）。AGENTS.md 用 "write-ahead execution claim" 而不是 "execution lock/lease"，措辞本身就是边界声明：它只承诺"写了再说"，不承诺"写了就只有我能做"。

### 4.2 重复执行的窗口在哪

"durable 先于副作用"的精确含义是：**事实记录先于副作用之后的下一步**，而非副作用本身被去重。三个不可消除的窗口：

1. **注入点④（工具侧）**：bash 命令已跑、文件已写、付费 API 已调，但 Tool.Completed 未落盘。重启后 settleStaleToolCalls 把它判为 aborted——副作用**已经发生**，模型侧看到的却是"没做成"。模型若按恢复指令重试，副作用执行两次。
2. **注入点④（provider 侧）**：LLM 请求已发出、token 已烧，响应流中断在半途。没有任何机制能把"这一次请求"要回来。
3. **Synthetic 续跑与用户输入并发**：恢复的 turn 与用户新输入各自 drain，交错产生事件序列——正确性靠事件 seq 全序，但**业务层面**模型可能重复汇报已完成的工作。

### 4.3 靠什么兜底

- **基础设施侧**：投递/投影全链路幂等（§5），保证重放**不损坏状态**——重放安全的下界。settleStale* 只发布 Failed 事件、从不重跑工具（llm.ts:383-384 的 `executed: tool.executed === true` 只是元数据透传，不是重放指令），把"哪些副作用算已完成、要不要重做"完整留给模型读历史。
- **模型行为侧**：恢复指令把幂等责任上交——"Continue from where you left off **without repeating completed work**"（restart.ts:14-15）。restart.ts:52-53 明言："local coordination prevents concurrent drains, **not repeated external side effects** after a crash"。这是诚实的设计：在无围栏的单机协议里假装 exactly-once 比承认 at-least-once 更危险——假保证会诱导上游省略自己的去重逻辑。
- **副作用自身的幂等**：一部分工具天然幂等（读文件、grep）；不幂等的（写文件、跑 bash）没有额外协议——用户视角的最终防线是事件历史里 aborted 工具的**可见性**：模型和用户都能看到"这个工具被中断了"，重复执行是知情决策而非静默损坏。
- **未来**：clustering 引入显式 placement + fencing 协议前的过渡态（[5_inbox §7](../6_modules/5_inbox-coordination.md)）。

---

## 5. 投影幂等三层防线（收口）

重放/恢复不损坏状态的保证，全部在 [1_data-layer §9](../6_modules/1_data-layer.md)，此处一行索引：

```
(a) 投影器 onConflictDoNothing ───── session/projector.ts、inbox.ts、instruction-state.ts 等全域
(b) Bus.replay 全等校验 ──────────── 已覆盖 seq 逐字段相同则跳过、相异 die "Replay diverged"
(c) event id 全库查重 ────────────── persist 开启时的写入侧兜底
```

三层在崩溃安全里的分工：(a) 让**正常写入路径**对重放免疫——恢复 sweep 发布的事件即使与残留投影撞键也安全；(b) 让**显式重放**（`Bus.replay`）可校验——分歧即 die 而非静默覆盖；(c) 让事件行本身不重复。本篇新增的唯一观察：settleStale* 系列发布的 Failed 事件也走这条链——结算本身可以安全地重复执行（崩溃在结算事务之间再扫一遍，见注入点③），这是恢复路径自举的可靠性基础：**恢复机制不需要为自己的崩溃再做一层恢复**。

---

## 6. 失效场景矩阵

| 注入点 | 丢失物 | 重启后看到 | 恢复动作 | 残余风险 |
|---|---|---|---|---|
| ① T1 内 | 用户输入（原子消失） | 无残迹 | 无需——用户重发 | 输入丢失（用户可感知） |
| ①' T1 后 drain 前 | 无（inbox 行已 durable） | session_inbox 残留行 | sweep 的 Synthetic 触发 wake；或下次用户操作 | 无 |
| ② T2 后 drain 前 | 无（claim 已 durable） | time_suspended 非空 | prepareResume + Synthetic 续跑 | 模型收到空泛的"继续"指令 |
| ③ settle 之间 | 无 | 部分 stale 已判死 | 下次 drain 重新扫（结算幂等） | 无 |
| ④ Step 中 | 内存中的流式内容/未落盘的工具结果 | running 工具投影 | settleStaleToolCalls 判 aborted + claim 续跑 | **副作用重复执行**（§4.2）；已烧 token 不可回收 |
| ⑤ 压缩中 | 内存中的摘要进度 | status='running' 的 compaction | settleStaleCompactions 判 interrupted（新→旧） | 已烧的摘要 token；需重新压缩 |
| ⑥ 工具 fiber | 运行中的 shell/subagent 进程 | Job status='running' + 孤儿子 claim | releaseChildClaims → shell 先于 subagent 恢复/补通知 | shell 进程随父死亡（cancelled）；subagent 结论取"最后无错 assistant"可能偏保守 |
| ⑦ 终态后 | 无 | 干净 | 无 | 无 |
| 恢复路径上再崩 | 恢复尝试本身 | attempts 已 +1（durable 先行） | 下一次 sweep 计入计数 | 10 次后 RESUME_EXHAUSTED 终结 |
| sweep 中后台恢复半程 | job 恢复状态 | Job 仍 running / 已补通知 | `jobs.get` 查重跳过已完成的（restart.ts:213） | 通知恰一次由 Synthetic 准入的第一次获胜保证 |

读法提示：矩阵里唯一的**不可接受风险列**是④的"副作用重复执行"——但它不是遗漏而是显式取舍（§4）；其余全部被三层协议收敛到"无"或"可感知的丢失"。整张矩阵没有一格是"状态损坏"——那是 §1 事务底座在事务内消灭的东西，本篇的注入点全部活在事务**之间**。

---

## 7. 测试覆盖（实测）

恢复语义的测试集中在两个文件，均为**模拟"无 teardown 死亡"**而非真 kill 进程——手工构造"事件已发布、终态缺失"的库状态（如 session-runner.test.ts:4432 注释明言 "These starts have no terminal events, as after SIGKILL"）。这种测法的含义：恢复协议的被测对象是**数据库签名**，不是进程行为——与 §2 末"统一签名"的设计互为印证。

**`test/session-execution.test.ts`**（1394 行，claim/sweep 主体，`it.effect` 标题即行为规格）：

```
claim 生命周期
  :55  the sweep only lists claimed top-level Sessions
  :75  claims at execution start, releases on completion, and preserves through teardown
  :122 a user interrupt releases the claim so the turn never resurrects
  :145 reports an idle interrupt as a no-op
sweep 语义
  :219 starts every claimed execution without waiting for earlier drains to finish
  :244 resumes each claimed Session at most once
  :348 the sweep leaves Sessions already draining in this process untouched
预算
  :291 terminalizes a turn that exhausts its resume budget instead of crash-looping
       （:336 断言注释："a crash anywhere in the resume path leaves both intact"）
  :318 counts every resume durably and never consumes the claim it recovers
       （恢复后再查 attempts 仍为 1——claim 不被预算消耗）
  :650 does not bypass a claimed owner's restart budget for a shell notice
shell / subagent 恢复
  :379 keeps shell owners idle until a user prompt delivers recovered notices exactly once
  :488 wakes the owner for a silent shell failure persisted before its completion notification
  :467 preserves locally running background work
  :789 resumes a background subagent and notifies its parent exactly once
  :869 delivers a subagent result persisted before restart without rerunning the child
 1040 terminalizes a recovered subagent that exhausts its resume budget
interrupt-continue（恢复的近亲：中断后选择性续跑）
 1095/:1128/:1157/:1179/:1212  steer/queued/compaction/控制项五种续跑选择
```

**`test/session-runner.test.ts`**（stale 结算，scenario 命名即文档）：

- `:4479` "durably fails local tools left running by a prior process before continuing"；`:4527` subagent 版；`:4592`/`:4640` hosted tool / pending tool input 同款——覆盖注入点④的全部工具形态。
- `:4432-4435`（SIGKILL 模拟 + 孤儿 checkpoint 也被结算）与 `:4822` "durably settles local tool failures before continuing"——注入点⑤。
- `:5139`/`:5204` interrupt 期间的阻塞工具结算——§2 注入点③的"本进程内异常也走结算"兜底。

另：`session-projector.test.ts:712` "does not revive a stale incomplete assistant projection" 守住投影侧。`grep -rn "resumeSuspendedSessions" test/` 命中仅 session-execution.test.ts——sweep 的测试面集中单文件，覆盖密度高；真正的多进程 kill 场景（托管服务器 successor 拉起）在 desktop/server 侧测试，超出本篇范围（指路 [8_testing-infra](8_testing-infra.md)）。

---

## 8. 相关篇

- [../6_modules/1_data-layer.md](../6_modules/1_data-layer.md) — §4 事件总线事务序、§5 事件溯源、§9 失败与恢复、§9.5 内存磁盘分界（本篇 §1/§5 的展开）
- [../6_modules/4_session.md](../6_modules/4_session.md) — §10 失败与恢复总表、§5.2 claim 生命周期（本篇 §2 注入点②⑦的机制面）
- [../6_modules/5_inbox-coordination.md](../6_modules/5_inbox-coordination.md) — §6 恢复子系统九步、§7 跨进程视角（本篇 §3 的顺序论证与协议排除）
- [../6_modules/7_compaction.md](../6_modules/7_compaction.md) — 压缩算法本体（注入点⑤只写崩溃面）
- [../6_modules/17_job.md](../6_modules/17_job.md) — 后台 shell 生命周期（注入点⑥ recoverShell 的功能面）
- [8_testing-infra.md](8_testing-infra.md) — 测试基建总览（本篇 §7 只收恢复相关切片）
- [5_resource-limits.md](5_resource-limits.md) — 恢复预算（10 次）与资源治理的限额表同属"有界化"家族

## 演进注记

本篇为**新增域**：崩溃安全机制此前散在 1_data-layer（§9）、4_session（§10）、5_inbox-coordination（§6-§7）三篇，本篇首次以"注入点枚举"为主线收口，并引入三篇未覆盖的整合视角——统一签名表（§2 末）、WAL 与 claim 两处 "write-ahead" 的辨析（§4.1）、恢复路径自举可靠性（§5）、失效矩阵的"宁可丢不可坏"读法（§6）。对照旧卷 4.19 基线（2026-08-03）：write-ahead claim、restart.ts、settleStale* 全套在当时均不存在（4_session 演进注记第 4/5 条印证），即崩溃安全协议是 v2 周期随托管服务器生命周期管理一起长出的新层。
