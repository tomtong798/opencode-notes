# 生命周期与关闭（Lifecycle & Shutdown）

> 一句话：**opencode 的退出哲学是"数据库是唯一的真相，关闭钩子只是锦上添花"——write-ahead claim 让 SIGKILL 与优雅关停留下同一种可恢复签名，Effect Scope finalizer 链把 53 节点的 Location 服务图按 owner scope 逆序收口，PTY 守护进程凭票据在 server 换代时原地交接，而双实例打架由 5 秒一轮的注册文件看门狗裁决。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，sed/grep 抽查 ≥15 处）。
> 权威出处：`AGENTS.md` "Session Core" 节（claim 语义官方声明）+ `packages/core/src/session/execution.ts:76-80` 注释（"recovery is a property of the database, never of a shutdown hook"）。
> [返回 README](README.md) | 关联 [../6_modules/15_location.md](../6_modules/15_location.md) §5（owner scope 三文件） / [../6_modules/17_job.md](../6_modules/17_job.md) §3（daemon/handoff） / [1_durability.md](1_durability.md)（claim 保留语义，并行篇）
> 本篇是**退出视角的收口**：15_location 已写的 activity 逐出、16_pty-job 已写的票据细节只引用不重写，这里回答"进程要没了，系统怎么做到体面"。

---

## 0. 恐惧清单（退出时怕什么）

```
退出 = 把"正在跑的东西"变成"要么干净结束、要么可恢复地暂停"。opencode 怕的是中间态：

├─ 恐惧 1：关服务丢用户终端内容
│    └─ 终端里可能有用户正在看的 htop/vim 输出。防线：persistent PTY 外置守护进程
│         + handoff 票据，server 换代终端进程原地不动（§4）
├─ 恐惧 2：半开的工具进程残留（MCP stdio / PTY / 后台 shell 死不掉变孤儿）
│    └─ 防线：SIGTERM → 超时 → SIGKILL 升级链是通例（mcp/stdio.ts:61、
│         client/effect/service.ts:182-199、daemon.ts:269），Effect Scope 保证
│         watcher/socket 逐个 close（§3）
├─ 恐惧 3：watcher 泄漏 fd（每打开一个目录就挂一个 parcel watcher）
│    └─ 防线：RcMap + acquireRelease，PubSub.shutdown 在 scope 关闭时必达
│         （filesystem/watcher.ts:94-116）
├─ 恐惧 4：关闭时把进行中的工作标成失败（下次打开满屏红叉）
│    └─ 防线：interrupt reason 三分（user/shutdown/inactivity），只有 user 才是
│         "用户不要了"；shutdown 保留 claim 留给下次续跑（§5）
├─ 恐惧 5：双实例退出打架（新旧 server 同时活着，各抢一半连接）
│    └─ 防线：注册文件 0600 原子写 + 每 5 秒自检"文件还是我的吗"，不是就自杀
│         （service-registration.ts:44-67）；客户端 terminate 先 SIGTERM 轮询
│         PID 再 SIGKILL（service.ts:182-199）
└─ 恐惧 6：桌面关了，服务成僵尸常驻
     └─ 防线：没有 idle 自灭——取舍是有意的：托管服务常驻换"任何客户端随开随用"，
          退出责任交给 `opencode service stop` 与版本替换（§6）
```

第 4 条是全篇的钥匙：**AGENTS.md "Session Core" 节原话**——

> "A write-ahead execution claim marks a process-local busy period for restart recovery: terminal completion, failure, or user interruption releases it, while **shutdown interruption and process death preserve it**."

（终态完成/失败/用户打断都释放 claim；**关停打断和进程死亡保留 claim**。）为什么关停要保留？因为关停不是"用户不要这个工作了"，而是"这个进程没机会干完了"——claim 留在数据库里，下次启动 `SessionRestart.resumeSuspendedSessions` 看到孤儿 claim 就续跑。§5 展开。

---

## 1. 三种退出路径全景

### 1.1 大图

