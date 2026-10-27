# 准入与协调详解（inbox 投递状态机 × run-coordinator 门铃执行 —— Session 内核两大语义支柱的下钻）

> 一句话：**inbox 是 durable 待办表——用 steer/queue 两种投递词汇加上 compaction/move 控制项精确编码"输入何时进入模型视野"；coordinator 是进程内执行的所有权管理器——用门铃合并把任意次唤醒折叠成一个 busy period，并用 write-ahead claim 把"这个进程正在跑"写成崩溃后可恢复的 durable 事实。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19，`sed -n` 逐段抽查）。
> 权威出处：`specs/v2/session.md`（"Prompt Admission Precedes Execution" / "Execution Is Process-Local" / "Recovery Boundaries Stay Explicit" 三节）+ 仓库根 `AGENTS.md` "Session Core" 条目。
> [返回 README](README.md) | 前置 [4_session.md](4_session.md) §4（准入概览）/ §5（协调概览）——本篇是它们的下钻：**方法行号表与门铃结构概览不再重复，只引用后展开完整语义**。

---

## 0. 模块地图

本篇覆盖 4 个源文件（+2 个关键联动文件），共约 1,400 行：

```
packages/core/src/session/
├─ inbox.ts ★ 投递状态机（550 行）
│   ├─ L29-59     类型与编解码器：re-export schema 四 Item、Promotable、KeyedMutex
│   ├─ L62-74     serialized() 会话级串行锁 + LifecycleConflict + matches()
│   ├─ L76-143    fromRow / find / promotedFromMessage（已投递项的幂等反解）
│   ├─ L149-271   Service 闭包：reconcile / admit / admitCompaction / cancel / steer / queue
│   ├─ L277-405   事务投影器：projectAdmitted / projectDelivered / projectCancelled /
│   │             projectDelivery(Changed) / list / moveIDs
│   └─ L407-550   提队算法：nextPromotable / has / publish / promote / pendingSteers
├─ run-coordinator.ts ★ 泛型执行协调器（196 行，非 Session 专用）
│   ├─ L7-30      Coordinator 接口（active/isActive/run/wake/interrupt/awaitIdle）
│   ├─ L40-47     Execution 结构（done/scope/pendingWake/stopping/interruptionReason）
│   ├─ L74-120    loop / start / settle —— 门铃驱动的执行循环
│   └─ L122-195   isActive / run / wake / interrupt / awaitIdle 实现
├─ execution.ts ★ 进程全局执行服务（195 行）
│   ├─ L17-54     Interface + terminal() 三分类（succeeded/interrupted/failed）
│   ├─ L76-110    write-ahead claim 钩子 + drain（放置发现 + DrainResult 三态分派）
│   ├─ L111-148   coordinator 装配：started/settled 钩子里的事件发布与 claim 生命周期
│   └─ L150-174   公开面：interrupt 的 resume 选项（steer-scope 重按门铃）
└─ execution/ 子目录（仅 1 文件）
    └─ restart.ts ★ 崩溃恢复（240 行）
        ├─ L15-44   常量与 Interface（resumeSuspendedSessions）
        ├─ L46-63   头注释：at-least-once / sweep 假设 owner 已死的协议保证
        ├─ L77-99   prepareResume：durable 计数 + 预算终结 + restart 续写指令
        ├─ L101-190 recoverShell / recoverSubagent
        └─ L192-232 resumeSuspendedSessions 主扫

联动文件（本篇引用，不展开）：
├─ runner/llm.ts L57-199   drain 主循环——安全步边界的调用方（时序细节归
│                          ../5_timeline/5_prompt-chain.md，本篇只取边界语义）
├─ store.ts L194-250       claim/release/releaseChildClaims/countResume/listSuspended
├─ projector.ts L618-682   四个 inbox 事件的投影落点
└─ schema/src/session-inbox.ts（78 行）Delivery 字面量 + Item tagged union
```

## 0.5 通俗完整描述（先读这段再看细节）

用户发来的每条消息（或 hook、通知）不是立刻喂给模型，而是先成为 inbox 表里的一行"待办"。**本篇讲的就是这份待办的两个管理者**：

**第一个管理者：inbox（投递状态机）**。它是一张数据库表，每行待办带着投递方式：`steer`（插话——"现在正在跑的这轮请捎上我"，赶上批的提升会一起给模型）或 `queue`（排队——"等模型闲下来再轮到我"，空闲边界一次只放行一个）。还有两种特殊的"控制项"（手动压缩、会话迁移）——它们不是消息，而是"改写历史"的指令，是投递的**硬边界**（steer 批提升到这里必须停刀，边界后的话属于改写后的历史）。待办可以被取消、可以从 queue 改 steer（编辑体验的基础）。

**第二个管理者：coordinator（门铃执行器）**。输入入库后按一下门铃（wake）——如果这个会话已经有执行在跑，只留一个"再跑一轮"的标记（十次按铃折叠成一次）；没有就雇一个临时工（fork 该会话专属 fiber），工人在"排空循环"里反复取活：提升待办→调模型→跑工具→直到没活干；干完且门铃没再响，工人解散（下次 prompt 再雇新的）。开工前工人会在数据库写一张"我正在跑"的条子（write-ahead claim）——进程若崩了，下次启动扫到残留条子就知道哪些活干到一半，接着续跑。

两个容易理解偏差的关键点：