```
                        ┌─────────────────────────────────────────────────┐
                        │            谁触发了"进程要没了"？                │
                        └─────────────────────────────────────────────────┘
             │                            │                              │
   [A] 服务优雅关                [B] 进程被杀                    [C] 桌面/客户端退出
   (`opencode service stop`、     (SIGKILL、crash、              (Electron 窗口关闭、
    版本替换、注册被抢)             OOM、eviction)                  浏览器 tab 关闭)
             │                            │                              │
             ▼                            ▼                              ▼
  ServerProcess.start            没有任何钩子能跑！               服务进程 detached
  (server/src/process.ts)        数据库里的 claim 原样躺着        不死（spawn 时就不是
   latch.open (:57)              execution.ts:77-80 注释：        子进程；§6）
   → boot race interrupt          "recovery is a property                     │
     (:139)                       of the database, never of                    ▼
   → Scope finalizer 链逆序       a shutdown hook"                客户端侧自己收口：
     (:82,89-94)                            │                     - 桌面：Shutdown.add
   ① status.beginStopping                  │                       注册的 effect 并发跑
     + closeAllConnections                  │                       (lifecycle/shutdown.ts)
     + application=none (:89-94)            │                     - 渲染层：pagehide 排空
   ② applicationScope 关闭        ┌─────────┴──────────┐          beforeDispose 集合
     → Instance 图 finalizer      │ 下次任何 server 启动 │          (ipc-client.ts:57-62)
     → instance.ts:172 Layer.tap  │ installRestart-     │        - Solid: onCleanup 链
       → LocationLifecycle        │ Continuity (:240-242)│          (connection.ts:234-247)
       .shutdown per location     │ → resumeSuspended-  │
   ③ Location 图内 finalizers     │   Sessions 扫孤儿    │           服务继续活着，
     (bus 退订、watcher、PTY…)     │   claim → 续跑      │           等 `service stop`
                                  └────────────────────┘           或版本替换时走 [A]
```

关键认知（实测纠正一个直觉）：**托管服务进程自己不装 SIGTERM handler**。grep 全仓 `process.on("SIGTERM")` 只有 desktop 主进程（lifecycle/index.ts:145-146）和 CLI mini-host（SIGINT，mini-host.ts:32-36）装了；core/server 包一个都没有。客户端 terminate 旧服务就是发 SIGTERM → 轮询 PID → 没死就 SIGKILL（client/effect/service.ts:182-199）——对 server 进程而言，SIGTERM 和 SIGKILL 的**可恢复语义完全一样**（claim 都在库里）。优雅关停的"优雅"只体现在 latch 路径（`service stop` 内部先 shutdown 再 kill）和桌面 Electron 的 before-quit 序列里，不体现在信号处理上。

### 1.2 路径 A：服务优雅关（实测 shutdown 序列）

`packages/server/src/process.ts`（242 行）是唯一的服务侧关闭编排：

- **latch**（:57）：`shutdown = yield* Latch.make()`，`lifecycle.onListen` 把 `shutdown.open` 交给 ServiceRegistration 当"自杀开关"（见 §6 看门狗）。
- **双 scope**（:87-94）：`applicationScope` 从父 scope fork 出来装整个路由/服务图；`Effect.addFinalizer` 注册的收尾序列是 `status.beginStopping → application 置 none → server.closeAllConnections()`——先宣告 stopping（此后 `/api/info` 返回 503+retry-after，:214、:229-232），再摘路由，再掐连接。
- **race**（:139）：`Effect.raceFirst(boot, shutdown.await |> interrupt)`——latch 一开，boot 被 interrupt，finalizer 链逆序跑：applicationScope 关闭 → Instance 图逐 Location 收口（§3）→ HTTP server scope 关闭。
- **boot 失败也是退出**（:123-136）：后台服务起不来时 `status.fail` + 关 applicationScope + `Effect.never` 挂住等外部 SIGKILL——不退出进程，因为托管服务由客户端管理生死。

### 1.3 路径 B：被杀（SIGKILL → claim 保留）

`execution.ts:76-80` 注释原文（本篇最重要的三行）：