1. **没有常驻消费者**——不存在一个后台线程持续扫表；一切由 wake 点火：fork→干→散。同会话串行（防撕裂数据）、跨会话并行是 coordinator 的两条铁律。
2. **"为什么叫协调"**——它不执行任何业务，只管"谁在跑、跑几份、崩了怎么认领"的所有权；真正干活的能力（runner）在会话所在目录的图里，工人按需借用（fiber 是手、runner 是工具箱）。

一句话：**inbox 管"活什么时候进窗口"（steer/queue/控制项），coordinator 管"谁在干这个活"（门铃合并、临时工、claim 遗嘱）——两张账合起来，就是"输入永不丢、执行不打架、崩溃可续跑"**。

---

## 1. 心智模型

两个组件回答两个正交的问题：

```
        "什么输入该进模型视野？"                "谁、在此进程、正在跑？"
        ┌────────────────────────┐            ┌──────────────────────────┐
        │ session_inbox 表 (durable)│          │ Coordinator Map (进程内)   │
        │                        │            │                          │
        │ user/synthetic 提示    │            │ Key → Execution          │
        │ compaction/move 控制   │            │   ├─ owner fiber（排水） │
        │ delivery: steer|queue │            │   ├─ pendingWake（门铃） │
        │ enqueued_seq 全序     │            │   └─ stopping 标志       │
        └───────────┬────────────┘            └────────────┬─────────────┘
                    │ promote（安全步边界）                  │ drain（Location 内）
                    ▼                                       ▼
            session_message 可见行                 runner.drain → llm.stream
                    │                                       │
                    └────────── 事件溯源同一 Bus  ──────────┘
                              （Bus + seq + 投影事务）
```

- **inbox 是 durable 待办**：它的行只在"准入但未投递"这个区间存在（projectDelivered 的删除就是"pending-only"的由来，inbox.ts L319-323 注释）。它不属于任何执行——执行死了行还在，执行来了行按词汇规则被消费。
- **coordinator 是进程内所有权**：`Map<Key, Execution>` 是纯内存结构，进程重启即清零。它不知道 inbox、不知道 LLM；它只保证"每个 key 至多一个 fiber 在 drain，期间到来的唤醒被合并成一次后续 drain"。
- 二者的接合点在 runner：`llm.ts` 的边界循环在 `SessionInbox.serialized` 锁内调 `nextPromotable`/`promote`，drain 返回后 coordinator 决定是否再转一圈。
- **claim 是第三种东西**——不是锁也不是队列，而是写在 `session` 表 `time_suspended` 列上的**恢复标记**（store.ts L205-216），跨进程存活但语义上只对"确认前任已死"的启动恢复有意义（§7）。

## 2. 投递词汇全语义

对照 `specs/v2/session.md` L16-23（"Delivery is explicit"节）逐条落地。词汇表只有两个普通词 + 两个控制词：

```
Delivery = "steer" | "queue"          （schema/session-inbox.ts L11）
Item     = user | synthetic | compaction | move   （L37-47 tagged union，四者都带 delivery）
```

### 2.1 steer —— 安全步边界插入

spec L18：*"steer is the default. Steers deliver in enqueue order at the next Safe Step Boundary. Delivery stops before a compaction or move control item."*

三条规则在 `promote`（inbox.ts L496-531）一次读全：

1. **默认值**：`input.delivery ?? "steer"`（session.ts L155、L257、L296 三处统一）。
2. **按准入序投递**：`pendingSteers` 以 `enqueued_seq` 升序取全部 steer 行（L533-539），promote 发布 `steers.slice(0, control)`——到第一个控制项为止，一批全发（L509）。
3. **控制项是硬边界**：`findIndex(row => row.type === "compaction" || row.type === "move")`（L507）；`control === 0`（控制项就在队首）时 promote 返回 `undefined`——不投递、把边界让给 runner，由后者在锁内原子发布 `InboxDelivered + Compaction.Started`（llm.ts L91-95）或 `InboxDelivered + Moved`（L96-105）。

**steered compaction 的优先规则**（pendingSteers L542-549 的排序修正）：

```
准入序: [U1(steer), U2(steer), C(compaction,steer), U3(steer)]
修正后: [C, U1, U2, U3]      ← L547: rows.unshift(...rows.splice(control, 1))
```

源码注释（L543-545）给出理由：move 改变 context 的 Location，compaction 绝不跨 move 提队；但在 move 边界**之内**，先把 compaction 提到队首，能让 U1/U2 的原文在检查点之后逐字保留（verbatim），而不是被卷进检查点的摘要里。这是"摘要完整性质"对"入队顺序"的一次显式让步——spec 的 "steered compaction takes priority" 就是这一行 splice。

### 2.2 queue —— 空闲边界恰好一个

spec L19：*"queue remains pending while the Session can continue. At an idle boundary, steers still take priority; otherwise one queued item delivers, followed by any steers that arrived during delivery. The runner then reevaluates continuation before another queued item."*

promote 的 input-scope 分支（L512-528）逐句对应：

```
steers 为空（空闲边界无插队）:
  queued = 取 enqueued_seq 最小的一个 queue 行 (L512-519, LIMIT 1)
  ├─ 无 → return 0                     ← 会话真正空闲
  ├─ 是控制项 → return undefined        ← 同样让给 runner 处理
  └─ 投递这一个 (L522)
       ↓
  arrivedSteers = 重新拉 steer 队列      ← 投递期间新到的 steer 补投 (L523-527)
  （同样止于控制项）
```

"runner 重新评估"落在两处：入口资格检查里 steer-scope drain 遇 queued 普通提示直接 `Complete`（llm.ts L68-69）；以及第二个 queued 项**永远**要等下一轮 drain——promote 每次 input-scope 至多消费一个 queued 行，这是把"下一 turn 的输入"和"本 turn 的插话"在数据层就分开。