```
// Write-ahead claim: starting records the durable intent that a turn is in flight, in the same
// transaction as the started event. Terminals release it — except shutdown interruption, which
// preserves the claim so the next server start resumes the turn. A claim that survives with no
// terminal is the signature of a process that died without teardown (crash, SIGKILL, eviction);
// recovery is a property of the database, never of a shutdown hook that may not run.
```

claim 与 `session.started` 事件**同事务**写入（`claimOnCommit`，execution.ts:81-83）——所以"跑着的工作"这个事实在 drain 开始前就已落盘，杀进程杀不掉它。孤儿 claim 的恢复见 §5；语义细节（at-least-once、非 fencing）与 [1_durability.md](1_durability.md) 互链。

### 1.4 路径 C：桌面退出（服务 detached 不死）

- 桌面主进程装了完整的 Electron 退出序列：`before-quit → will-quit → child-process-gone` + `process.on("SIGINT"/"SIGTERM")`（desktop/src/main/lifecycle/index.ts:139-146），`Shutdown.Service` 用 `Effect.cached` + `Effect.forEach(concurrency: "unbounded")` 把注册过的清理 effect **并发**跑一遍且只跑一遍（lifecycle/shutdown.ts:15-19）。
- 但这些只收口**桌面自己**的窗口/子进程。托管 server 与 `opencode-pty` 守护进程都不是桌面的子进程：PTY daemon `spawn(..., { detached: true, stdio: "ignore" })` + `child.unref()`（daemon.ts:233-240），托管服务由客户端 service 层当"contender"拉起。桌面退出后它们常驻——**这是取舍不是疏忽**（§6）。
- 渲染层的最后一口气：`pagehide` 事件（比 `unload` 可靠，移动端也触发）→ `Promise.allSettled([...beforeDispose])` 排空回调 → 才 `port.close()`（desktop/src/renderer/ipc-client.ts:57-62）。dirty storage 必须在这里冲刷（§7）。

### 1.5 一次优雅关停的完整时序（把三条链串起来走一遍）

```
opencode service stop（或版本替换 service.stop）
  │
  ├─① PtyHandoff.prepare（client/pty-handoff.ts:17）
  │    向旧 server 发 persistentPty.handoff → daemon prepare_handoff（5s 超时）
  │    → 票据 {directory, instanceID, ticket, expiresAt} 原子写进 sidecar（0600/wx/rename）
  │    失败 → 降级 persistentPty.shutdown 杀终端，绝不阻塞
  │
  ├─② terminate(info)（client/effect/service.ts:182）
  │    先核对注册文件还是这个 info（防杀错人）→ SIGTERM → 轮询 PID
  │    → 没死 → SIGKILL → 再轮询 → 清注册文件（若未易主）
  │         │
  │         ▼ 进程内（若走了 SIGTERM+latch 路径，SIGTERM 直接死则跳到数据库层）
  │    latch.open（由看门狗/stop 内部触发，process.ts:57）
  │      → race 输给 shutdown.await → boot 被 interrupt（:139）
  │      → finalizer 逆序：
  │         a. status.beginStopping + application=none + closeAllConnections（:89-94）
  │            此后 /api/info → 503 + retry-after: 1
  │         b. applicationScope 关闭 → Instance 图 finalizer
  │            → instance.ts:172 每 Location 一次 lifecycle.shutdown（§3.2）
  │         c. 图内 finalizers：bus 退订、watcher PubSub.shutdown、MCP SIGTERM→SIGKILL
  │         d. HTTP server scope 关闭
  │      → 进行中 turn 被 interrupt，reason=shutdown → claim 保留（§5）
  │
  ├─③ ensure 重招（下次任何客户端连上来）
  │    spawnContender：PtyHandoff.environment 读 sidecar → 未过期且 source 匹配
  │    → OPENCODE_PTY_HANDOFF=票据 注入新 server 环境变量（pty-handoff.ts:68-78）
  │
  └─④ 新 server 启动
       ├─ installRestartContinuity → resumeSuspendedSessions 扫孤儿 claim（§5.2）
       └─ daemon claim(inherited ticket) → 校验过期/instance_id → 成功=终端原地找回
            失败=降级 start fresh + oneShot shutdown 清旧 daemon（daemon.ts:420-423）
       → PtyHandoff.complete：source 不匹配新 Info 则 clear（票据单次有效）
```