drain 入口的资格检查完整语义（llm.ts L64-69，非 force 且非续跑时）：

```
pending = nextPromotable(db, sessionID, "input")
├─ 无 pending                       → Complete（真正空闲，白来一趟）
├─ pending 是控制项                  → 继续（steer-scope 也要处理控制项）
└─ promotable === "steer"           → Complete
   且 pending.delivery === "queue"
   且非控制项
```

第三条就是"queue remains pending while the Session can continue"的守门员：一个 steer-scope 的 drain（例如 interrupt-resume 按的门铃）发现队首只是排队的下一 turn 提示时，**不开新 turn**，安静离开。queued 项要等一个 input-scope 的 drain（空闲边界）才被消费。

### 2.2.1 空闲边界的完整代码走查

```ts
// inbox.ts L512-528（实测）
const queued = yield* db.select().from(SessionInboxTable)
  .where(and(eq(...session_id), eq(...delivery, "queue")))
  .orderBy(asc(...enqueued_seq)).limit(1).get()          // 恰好一个
if (!queued) return 0
if (queued.type === "compaction" || queued.type === "move") return undefined
const promoted = yield* publish(db, bus, sessionID, [queued])  // 投递它
const arrivedSteers = yield* pendingSteers(db, sessionID)      // 重拉 steer 队列
const control = arrivedSteers.findIndex((row) => row.type === "compaction" || row.type === "move")
return promoted + (yield* publish(db, bus, sessionID,
  control === -1 ? arrivedSteers : arrivedSteers.slice(0, control)))
```

注意第二次 `pendingSteers` 调用发生在锁内（整个 promote 被 `serialized(sessionID)` 包裹，L502-504）——但投递 queued 项的 publish 会释放锁吗？不会：publish 只是逐条发 Bus 事件，投影事务在 Bus 的 project 回调里同步跑，全程持有 inbox 串行锁。所以"arrivedSteers"的窗口不是锁的窗口，而是 **drain 纤程自己**从投递 queued 到重查 steer 之间的并发准入窗口（准入也在同一把锁上排队，投递事务提交后立即能插进来）。

### 2.3 控制项作为投递边界

spec L23：*"Manual compaction and Session movement use the same inbox as control items. Each request has its own inbox identity and delivery mode. A control item forms a delivery boundary so later steers do not cross it."*

**"硬边界"的语境解释（为什么不能跨）**：压缩/迁移是要**改写对话历史本身**的操作（压缩折叠前缀成摘要、迁移换执行环境）。想象边界后的 steer 混进边界前的批次：

```
steers = [steer1, steer2, 压缩控制项, steer3]
                      ↑
promote 批提升：findIndex 找到首个控制项
→ 只提 [steer1, steer2]（slice(0, control)），steer3 不跨

如果不停刀：steer3（压缩之后用户才说的话）
混进压缩前的历史位置——它属于"压缩后语境"，
落进"压缩前版本"的历史 = 歧义
```

规则的效果：压缩前的话进压缩前的最后一轮上下文；压缩自己执行、历史改写；steer3 留在 inbox，等下一轮 promote 进入压缩后的新历史——**每句话都落在它该属于的历史版本里**。

配套的两刀（promote 现场，inbox.ts:507-531）：`control === 0`（控制项就在队首）→ promote 返回 `undefined`，这一轮不提任何消息，把边界让给 runner 先处理控制项；queue 分支放行一个 queue 项前同样检查——是控制项也交给 runner 直接处理（:527）。

其余边界性质：

- 控制项与普通项同表同行（同 `session_inbox` 表、同 `Delivery` 字面量），只是 `type` 不同；投递时"行删除但不成为消息"——projector 对 compaction/move 直接 `return`，不插 transcript（projector.ts L628-629）。
- **admitCompaction 的特化冲突规则**（inbox.ts L204-228，`serialized` 锁内四步）：精确命中同会话 compaction → 返回它（幂等）；同 ID 已是别的东西/已 promoted → `LifecycleConflict`；**已有任一 pending compaction → 返回那个**（一会话至多一个待处理 compaction）；否则 admit。这是 spec L9 "operation-specific conflict behavior" 的 compaction 落点。
- move 控制项被 runner 消费后 drain 返回 `Moved`，execution 层换 Location 带着续跑状态重入（execution.ts L107）——控制项因此天然是"换执行环境"的同步点。

### 2.4 cancel / change —— delivery.changed 的语义

三个 mutation 命令（inbox.ts L230-260）全部走 `publishMutation` → `serialized` 锁 → `Effect.catchDefect` 把投影事务的 LifecycleConflict defect 翻译成失败（L451-456）：

| 命令 | 事件 | 投影效果 | 语义约束 |
|---|---|---|---|
| `cancel` | InboxCancelled（L233） | 删行，但 **WHERE 限定 delivery ∈ (steer, queue)**（projectCancelled L348） | 已投递（行已不存在）或异常状态不可取消；删不到行即 LifecycleConflict |
| `steer` | InboxDeliveryChanged{delivery:"steer"}（L243） | `UPDATE ... SET delivery='steer' WHERE delivery='queue'`（projectDeliveryChanged L377-384，**CAS on 旧值**） | queue→steer 改投递：排队项升级为插话 |
| `queue` | InboxDeliveryChanged{delivery:"queue"}（L254） | 反向 CAS：steer→queue | 插话降级为排队 |