这条链上每一环"失败"都只是降级（终端新开、恢复跳过），唯一不可降级的是数据库里的 claim——它从 turn 开始那一刻就落盘，任何一环死掉都不影响它。

---

## 2. 模块地图

```
服务侧关闭编排
├─ packages/server/src/process.ts ......... 242 行 latch/race/finalizer 序列（§1.2）
├─ packages/core/src/instance.ts:172 ...... Layer.tap：图 finalizer → lifecycle.shutdown
├─ packages/core/src/location-lifecycle.ts  70 行 单 Location 的体面关停（§3.2）
├─ packages/core/src/location-services.ts . 87 行 builds 账本：谁按 shutdown 按钮（§3.1）

被杀恢复
├─ packages/core/src/session/execution.ts . InterruptReason 三分 + claimOnCommit（§5）
├─ packages/core/src/session/execution/restart.ts  240 行 孤儿 claim 扫描续跑（§5.2）

PTY 交接
├─ packages/client/src/pty-handoff.ts ..... 129 行 sidecar 票据文件 prepare/complete（§4.1）
├─ packages/core/src/persistent-pty/daemon.ts  690 行 daemon 侧 prepare_handoff/claim（§4.2）

服务生死管理（客户端侧）
├─ packages/client/src/effect/service.ts .. terminate 升级链 + stop 的 handoff 编排（§4.1/§6）
├─ packages/cli/src/services/service-registration.ts  注册 + 双实例看门狗（§6）

客户端退出
├─ packages/desktop/src/renderer/ipc-client.ts:57-62  pagehide 排空（§7）
├─ packages/desktop/src/renderer/platform/storage.ts .. onBeforeDispose(flush)（§7）
├─ packages/client/src/solid/connection.ts . pagehide/pageshow 重连 + onCleanup（§7）
```

---

## 3. finalizer 链：Effect Scope 如何关停 53 节点的服务图

[15_location.md](../6_modules/15_location.md) §5 已给出三文件分工（activity 管逐出、lifecycle 管关停、builds 账本管按钮权）。本篇从退出视角把**链**走通。

### 3.1 builds 账本：三条路都通向同一个按钮

`location-services.ts:24-53`（实测）：每个 Location 的 Instance 图构建成功时，把 `Context.get(exit.value, LocationLifecycle.Service).shutdown` 存进 `build.close`（:44-45）。能按这个按钮的有三条路，殊途同归：

```
① Layer finalizer（服务整体退出）          ② invalidate（config 变更/逐出）     ③ 脱管 boot
   instance.ts:172                            location-services.ts:70-77            :41-42
   Layer.tap(ctx => addFinalizer(              inner.invalidate(key)                 build 不在账本里 =
     LocationLifecycle.shutdown)                 |> build?.close                       直接 lifecycle.shutdown
        │              │                        （先摘路由再关，uninterruptible）
        └──────────────┴────────────────────────┴──────────────────→ shutdown 按钮按下
```

instance.ts:172 的 `Layer.tap` 只有一行，但它是**服务优雅关停传导到每个 Location 的唯一桥梁**：applicationScope 关闭 → Instance 图 finalizer 逆序触发 → 每个 Location 各自走一遍 lifecycle.shutdown。15_location §5.1 已讲的 activity 逐出走的是 ②，语义等价。

### 3.2 LocationLifecycle.shutdown：单 Location 的体面关停

`location-lifecycle.ts:41-58`（实测 70 行全读）：

```typescript
// location-lifecycle.ts:43-58（节选）
const shutdown = Effect.suspend(() => {
  if (closed) return Deferred.await(done)        // 幂等：二连按只等第一次
  closed = true
  return Effect.gen(function* () {
    yield* permission.close    // ① 权限弹窗关掉（不能留着悬挂的询问）
    yield* forms.close         // ② 表单关掉
    yield* rpc.close           // ③ 该 Location 的 RPC 出口关掉
    yield* bus.publish(LocationEvent.Shutdown, {}, { location })  // ④ 广播
  }).pipe(Effect.ensuring(Deferred.succeed(done, undefined)))
}).pipe(Effect.uninterruptible)                   // ⑤ 关停本身不可打断
```

细节：`Effect.addFinalizer(() => unsubscribe)`（:40）保证 bus 监听随图关闭退订；`Deferred` + `closed` 标志实现幂等（三条路同时按下也不重入）；`uninterruptible` 保证半路杀进程也不会停在"权限关了、事件没广播"的中间态。

### 3.3 图内其余 finalizer（抽样实测）

每个 Location 图里的资源各自用 `acquireRelease`/`addFinalizer` 挂在图 scope 上，shutdown 时随图逆序释放：

- **watcher**：`RcMap.make` + `Effect.acquireRelease(PubSub.unbounded, (pubsub) => PubSub.shutdown(pubsub))`（filesystem/watcher.ts:94-97），且 :111 注释特意说明"scope shutdown must not wait behind an uninterruptible acquisition"——fd 不会因关闭顺序问题泄漏。
- **MCP stdio**：kill SIGTERM，`FORCE_KILL_AFTER` 超时升级 SIGKILL，注释 "Mirrors StdioClientTransport"（mcp/stdio.ts:12、:61）。
- **PTY daemon 连接**：owner socket destroy（daemon.ts:291-293 abandon）。

---

## 4. PTY handoff：退出前把终端托付给下一任

[17_job.md](../6_modules/17_job.md) §3 已写守护进程与票据协议本身。这里收口**退出时序**：旧 server 关闭前如何把终端托付出去。

### 4.1 客户端侧：sidecar 票据文件（client/src/pty-handoff.ts，129 行全读）

`service.stop({ pty: "handoff" })`（client/effect/service.ts:126-137）在 terminate 旧进程**之前**先 `PtyHandoff.prepare`——注释原话："Terminal handoff is best-effort; it must never keep the old service running"（:129）。票据落盘为 sidecar 文件 `<file>.pty-handoff`：

```
// pty-handoff.ts:54-66 简化
临时文件 = `${file}.pty-handoff.${uuid}.tmp`   // mode 0o600, flag "wx"（不存在才建）
写 { source: {id, pid, url}, handoff, expiresAt }   // handoff 可为 null（= 无可交接）
rename(临时, 正式)                              // 原子发布；失败也 force rm 临时文件
```

- **prepare**（:17-43）：向旧 server 请求 `persistentPty.handoff`；**永不抛错**——请求失败时先看是否有并发调用者已备好（:32-34），否则警告降级为 `persistentPty.shutdown`（直接杀终端，"terminals are best-effort"）。无 handoff 时 sidecar 的 `expiresAt = now + 30s`（:61），即"30 秒内可以安全清掉"。
- **environment**（:68-78）：新 server contender 拉起前读 sidecar，未过期且 source 匹配 → 注入 `OPENCODE_PTY_HANDOFF` 环境变量（JSON 票据）。
- **complete**（:80-83）：新 server 就位后核对——sidecar 的 source 若与新 Info 不一致就 `clear`（清场，票据只许一次有效交接）。
- **防僵尸链**：探测旧服务 3 次超时 → 判定不可恢复 → `PtyHandoff.clear` + terminate + 重招（service.ts:74-89，"recovery cannot preserve persistent terminals"）。

### 4.2 daemon 侧：票据的签发与核销（daemon.ts 实测）

- **prepare_handoff**（:356-378）：`startup.withPermit` 串行化；向 daemon 发 `prepare_handoff`，**5 秒超时**（:367-368 "PTY handoff preparation timed out"），响应必须是 `{type:"handoff", ticket, expires_at}`（wire schema :53），返回 `{directory, instanceID, ticket, expiresAt}`。
- **claim 继承**（:400-430）：新 server 启动带 inherited ticket → 校验未过期（:407 "PTY restart handoff expired"）→ 凭 ticket `claim` 旧注册（`own` op 带 instance_id+ticket，:471）；任何失败只**降级**不阻塞——"Discarding PTY restart handoff; persistent terminals will start fresh"（:420），并且顺手给旧 daemon 发 oneShot shutdown 清场（:423）。
- **shutdown**（:340-355）：发 `{op:"shutdown"}` 后**轮询 discover 最多 5 秒**等 daemon 真死，超时报 "opencode-pty did not stop"——daemon 是 detached 进程，杀它必须确认，否则终端进程变双份。