CAS 旧值匹配意味着：并发的两次 change 只有一次生效，另一次 WHERE 落空 → defect → `LifecycleConflict` → session.ts 的 `mutatePending` 翻译成 `InboxConflictError`（session.ts L109-120）。**wake 策略在 session 层不在 inbox 层**（spec L51）："steering wakes after a successful mutation, while queueing and cancellation do not"——session.ts L134-142 只有 `steerInbox` 在 mutation 成功后跟一句 `execution.wake(sessionID)`，cancel/queue 不唤醒。

**queue 期间编辑**：编辑消息走 message-updater 路径而非 inbox mutation；inbox 词汇只管"何时投递"，不管"投递内容可否改"——pending 行的 payload 在投影里是不可变的（只在 projectAdmitted 时写入一次，L294-316），想改内容只能 cancel 后重新提交新 ID。

### 2.5 step allowance 的重置规则

spec L21：*"Promoting new user input resets the selected agent's step allowance. A batch of steers resets it once."*

落点在 llm.ts 边界循环的两处（实测）：

```
llm.ts L109-111: 非 steer 输入（即新 turn 的输入）被提升时 → entering = true; step = 1
llm.ts L181:     promote 返回 promoted > 0（本批投递了输入）→ step = 1
```

关键在"批量"：promote 一次发布**一批** steer（L509 的 slice），或一个 queued + 期间到的 steer（L522-527），`promoted > 0` 只把 step 重置**一次**——一批插话算一次新输入，不按条数消耗 allowance。这也解释了为什么 idle 边界"恰好一个 queued"：每个 queued 项独享一次 step 重置与一个 turn，批的概念只存在于 steer 词汇内。

两处重置的分工（L109-111 实测代码）：

```ts
if (!continuing && pending?.delivery !== "steer") {   // queued 项被提升 → 新 turn
  entering = true
  step = 1
}
```

`entering` 置位还改变**下一圈的提升范围**：`nextPromotable(db, sessionID, entering || !continuing ? "input" : "steer")`（llm.ts L86-89）——刚进入新 turn 时允许 input 范围（可能取到 queued 项），turn 进行中只允许 steer 范围。同样，promote 的 scope 也是 `entering && !continuing ? promotable : "steer"`（llm.ts L169-173）。这两个三元把"入口/边界用宽范围、步间用窄范围"的规则编码在两处调用点上，而不是藏在 inbox 里——inbox 只忠于词汇表，范围选择永远是调用方的责任。

## 3. Inbox 状态机全图

4_session §4.2 给的是方法表；这里画**完整状态机**——状态是行的 DB 存在形态，弧上是事件（事件名即投影事务的原子单位）：

```
                          admit (reconcile miss)
   (不存在) ═══════════════════════════════════▶ ┌─────────────────────┐
              InboxEnqueued                      │ pending             │
              [projectAdmitted:                  │ delivery=steer|queue│
                 INSERT 行 + touch               └─────────┬───────────┘
               session.time_updated]                       │
        ┌────────────────────────────┬─────────────────────┤
        │ steer  (queue→steer)       │ queue (steer→queue) │ cancel
        │ InboxDeliveryChanged       │ InboxDeliveryChanged│ InboxCancelled
        │ [CAS UPDATE 旧值匹配]       │ [CAS UPDATE]        │ [DELETE WHERE
        ▼                            ▼                    │  delivery∈{s,q}]
   pending(steer) ◄──────────────pending(queue)            │
        └──────────────┬──────────────────┬───────────────┘
                       │ promote          │ promote
                       │ InboxDelivered×N │ InboxDelivered(+Compaction.Started
                       ▼                  │  或 +Moved, runner 锁内原子发布)
                  (行删除)                 ▼
                       │        ┌─ user/synthetic: 删行 + INSERT session_message
                       │        └─ compaction/move: 只删行（projector L628 直接过）
                       ▼
                 (delivered：幂等面交给 reconcile)
                       ▲
                       │ 重试同 ID admit
                       │ [reconcile: inbox 表 miss → promotedFromMessage
                       │  从 session_message 反解回 Info，L113-143；
                       │  sessionID/类型不匹配 → LifecycleConflict]
```

要点：

- **每个事件 = 一个投影事务**。事件发布（Bus durable append + seq 分配）与投影效果同事务提交——失败以 defect 形态冒泡，被 publishMutation / admit 的 catchDefect 捕获翻译（inbox.ts L191-197、L451-456）。
- **enqueued_seq 的双重身份**：既是投递顺序（所有 ORDER BY 的键），又是事件聚合序列（projectAdmitted 用 `event.durable.seq` 写入，L280）——准入顺序因此天然等于事件顺序，无第二真相源。
- **唯一批量删 inbox 的非投递路径**是 revert commit：按 `enqueued_seq >= boundary.seq` 删（projector.ts L778-786）——回滚语义要求把边界之后准入的一切都退掉。
- **终态没有"delivered"行**：表只存 pending。delivered 之后的重试靠 reconcile 从消息表反演（"first admission wins"），这是"表 = 未消费工作"（AGENTS.md）的直接推论。

## 4. 提队算法

`nextPromotable`（L407-424）与 `promote`（L496-531）共享 `pendingSteers`（L533-550），但有细微分工：

```
nextPromotable(db, sessionID, promotable):
  steer 队首存在? ──是──▶ 返回它（无论 promotable 是什么 —— steer 恒优先）
  promotable ≠ "input" ──▶ undefined        ← steer-scope 询问到此为止
  否则取最旧的 queue 行（LIMIT 1）──▶ 有则返回，无则 undefined

pendingSteers(db, sessionID):
  SELECT ... WHERE delivery='steer' ORDER BY enqueued_seq
  排序修正：首个控制项若是 compaction 且不在队首 → splice+unshift 提到队首
  （move 永不提：跨 move 的 compaction 会作用于错误的 Location）
```

### 4.1 为什么"提一个 queue 再收新到 steer"

promote input-scope 分支（L512-528）的顺序乍看奇怪：先投一个 queued，**再**重新拉 steer 队列补投。为什么不等 steers 一起？

时序场景还原：

```
T0  runner 到达空闲边界（steer 队列为空）
T1  promote: 投递 Q1（最旧 queued）           ← 此刻 Q1 已是"本 turn 输入"
T2  用户插话 S1(steer) 准入（另一 fiber，serialized 锁在 T1 后已释放）
T3  promote: arrivedSteers=[S1] → 补投 S1
T4  runner: S1 是 steer → 在 Q1 的 turn 内作为插话消费，step 不重置
```

- 若 T3 不补投：S1 要等 runner 完成本 turn、重新走入口资格检查才进视野——"steer 在下一个安全步边界投递"的承诺被白白推迟一轮。
- 若 T1 之前先等 steer：空闲边界本身已经确认 steer 为空，等不到东西。
- 补投的这批 steer **不重置 step**（它们走 steer 路径，llm.ts L109 的 `pending?.delivery !== "steer"` 判断在 queued 项上才触发 entering/step=1；补投批到达时 pending 已消费）——语义上它们是 Q1 这个 turn 的插话，与"批 steer 只重置一次"一致。
- 并发安全：投递 Q1（发布事件+投影事务）与 S1 准入都在 `serialized(sessionID)` 串行域内，T2 只能插在 T1 的事务之后——arrivedSteers 读到的是稳定的增量。

这就是 spec L21-22 "one queued item delivers, followed by any steers that arrived during delivery" 的全部实现——一句规格，四行代码，一个精确的锁窗口。

### 4.2 promote 的三态返回

| 返回 | 含义 | runner 反应（llm.ts） |
|---|---|---|
| `undefined` | 边界被控制项占领 | 锁内已原子发布控制项事件；compaction → 执行后 `continue` 重进边界；move → 返回 `Moved` |
| `0` | 无 pending | 资格检查失败 → `Complete`（L164-165） |
| `>0` | 投递了 N 项 | `step = 1`（L181）+ 标题生成 + 进入 Step |

## 5. RunCoordinator 状态机

run-coordinator.ts 是**泛型**的（`Key` 不限于 Session ID），Session 语义全部由 execution.ts 装配进去——协调器本身可单测、可复用。

### 5.1 Execution 结构（L40-47）

```ts
type Execution = {
  done: Deferred<void, E>        // joiner 的票据：resolve 时携带本执行的 exit
  owner?: Fiber<void>            // 排水 fiber；undefined = 结算窗口
  scope: Promotable              // 本圈 drain 的提升范围（"input"|"steer"）
  pendingWake?: Promotable       // 门铃：drain 中途到来的唤醒（合并取最宽 scope）
  stopping: boolean              // interrupt 已接受，清理中
  interruptionReason?: Reason    // terminal 分类用（user/shutdown/inactivity）
}
```

### 5.2 状态机全图

```
                          wake(key)/run(key)
     ┌─────┐ ───────────────────────────────────▶ ┌──────────────────────────┐
     │idle │                                    │ │ execution: owner fiber   │
     │(无键)│ ◀─────────────────────────────────│ │  started→drain ⟲         │
     └─────┘   settle: 门铃静默                   │  (drain 完→loop 查门铃:    │
       ▲       [DELETE key + done(exit)]         │   pendingWake? → 吸收为    │
       │                                       │ │   scope, 蹦床再 drain)    │
       │  settle: 门铃又响                        │ └────┬─────────────────────┘
       │  [start(key,false,pendingWake)          │      │ exit（含被中断）
       │   = 后继执行；旧 done 仍 resolve 旧 exit] │      ▼ settle 钩子
       │                                       │ ┌──────────────────────────┐
       │   后继执行 ◀───────────────────────────┼─│ 结算窗口: owner=undefined │
       │                                       │ │  settled 钩子运行中       │
       │                                       │ └──────────────────────────┘
       │                                       │      ▲ interrupt 落在这里:
       │                                       │      返回 false 但**没收门铃**
       │  interrupt(key) ───────────────────────▶   stopping=true; pendingWake=undefined;
           （仅活跃执行）                            interruptionReason=reason;
                                                     fire-and-forget Fiber.interrupt(owner)
```

四方法精确语义（对照接口文档注释 L9-29）：

- **`run`（L124-134）**：显式 resume。键活跃且非 stopping → `Deferred.await(done)` **加入当前执行**（同会话 resume 合并）；stopping → 等旧 done 后**重跑**（拒绝 joiner 加入垂死执行）；空闲 → `start(key, force=true, "input")`。
- **`wake`（L136-145）**：按门铃。活跃 → 只置 `pendingWake`，合并规则 `pendingWake === "input" ? "input" : scope`（L141，input ⊃ steer 取最宽）；空闲 → `start(key, false, scope)`。永不阻塞、永不失败。
- **`interrupt`（L147-167）**：三窗口。idle/stopping → false（no-op）；**结算窗口**（owner 已退出、settled 未完）→ false 但清 pendingWake（终态已定，防止 settle 起一个"死意图"后继）；活跃 → 没收已记录的 wake（"wakes recorded so far belong to the interrupted intent"，L159-160）、fire-and-forget 中断（"callers like the interrupt endpoint must acknowledge immediately even when finalizers are slow"，L163-164）并立即返回 true。清理期间**新到**的 wake 是新准入，settle 时正常开后继。`awaitSettlement` 选项只等这一次执行的 done——不等清理期间新进的活。
- **`awaitIdle`（L171-176）**：等 done 后**重查**（尾递归）——覆盖门铃后继执行，这是它与 `awaitSettlement` 的本质区别（接口注释 L20-21）。