---

## 5. 进行中工作的退出语义：interrupt 三分与 claim

### 5.1 三种 InterruptReason（execution.ts:48）

```typescript
// execution.ts:48
type InterruptReason = "user" | "shutdown" | "inactivity"
```

| reason | 语义 | claim | 典型触发 |
|---|---|---|---|
| `user` | 用户不要了 | **释放** | TUI Esc、非交互 `process.on("SIGINT", interrupt)`（cli/run/noninteractive.ts:686） |
| `shutdown` | 进程没机会干完 | **保留** | latch 开、scope 关闭、owner-scope interrupt 缺省值 |
| `inactivity` | 没人用了（不是失败） | 释放 | activity 60min 逐出（15_location §5.1） |

判定核心 `terminal()`（execution.ts:51-55）：成功 → `succeeded`；cause 含 interrupt → `interrupted` + reason（**缺省 "shutdown"**——owner scope 里被打断的执行默认按关停处理，宁可多保留一次恢复也别误标失败）；其余 → `failed`。释放与否的分流在 :129-134：`outcome.reason !== "shutdown"` 才 `jobs.cancel`，release hook 只挂在非 shutdown 的终态提交上。

### 5.2 为什么 shutdown 保留 claim：下次恢复续跑

`restart.ts:37-41` 接口注释把闭环说全了：

> "Resumes Sessions whose execution claim was never released — turns orphaned by a process that died without teardown, or interrupted by a graceful shutdown (**which preserves the claim on purpose**). The claim is never cleared here: only a terminal event releases it, so a death anywhere in the resume path leaves the same orphaned claim for the next boot."

设计逻辑链（每环都有实测落点）：

1. **claim 与 started 同事务**（execution.ts:76-83）→ 关停钩子跑不跑都不影响 claim 存在；
2. **shutdown interrupt 不 release**（:129-134）→ 优雅关停与 SIGKILL 留下同一种数据库签名；
3. **启动扫描**：`ServerProcess.start` 里 `installRestartContinuity` fork `resumeSuspendedSessions`（process.ts:107-111、:240-242），注释明言 "no shutdown hook participates"；
4. **恢复预算**：续跑前 `countResume`，超过 10 次（`DEFAULT_MAX_ATTEMPTS`，restart.ts:28）就发 `Execution.Failed`（RESUME_EXHAUSTED）并在**同一事务**里 release——防崩溃循环（:81-90）；
5. **恢复是 at-least-once**："local coordination prevents concurrent drains, not repeated external side effects after a crash"（:48-50 注释）——工具副作用可能重放，claim 不是 exactly-once。

**interrupt 与 shutdown 的分界**一句话：打断发生在"用户意图层"（TUI/客户端主动取消）→ user，释放；打断发生在"进程容量层"（scope 关闭、服务换代）→ shutdown，保留。`inactivity` 是第三种：不是失败也不是要续跑，只是"这个目录没人看了"。

### 5.3 claim 状态机（数据库视角）

```
                 session.started 同事务写入
                        │ store.claim(sessionID)
                        ▼
                ┌──────────────┐
        ┌───────│  CLAIMED     │────────────────────────┐
        │       └──────────────┘                        │
        │ 终态事件（succeeded/failed）                    │ shutdown interrupt
        │ 或 user/inactivity interrupt                  │ 或 SIGKILL/进程死亡
        │ → release 挂在事件提交上                        │ （什么都不做）
        ▼                                               ▼
   ┌─────────┐                              下次启动 resumeSuspendedSessions
   │  IDLE   │◄────────────────────── 扫描到孤儿 claim：countResume 预算检查
   └─────────┘                                  │            │
        ▲                                  ≤10 次        >10 次
        │                                      │            │
        └── 续跑：发 Synthetic "Continuing    │      Execution.Failed
            after restart" + 恢复 shell job  │      (RESUME_EXHAUSTED)
            → 回到 turn 执行（claim 仍在，   │      + 原子 release（终态）
            续跑中途再死留给下次扫）  ────────┘
```