### 5.3 蹦床与 leading yield

`loop`（L74-85）drain 结束后若有 pendingWake，先 `Effect.yieldNow` 再递归调用自己——注释（L81）明说是防"同步完成的 drain 跨后继 start 增长栈"。`start`（L87-112）fork 时同样先 yield（L94-96 注释）：让 `owner` 在 drain 可能同步 settle 之前完成赋值，也让"自唤醒失败执行"不跨后继 start 涨栈。两处微小的调度让步是整个循环不会爆栈的保证。

### 5.4 Session 装配（execution.ts）

coordinator 的三个钩子把泛型机制接到 Session 事件与 claim：

- **`started`**（L112-116）：发布 `Execution.Started`，携带 `claimOnCommit`——**claim 写入与 started 事件同事务**（write-ahead：intent 先于任何 drain 工作落库）。
- **`drain`**（L87-110）：每次 `store.get` 重读会话 → `instances.provide(session)` 进其 Location 取 runner → 按 DrainResult 分派：`Moved` 换 Location 续跑、`Reloaded`（Location 关闭）带原 force 重入、`Complete` 结束。
- **`settled`**（L119-147）：每个 busy period **恰好一次**终态观察（覆盖所有合并的 drain）。`terminal()`（L50-54）分类：
  - succeeded → `Execution.Succeeded` + `releaseOnCommit`（释放 claim + 清 resume_attempts，store.ts L217-223）
  - interrupted 且 reason ≠ shutdown → `jobs.cancel` + `Execution.Interrupted` + release
  - interrupted 且 reason = shutdown → **不 release**（L134 的三元）——保留 claim 供下次启动续跑
  - failed → `Execution.Failed` + release

claim 三种终态一览：**开始**（started 同事务写入）、**释放**（成功/失败/用户或 inactivity 中断）、**保留**（shutdown 中断与进程死亡——后者根本没有 settled 钩子可跑）。

`interrupt` 的 `resume` 选项（L153-170）：中断后以 `nextPromotable("input")` 读 inbox——命中 steer 或控制项则以 `"steer"` scope 再按门铃（L165-168）；queued 的下一 turn 提示**留在原地**（steer-scope drain 永不提升 queued 项）。注释（L160-164）解释了为何与垂死的 drain 并发读 inbox 是安全的：投递在不可中断的发布事务里消费行，steer 行要么仍可提升、要么已完整投递无需恢复。

## 6. 恢复子系统（execution/）

execution/ 目前只有 restart.ts。核心入口 `resumeSuspendedSessions`（L193-231）——托管服务器启动时调用，embedder 可自行调用（头注释 L59-61：服务惰性，不自动跑）。

### 6.1 恢复顺序（为什么是这个序）

```
1. active = execution.active            ← 本进程已活跃的（不该有，防御）
2. pending = jobs.pendingBackground     ← 上代进程遗留的后台 job（shell/subagent）
3. children = pending 中 running 的 subagent 子会话
4. suspended = (listSuspended ∪ children) − active    ← L202-204
5. store.releaseChildClaims(children)   ← 先清孤儿子 claim
6. 逐个恢复后台 job，**shell 排在 subagent 前**：
   pending.toSorted(kind==="subagent" ? 1 : 0)       ← L207-208
   注释："Admit shell outcomes before a recovered child can start its first
         model request" —— shell 结果是环境事实，必须先于任何模型调用可见
7. 重读 execution.active（后台完成可能已唤醒父会话，L220）
8. 对每个仍 suspended 的：prepareResume 通过 → execution.resume（fork、并发不限）
9. suspended.clear()                    ← L230：异步观察者在投递时查这个集合
```

### 6.2 prepareResume（L77-99）：预算的 durable 先行

```
attempts = countResume(sessionID)       ← 先 +1 落库再决定
├─ undefined（无 claim）→ false
├─ attempts > maxAttempts(默认 10, L33) → 发布 Execution.Failed(RESUME_EXHAUSTED)
│                                        + commit 时 release（清 claim + 清零计数）
│                                        → false
└─ 否则发布 Synthetic(CONTINUE_AFTER_SERVER_RESTART, resume 语义) → true
```

注释（L78-79）点明要点：**计数 durable 先于恢复执行**——恢复路径上任何一点再崩溃，下一次 sweep 已经把这次尝试计入了，"the budget cannot be dodged"。预算是 **per-turn** 的：只有 terminal 事件清零（release 同事务），正常完成的 turn 永不累积（L26-30 接口注释）。

### 6.3 recoverShell / recoverSubagent

- **recoverShell**（L101-136）：running → cancelled（"Command cancelled because the server restarted"）；已完成的按状态补通知。通知走 `synthetic(resume:false)`（L129）——注释：重启通知**不得**唤醒空闲的 long-lived shell 拥有者；被中断的执行在通知准入后单独恢复。
- **recoverSubagent**（L138-190）：先验父子关系（child.parentID 匹配、parent 仍存在，L143-147，失联直接 completeBackground 丢弃）；非 running 直接补发结果；child 仍活跃（本进程）则跳过；否则 prepareResume 通过后 `jobs.start` 包装 `execution.resume(child)`，完成后找最后一条无错的 assistant 消息作为子代理结论（L177-182 `findLast`）交给 `SubagentCompletion.deliver`。`resume: suspended.has(parent) ? false : undefined`（L154）：父自己也要被恢复时抑制唤醒，避免双重 drain。