关键不变量（restart.ts:39-41 原话）："The claim is never cleared here: only a terminal event releases it"——恢复路径自己**无权**清 claim，任何一环死亡都留下同样的孤儿签名，恢复是幂等可重入的。

---

## 6. 空闲自愈：服务没人用了会怎样？

**实测结论：托管服务没有 idle 自灭机制。** grep `idle` 于 cli/src/services 与 server/src 均无命中（timeline/acp 的 idle 是 UI 概念）。这是显式取舍：

| 退出压力 | 由谁接 | 实测落点 |
|---|---|---|
| 桌面/所有客户端退出 | **没人接，服务常驻** | spawn contender 与 daemon 均 detached（§1.4） |
| 双实例打架 | 注册文件看门狗 | service-registration.ts:44-67 |
| 版本换代 | `service.stop({pty:"handoff"})` + terminate | service.ts:126-137、§4 |
| 用户显式停 | `opencode service stop` | 同上 |

看门狗细节（service-registration.ts:60-67，实测）：注册后每 5 秒（`Schedule.spaced("5 seconds")`）读回注册文件，逐字段核对 `id/version/url/pid/password` 是否仍是自己的（`owns`，:38-42）——**不是就调 `options.shutdown`（即 latch.open）优雅自杀**，日志 "managed service registration replaced; shutting down"；读文件失败同样自杀（:50-57 "registration check failed; shutting down"）。这解决了"两个 contender 同时拉起"的竞态：后写赢注册文件，先写的那位 5 秒内自我了断——**打架的解法不是抢，是输了就走**。

取舍的代价：`service stop` 之前服务（及其数据库连接、常驻 PTY daemon）一直占资源。换来的是：CLI/TUI/桌面任何客户端随开随连零启动等待，且终端会话跨客户端世代存活（§4）。

---

## 7. 客户端侧生命周期：pagehide 与 onCleanup 链

渲染层没有 `process.exit`，它的"死亡"是 pagehide（页面卸载）与组件销毁两条链：

**pagehide 排空链**（desktop/src/renderer/ipc-client.ts:57-62 实测）：

```typescript
// Let queued work (storage flushes) hand its messages to the port before it closes.
window.addEventListener(
  "pagehide",
  () => void Promise.allSettled([...beforeDispose].map((cb) => cb())).then(() => port.then((p) => p.close())),
  { once: true },
)
```

`beforeDispose` 是模块级 Set，`onBeforeDispose` 注册/注销（:70-75）。顺序敏感：**先排空再关端口**——storage 冲刷的最后一批 IPC 消息必须挤进 port 关闭之前。注册顺序也有讲究：desktop 的 storage 平台层特意用 `onBeforeDispose(flush)` 而非自己的 pagehide，因为"the app's own pagehide listener registers after the IPC client's, so it cannot be relied on here"（renderer/platform/storage.ts:26-28 注释）——app 层监听注册得更晚，赶不上这班车。flush 同时挂在 `visibilitychange → hidden` 上（storage.ts:38-40）做前台离场兜底；app 侧 persist 层另有自己的 pagehide flush（app/src/runtime/persistence/persist.ts:20）。

**Solid onCleanup 链**（client/src/solid/connection.ts:215-247 实测）：连接组件注册四件套——`pagehide → stop()` 断流、`pageshow → start()` 重连、`online → resync`、`visibilitychange` 前台静默检测（手机锁屏时 socket 悄悄死了，浏览器未必报错，:222-224 注释）；`onCleanup` 里摘掉全部监听并 `stop() + abort + clearTimeout + pending = []`（:234-247）——组件销毁 = 微型 shutdown，不留悬挂 fiber。

---

## 8. 失效场景矩阵