### 6.4 重试上限语义汇总

`DEFAULT_MAX_ATTEMPTS = 10` 是**单个 turn 的自动恢复预算**，不是会话级：一个反复在恢复路径上死掉的 turn 最多被自动拉起 10 次，第 11 次被 `RESUME_EXHAUSTED` 终结（type: "aborted"）。设计目标是防 crash-loop 跨重启（L27-29 注释），同时正常完成即清零，长会话不受影响。

### 6.5 store.ts 恢复面（L194-250 实测）

| 方法 | SQL 语义 | 关键细节 |
|---|---|---|
| `listSuspended` (L194) | `time_suspended NOT NULL AND parent_id IS NULL` | 只扫**顶层**会话——子代理 claim 由 `releaseChildClaims` 单独处理 |
| `claim` (L205) | `SET time_suspended=now WHERE id=? AND time_suspended IS NULL` | NULL 守卫：重复 claim 零行 no-op（恢复的 turn 经同一 started 钩子再 claim 天然幂等）；`time_updated` 自赋值——claim **不算用户活动**，不会把会话顶到列表前面 |
| `release` (L217) | `SET time_suspended=NULL, resume_attempts=0` | 清 claim 与清零计数**同一条 UPDATE**——与 terminal 事件同事务提交，不存在"放了 claim 但计数残留"的中间态 |
| `releaseChildClaims` (L225) | 同 release，但排除 recoverable 名单 | 恢复扫先清孤儿子 claim，可恢复的 subagent 除外 |
| `countResume` (L239) | `SET resume_attempts = resume_attempts+1 RETURNING attempts` | 读改写一条语句原子完成；返回 undefined = 无 claim（不该恢复） |

一个值得记录的复用：`time_suspended` 列名暴露了历史——它本是"会话挂起时间戳"语义，被 write-ahead claim 复用为恢复标记（4_session.md §5.2 已指出）。列没有改名，但语义已完全迁移到 claim 生命周期。

## 7. 跨进程视角

AGENTS.md 的两句原话是本节的全部纲领：

> "Keep local Session drains process-local until clustering is implemented."
> "The claim is a recovery marker, not clustered ownership, fencing, or an exactly-once guarantee."

三个推论，每条都有代码证据：

1. **drain 是进程局部的**：Coordinator 是内存 Map（run-coordinator.ts L71），execution.ts 是 `Layer.effect` 的进程全局单服务。没有任何跨进程互斥——"不同进程同时 drain 同一会话"在当前协议下**不被防御**，而是被排除：托管服务器的注册锁一次只许一个 managed server，sweep 只由 managed server 执行（restart.ts L55-61 头注释的三个协议保证：successor 只在前任确认死亡后拉起——client service kill/evict 轮询 PID；注册锁串行化；未注册进程永不 sweep）。
2. **claim 不是分布式锁**：`store.claim` 的 NULL 守卫只让重复 claim 成为零行 no-op（store.ts L206-208），它不拒绝"另一个进程已 claim"——因为它根本不是给第二个进程看的，它是给"确认前任已死"的下一次启动看的。execution.ts L76-80 注释给出一句话哲学："recovery is a property of the database, never of a shutdown hook that may not run"——恢复依据必须只依赖数据库事实。
3. **at-least-once**：restart.ts L52-53 明言 "local coordination prevents concurrent drains, not repeated external side effects after a crash"。崩溃前已发出的 provider 请求、已执行的本地工具副作用，恢复后**可能重放**——spec（"Recovery Boundaries Stay Explicit"）同义重复："does not guarantee exactly-once provider or tool behavior"。上游契约是恢复指令里的 "Continue from where you left off without repeating completed work"（L15-16）——把幂等责任交给模型的行为而非基础设施。

## 8. 不变量与设计取舍

对照 spec 与 AGENTS.md 逐条（左：不变量；右：代码保证）：

| # | 不变量 | 保证机制 |
|---|---|---|
| 1 | 准入先于执行、独立于执行（spec L5-7） | InboxEnqueued 投影落行后才 wake；`resume:false` 只录不排；admit 不需要任何执行存活 |
| 2 | inbox 表只存未消费工作（AGENTS.md） | projectDelivered 删行是投递投影的一部分；delivered 后幂等靠消息表反解 |
| 3 | 第一次准入获胜（spec L9） | reconcile：inbox 表 ?? promotedFromMessage；重试的 payload/delivery 全忽略（inbox.ts L155-167） |
| 4 | 每个 key 至多一个 drain fiber | Coordinator Map + start 只在 idle/settle 路径调用 |
| 5 | 唤醒合并、取最宽 scope | pendingWake 的 input ⊃ steer 合并（L141） |
| 6 | 每 busy period 恰一次终态 | settled 钩子只在执行 fiber 的 onExit 跑一次（L101-104），覆盖合并的 drain |
| 7 | claim 生命周期 = busy period | started/settled 钩子的 onCommit 同事务写/放 |
| 8 | 投递顺序 = enqueued_seq 序（控制项边界内） | 所有查询 ORDER BY enqueued_seq；pendingSteers 唯一的排序修正是 compaction 提队 |
| 9 | mutation 与投递互斥 | 全部经 `serialized(sessionID)`（KeyedMutex） |
| 10 | 恢复预算不可绕过 | countResume durable 先行；只有 terminal release 清零 |