| 场景 | 实际行为 | 落点 |
|---|---|---|
| 优雅关停时有 turn 在跑 | scope 关闭 interrupt，reason=shutdown，claim 保留，下次启动续跑 | execution.ts:129-134 + restart.ts |
| SIGKILL 时有 turn 在跑 | claim 早已同事务落盘，孤儿 claim 下次扫描续跑（at-least-once） | execution.ts:76-83 |
| 续跑反复崩溃 | 10 次预算耗尽 → Execution.Failed + 原子 release | restart.ts:28、81-90 |
| 关服务丢终端？ | handoff 票据交接，daemon detached 不死，新 server 凭 ticket claim | §4 |
| handoff 请求超时/失败 | 降级 shutdown 直接杀终端（best-effort），绝不阻塞换代 | pty-handoff.ts:31-38 |
| handoff 票据过期 | 继承失败 → "start fresh" 降级 + 给旧 daemon oneShot shutdown | daemon.ts:407、420-423 |
| daemon 杀不死 | 轮询 5s 后报 "did not stop"（宁报错不装死） | daemon.ts:349-354 |
| 双 server 同时拉起 | 注册文件原子换代，输家看门狗 5s 内优雅自杀 | service-registration.ts:60-67 |
| 旧服务 SIGTERM 不理 | 客户端轮询 PID 超时 → 升级 SIGKILL | service.ts:182-199 |
| server 起不来 | status.fail + 503 + 挂住等外部管理，不自己 exit | process.ts:123-136 |
| 关闭顺序踩坑（权限关了事件没广播） | shutdown uninterruptible + Deferred 幂等 | location-lifecycle.ts:43-58 |
| watcher fd 泄漏 | RcMap acquireRelease，PubSub.shutdown 随 scope 必达 | watcher.ts:94-116 |
| 页面关闭时 storage 没冲刷 | beforeDispose 先于 port.close 排空 + visibilitychange 兜底 | ipc-client.ts:57-62、storage.ts:38-40 |
| 目录空闲 60min | inactivity interrupt（awaitSettlement）→ invalidate → lifecycle.shutdown | 15_location §5.1（引用） |
| resume 期间再死 | claim 依旧保留，下次再扫，预算继续扣 | restart.ts:39-41 |

---

## 9. 测试覆盖实测

| 测试文件 | 行数 | 覆盖点（实测断言） |
|---|---|---|
| `packages/core/test/session-execution.test.ts` | 1394 | :49-52 `terminal()` 缺省 reason=shutdown 且保留显式 reason；:55-71 "sweep only lists claimed top-level Sessions"（父保留、子清除、idle 无 claim）；:75-102 "claims at execution start, releases on completion, **preserves through teardown**"——claim 在 turn 跑时已存在，"no shutdown hook involved" |
| `packages/core/test/persistent-pty-daemon.test.ts` | 366 | :216 无 owner 时 handoff 返回 null；:226-239 shutdown 恰好发一次 op 且票据流转正确 |
| `packages/cli/test/service-lifecycle.test.ts` | 345 | :146 对 replacement.pid 发 SIGKILL 的换代竞态；:197 批量 SIGKILL 清场 |
| `packages/sdk/test/instances-lifecycle.test.ts` | — | 嵌入方 Instance 图生命周期 |
| `packages/desktop/src/main/extension/lifecycle.test.ts` | — | 桌面扩展进程生命周期 |

值得注意：**最核心的"claim 保留"语义是纯数据库性质**，所以它的测试不 mock 任何关闭钩子，直接断言 claim 在执行中途就存在于库中（session-execution.test.ts:101-102 注释与产品注释互相印证）。

---

## 10. 相关篇

- [../6_modules/15_location.md](../6_modules/15_location.md) §5/§5.1 —— owner scope 三文件与 activity 逐出链（本篇 §3 只做退出侧收口）
- [../6_modules/17_job.md](../6_modules/17_job.md) §2/§3 —— 票据协议、daemon wire 协议、migrate 细节（本篇 §4 只讲退出时序）
- [1_durability.md](1_durability.md) —— claim 的事务性与恢复语义的持久化视角（并行篇，互链）
- [7_observability.md](7_observability.md) —— 关闭/恢复路径上的日志与 status 机（stopping/failed → 503 的可观测面）
- [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) —— 桌面启动侧（退出的对偶面）