**取舍**：① 门铃 vs 队列——用"合并唤醒 + 重查"取代显式工作队列，代价是 drain 的资格判断必须可重入（llm.ts 的入口检查 + 边界再检查双保险）；② compaction 提队破坏严格 FIFO——换"检查点后原文逐字可见"，这是给摘要语义让路的显式例外（源码注释自己承认）；③ steer-scope 的存在——为 interrupt-resume 提供一个"只消费插话、不碰下一 turn"的最小恢复语义，避免打断后意外开跑用户排队的下一个任务；④ 泛型 Coordinator 换取可测试性——Session 语义（事件、claim）全部在钩子里，协调器不 import 任何 Session 类型（唯一的 Session 依赖是 `import type { Promotable }`，L4）。

## 9. 失败与恢复

1. **崩溃时 inbox 残留**：行是 durable 事实，无 TTL、无清理任务。重启后若无人 wake，行静静躺着等下一次用户操作；托管服务器的恢复路径（restart 续写 Synthetic）本身会触发 wake。**唯一意外清空路径**是 revert commit 的边界删除（projector.ts L778-786）——那是语义正确的"撤销准入"。
2. **claim 泄漏（kill -9）**：进程死亡没有 settled 钩子，claim 原样留在 `time_suspended`——这正是设计（execution.ts L78-80：泄漏的 claim 就是"无 teardown 死亡"的签名）。下一次托管启动 sweep 接手；预算耗尽则 RESUME_EXHAUSTED 终结。**非托管场景**（两个未注册进程共享数据库）claim 无人 sweep，也只是残留无害——它不是锁，不阻塞任何读写。
3. **双进程同会话的竞态防护——有哪些？诚实的答案：几乎没有，靠协议排除。** 防护只有三层：(a) 托管注册锁保证同库同时只有一个 managed server（restart.ts L57-58）；(b) 未注册进程（嵌入式、测试）自觉不 sweep（L59）；(c) 数据库层最终一致——事件 seq 全序使两个进程的写入串行化，但两个进程**同时 drain** 仍会产生交错的事件序列（无围栏）。这就是 spec 把 clustering、placement、fencing 全部列为 future 的原因——"Local execution remains process-owned until clustering introduces an explicit placement and fencing protocol"。
4. **恢复期间的投递竞态**：`suspend 集合`在后台恢复完成后 `clear()`（L230），异步观察者投递时查它决定 resume 语义——窗口期内完成的 subagent 不再抑制父唤醒，之后完成的正常唤醒。
5. **结算窗口的 interrupt**（run-coordinator.ts L151-157）：终态已定但 settled 未完，interrupt 返回 false 却清 pendingWake——防止一个已被打断意图的残余唤醒在 settle 时复活成后继执行。这是状态机里最微妙的一条弧，也是"interrupt 的语义是接受而非完成"的边界案例。

## 演进注记

对照旧笔记 4.3 §5（SessionInput admit/promote 状态机，2026-08-03 基线）与现行代码：

1. **delivery 词汇是后来加的**：旧 `session_input` 表只有 admit/promote 两态，没有 `delivery` 列；steer/queue 的显式词汇、`InboxDeliveryChanged` 事件、CAS 式 projectDelivery 都是 specs 演进后进入的（现表四列 type/payload/delivery/enqueued_seq，sql.ts L125-145，含 `(session_id, delivery, enqueued_seq)` 复合索引——为按词汇查询专门建）。旧状态机的骨架（admit 幂等 / promote 删行+插消息）原样保留，词汇层是叠加而非重构。
2. **控制项入 inbox**：旧基线 compaction/move 是执行内联动作；现行作为带 delivery 的 inbox 项走同一准入/投递管道——"control item forms a delivery boundary"由此才可能成立。
3. **pendingSteers 的 compaction 提队**是精细化的产物：旧 promote 只按序；"steered compaction priority" 规则（含 move 不跨的约束）是 specs/v2 里篇幅最大的单条投递规则。
4. **协调器从内联到泛型**：旧 `execution/local.ts` 把定位 runner 与循环耦在一起；现行 run-coordinator.ts 是零 Session 依赖的泛型门铃机器，Session 装配（事件/claim/DryResult 分派）全部上移到 execution.ts 的钩子——分层是重构目标而非初始形态。
5. **write-ahead claim + restart.ts 整个文件**在旧基线不存在：旧恢复最多是"下次 wake"；现行的 per-turn 预算、durable 先行计数、RESUME_EXHAUSTED 终结、shell-before-subagent 顺序都是随托管服务器生命周期管理一起新增的。
6. **step allowance 语义**从"每次输入重置"细化为"新输入重置、批 steer 只重置一次"（llm.ts L109-111 vs L181 的双落点）——与 "turn" 概念的预留（AGENTS.md：turn 留给未来的 assistant-turn 单元）同属一步语义收紧。

## 相关篇

- [4_session.md](4_session.md)：§4/§5 概览（本篇前置）、§5.2.1 drain 主循环结构、§10 失败面总表
- [1_data-layer.md](1_data-layer.md)：Bus durable 事件、seq 分配与投影事务机制（本篇所有"同事务"声明的底层）
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)：一条 prompt 的端到端时序（runStep / 工具 / 流式细节，本篇刻意不展开）
- [7_compaction.md](7_compaction.md)：压缩算法本体（本篇只覆盖 compaction 作为**控制项**的投递边界语义）
- [11_permission.md](11_permission.md)：工具权限——drain 内 Step 执行的另一道闸门
