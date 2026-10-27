# Job 注册表详解（进程内后台任务账本：身份 × 通知路由 × 崩溃恢复描述）

> 一句话：**Job 是进程内后台任务注册表——给"超出一次请求生命周期的活"发身份（id）、记通知路由（notificationID→完成后往哪个会话发消息）、管理生命周期（running→终态→≤25 条消费历史）；带 recovery 描述的任务落 KV，重启后恢复子系统照描述认领（shell 补通知、subagent 真续跑）。前台会话执行零 Job 项。**
> 代码基线：v2.0.30，`dev` 分支；行号实测（2026-10-19）。
> 拆分说明：本篇原与 PTY 合卷，2026-10 拆分——源码双向 grep 证实零关联；PTY 归 [16_pty.md](16_pty.md)。**后台 shell 的命令执行走 Shell 服务（直接 spawn），不经 PTY**——"终端面板的 shell"与"工具执行的 shell"是两条路。
> [返回 README](README.md) | 前置 [4_session.md](4_session.md)（执行循环——为什么后台任务要脱离它） | 关联 [12_subagent.md](12_subagent.md) §3（subagent 侧应用）| [16_pty.md](16_pty.md)（拆分姊妹篇）

---

## 0.5 通俗完整描述（先读这段再看细节）

有些活不该占着会话的执行循环——它可能跑很久（npm build 十分钟）、可能要在进程重启后还活着、也可能它的过程根本不该进父会话的窗口（子代理探查）。**Job 就是这类"超出一次请求生命周期的活"的账本**：登记身份、路由完成通知、崩溃后凭描述认领续办。

**Job 不是"会话状态的记录"**——前台会话的执行状态由 coordinator Map + claim 管（[4_session.md](4_session.md) §5），普通 prompt→drain→Step **从不注册 Job**。只有活被转入后台（不再阻塞会话执行循环）时才有 Job 项。

**四种用户面孔**（jobs.start 调用矩阵实测，§4.5 详述）：后台 shell（最常见）、subagent 派生、崩溃恢复认领、会话批量转后台 API。共性不是"子会话"，是**"超出一次请求生命周期的活"**——后台 shell 不一定有子会话，subagent 只是其一。

**主动转后台 vs 阻塞被转**（后台 shell 的两个分叉点，§5）：①**主动**——模型调 shell 工具时显式带 `background:true`（工具描述明言"完成时会通知你，不要轮询"），立即转后台返回"输出在流式文件里"；②**被动**——模型没要后台，前台阻塞等待期间**用户打断 turn**：block 等待返回 `backgrounded`，命令不杀、**摘掉超时**（`shell.timeout(id, 0)`——用户既然打断说明知道它在跑，让它跑完），完成时通知。两条路最终都靠 `notifyWhenDone`：结果格式化成 `<shell state=...>` XML 消息，以 synthetic 通知落进原会话（复用 notificationID 幂等），最后清 KV 标记。

一句话：**Job = 后台任务的账本（谁在跑/结果/通知给谁/崩了怎么续）——身份+通知路由+恢复描述三件套**。

---

## 0. 模块地图

```
packages/core/src/
├─ job.ts                     ★ Job 注册表本体（480 行）
│   ├─ Background schema（durable 形态：recovery+notificationID）
│   ├─ start/block/wait/background/cancel/pendingBackground
│   └─ KV 持久化 job.background/<id>（仅可恢复任务）
├─ session/subagent-job.ts     subagent 侧包装（归 12_subagent §3）
└─ session/execution/restart.ts 恢复消费方（shell 先 subagent 后）
```


---

## 1. Job 注册表（job.ts，480 行）

服务 `"@opencode/Job"`（job.ts:141），一个 **scoped、进程本地**的注册表（头部注释 job.ts:172-177：可恢复后台工作同时持有一份 durable notification 标记直到通知被准入；未消费结果跨 start→wait 交接存活；前台 block/cancel 与不可恢复 wait 结果进入 25 条已消费历史；可恢复 wait 结果留给后台登记与确认）。

**Background schema**（job.ts:9-29，durable 形态）：

```
$ sed -n 9,29p packages/core/src/job.ts
const Background = Schema.Struct({
  id: Schema.String,
  notificationID: SessionMessage.ID,
  recovery: Schema.Union([
    Schema.Struct({
      kind: Schema.Literal("shell"),
      sessionID: SessionSchema.ID,
      shellID: Schema.String,
      command: Schema.String,
    }),
    Schema.Struct({
      kind: Schema.Literal("subagent"),
      parentSessionID: SessionSchema.ID,
      childSessionID: SessionSchema.ID,
      agent: Schema.String,
      description: Schema.String,
    }),
  ]),
  status: Schema.Literals(["running", "completed", "error", "cancelled"]),
  output: Schema.optionalKey(Schema.String),
  error: Schema.optionalKey(Schema.String),
})
```

```
Background = { id, notificationID: SessionMessage.ID,
               recovery: shell{sessionID, shellID, command}
                       | subagent{parentSessionID, childSessionID, agent, description},
               status: running|completed|error|cancelled, output?, error? }
```

只有带 `recovery && notificationID` 的 job 才会 `persistBackground` 写 KV（key 前缀 `job.background/`，job.ts:37, 197-207）——**普通前台任务完全不落盘**。

进程内 `Active`（job.ts:72-83）：`done`/`backgrounded` 两个 Deferred + 独立 `Scope`（cancel 时关 fiber）+ `blockingSessions` 计数 + `isBackgrounded`/`consumed` 标志。

逐方法：

- **start**（job.ts:252-314）：原子 upsert——同 id 且 running 则幂等返回现状；否则 fork scope、登记、`Effect.forkIn` 跑 `run`，完成进 `settle`（job.ts:210-249：按 exit 判 completed/cancelled(仅中断)/error，成功 resolve `done`，**cancelled 不 persistBackground**，job.ts:240）。
- **wait**（job.ts:316-329）：可带 timeout。收尾的 tap 是消费规则核心（job.ts:325-328）：running 或**有 recovery** 的不消费——recovery 的 wait→background 是受支持的交接（"即使失败后也可以"，注释 job.ts:324）。
- **block**（job.ts:328-355）：session 语义的前台等待。modify 返回四态 `missing/finished/backgrounded/wait`；wait 态把 sessionID 计入 `blockingSessions`，然后 `raceFirst(done, backgrounded)`；finished 分支消费（consume：置 consumed、按**首次消费顺序**重排历史并裁到 `COMPLETED_LIMIT=25`，job.ts:180-192）；`Effect.ensuring(removeBlock)` 保证打断也递减计数。
- **background / backgroundAll**（job.ts:371-409）：`markBackground`（job.ts:357-368）置位、清 blockingSessions、**没有 notificationID 时补造一个**（仅 recovery job）、persistBackground；然后 resolve `backgrounded` Deferred 唤醒所有 block 方。background 允许在 job 已结束后补登记（"可恢复工作可能在调用者后台化之前就完成了"，job.ts:377-378）。backgroundAll 按 session+type 批量。
- **cancel**（job.ts:411-439）：置 cancelled、resolve done、**关闭 scope 杀 fiber**、消费。注意 cancel 也 persistBackground（与 settle 的 cancelled 分支不同——主动取消要留下标记以便通知层补一条 cancelled 通知）。
- **pendingBackground**（job.ts:441-450）：KV 分页 scan 出全部待恢复 Background。
- **completeBackground**（job.ts:452-461）：删 KV 标记 + 把内存里对应 job 摘除。由通知送达方调用（见 §5 notifyWhenDone 尾部）。

subagent 侧如何使用这套 API（Job ID == childSessionID 等）归 [12_subagent.md](12_subagent.md) §3，不重复。

一个 job 的完整状态机（进程内视角）：

```
                 start(id)
                    │ fork scope + fiber(run)
                    ▼
              ┌──────────┐  settle(success)   ┌───────────┐
              │ running  │───────────────────▶│ completed │
              │          │                    └─────┬─────┘
              │          │ settle(仅中断)            │ consume(block/cancel/wait 无 recovery)
              │          │──────────────▶┌──────────┐│
              │          │               │ cancelled ││
              │          │ settle(其余)   └──────────┘│
              │          │──────────────▶┌───────┐   │
              │          │               │ error │───┤
              │          │               └───────┘   ▼
              │          │  background/backgroundAll  [consumed 历史 ≤25]
              │          │──────────────────▶┌──────────────┐
              └──────────┘                    │ backgrounded │ 终态后仍收
                    │ cancel                 └──────┬───────┘ 通知
                    └──────────────▶ cancelled ─────┘

  旁路（仅 recovery+notificationID 的 job）：
    running/settled ──persistBackground──▶ KV "job.background/<notificationID>"
    completeBackground(notificationID) ◀── 通知送达时删除
```

## 2. 后台 Shell（主动/被动转后台与通知）

`packages/core/src/tool/plugin/shell.ts`（293 行）。shell 工具是 Job 的第一个消费者，也是"前台/后台 shell 的分叉点"：

```
execute(input)
  │
  ├─ shell.create(...)            建 Shell 服务实例（capture 流，与 Pty 无关）
  │    └─ prepare(): ShellParse.scan → 权限断言（§6）
  ├─ jobs.start({ id: shellID, recovery:{kind:"shell",...}, run })
  │                                       (shell.ts:230-252)
  ├─ input.background === true ──────────┐
  │                                      ▼
  │              jobs.background(job.id) + notifyWhenDone fork
  │              → 返回 backgroundResult（"Output is streaming to: <file>"）
  │
  └─ 否则 jobs.block({id, sessionID})     (shell.ts:260-262)
       ├─ finished  → 正常返回输出
       ├─ backgrounded → shell.timeout(id, 0)   ← 摘掉超时！
       │                 + notifyWhenDone → backgroundResult
       └─ 调用方被打断 → Effect.onInterrupt(jobs.cancel)  (shell.ts:262)
```

两个分叉点（shell.ts:254-267），实测源码：

```
$ sed -n 254,268p packages/core/src/tool/plugin/shell.ts
              if (input.background === true) {
                yield* jobs.background(job.id)
                yield* notifyWhenDone(context.sessionID, job.id, info.id, info.command, settled)
                return backgroundResult(info.id, info.file)
              }

              const result = yield* jobs
                .block({ id: job.id, sessionID: context.sessionID })
                .pipe(Effect.onInterrupt(() => jobs.cancel(job.id).pipe(Effect.ignore)))
              if (result?.type === "backgrounded") {
                yield* shell.timeout(info.id, 0)
                yield* notifyWhenDone(context.sessionID, job.id, info.id, info.command, settled)
                return backgroundResult(info.id, info.file)
              }
```
1. **主动后台**：`background:true`（工具描述明言"完成时会通知你，不要轮询"，shell.ts:59）→ 立即 `jobs.background`，返回流式文件路径。后台默认 **timeout=0（无超时）**（shell.ts:201：`input.background===true ? (input.timeout ?? 0) : ...DEFAULT_TIMEOUT_MS`）。
2. **被动后台**：用户在 block 等待期间打断 turn → block 返回 `backgrounded`（backgroundAll 由 session 执行侧触发，见 12_subagent §3 同机制）→ **`shell.timeout(info.id, 0)` 摘掉超时**让 dev server 类长命令继续跑，同样走通知。

**通知**（notifyWhenDone，shell.ts:163-188）：fork 的 fiber `jobs.wait` 到终态，把结果格式化为 `<shell id="..." state="..." command="...">...</shell>` XML 消息（`shell/result.ts:50-67` 的 `notification()`，metadata 带 exit/signal/timeout），以 `sessions.synthetic` 落成一条**复用 job.notificationID** 的通知消息（幂等锚点），最后 `jobs.completeBackground` 清 KV 标记。前台 shell turn 的 HTTP 同步等待与 1500ms 宽限归 [../5_timeline/6](../5_timeline/6_tui-cli-prompt-chain.md)。

**崩溃恢复**（`session/execution/restart.ts`，240 行）：boot 扫 `jobs.pendingBackground`，按 **shell 先、subagent 后**排序准入（restart.ts:208 的排序注释："在恢复的 child 发起第一个模型请求之前先准入 shell 结果"）。`recoverShell`（restart.ts:101-137）：进程死了 shell 结果必然不可再得，running 一律改判 **cancelled**：

```
$ sed -n 101,110p packages/core/src/session/execution/restart.ts
      const recoverShell = Effect.fnUntraced(function* (
        background: Job.Background,
        recovery: Extract<Job.Recovery, { kind: "shell" }>,
      ) {
        const state = background.status === "running" ? "cancelled" : background.status
        const text =
          background.status === "running"
            ? "Command cancelled because the server restarted"
```

其余状态原样补发 synthetic 通知（`resume:false`——重启通知不许唤醒空闲 session，restart.ts:130-131），然后 `completeBackground` 清标记。与 subagent 分支（可真正 resume child 执行，restart.ts:139-190，预算耗尽降级为 error 通知）的差异见 §8 矩阵。


## 2.5 后台任务的四张面孔（调用矩阵与场景）

jobs.start 的全部调用方（grep 实测）与对应场景：

```
├─ tool/plugin/shell.ts:239      后台 shell（最大用户）
│    主动转后台（background:true）或阻塞被转（用户打断）
├─ session/execution/restart.ts:168  崩溃恢复
│    重启时把孤儿后台任务重新登记续办
├─ session.ts:444 backgroundAll  会话批量转后台 API（编排面）
│    把某会话名下全部 pending 任务转后台——SDK/脚本用，
│    UI 暂无入口（grep app/server 零调用方）
└─ （subagent 派生经 SubagentJob 包装，12_subagent §3）
```

四个用户场景时间线（示例化）：

**① 后台 shell（最常见）**：

```
你：跑一下 npm test
模型：调 shell 工具（预计 30s）→ 你打断 → 命令转后台（不杀）
你：继续问"顺便看下 package.json" → 模型正常回答
     ★执行循环没被测试阻塞
…2 分钟后…会话里出现通知消息：
  "✓ npm test 完成（输出在 shell:xyz）" ← notifyWhenDone→synthetic
```

**② 子代理探查**：subagent 工具派生子会话（parentID 回指），前台等待版不注册 Job；被转后台或崩溃恢复时才以 Job 形态出现（过程不进父窗口，结果回流）。

**③ 崩溃后认领**：

```
昨天：后台 npm build（running）+ 后台子代理 | 进程 kill -9
今天：重启 → 恢复子系统扫 KV 的 Background 行：
  ├ recovery.kind=shell → 知道"当时在跑什么命令"
  │   → shell 进程已死不可续 → 改判 cancelled 补通知
  └ recovery.kind=subagent → 知道"子会话是谁"
      → execution.resume(child) 真正续跑 + 完成通知
  ★recovery 字段就是"任务说明书"——没有它重启后无人知道该续什么
```

**④ 编排面**：`backgroundAll`——自动化脚本收尾时"这些活别跟着会话死，跑完通知"。

## 2.6 block 的双通道等待与"被动转后台"的机制细节

block（job.ts:328-355）是前台等待的核心，也是"被动转后台"的发生地：

```
block({id, sessionID})
│
├─ modify 原子查四态：
│   ├ missing → undefined（任务不存在）
│   ├ finished（已终态）→ 直接消费返回
│   ├ backgrounded（已被转后台）→ 直接返回 backgrounded
│   └ running → ★wait 态：
│       ├ sessionID 计入 blockingSessions（计数）
│       └ Effect.race_first(
│            Deferred.await(done),          ← 通道1：正常完成
│            Deferred.await(backgrounded))  ← 通道2：有人调了 background()
│          )
│          └ Ensuring(removeBlock)——打断也递减计数
```

**"阻塞被转"的完整链**（shell 工具视角，shell.ts:260-268）：模型前台 block 等待 → 用户 Esc 打断 turn → **执行层把该会话全部 Job backgroundAll**（session.ts:444，12_subagent §3 同机制）→ backgrounded Deferred resolve → block 的 race 走通道 2 返回 → shell.ts 收到后 **`shell.timeout(id, 0)` 摘掉超时**（用户知道它在跑，让它跑完）→ notifyWhenDone 挂通知 → 工具立即返回"已转后台"（模型可继续对话）。

对比"主动转后台"（:254-258）：模型自己带 `background:true` 调用 → 不进 block 直接 background——**差别只在"谁决定"**（模型判断长命令该后台 vs 用户打断强制），转后台后的处理（摘超时/通知/幂等标记）完全一致。

## 3. 不变量与设计取舍（Job 部分）

- **Job 是进程本地的，durable 只覆盖"通知义务"**：start 注释（job.ts:172-177）明确了什么落盘（recovery+notificationID）什么不落（前台结果、fiber 本身）。崩溃后**不恢复执行**，只恢复**通知**——shell 进程死了就补 cancelled 通知，subagent 才有 resume。
- **消费历史按首次消费排序**（consume，job.ts:185 注释），不是 start 时间也不是后续读取时间。
- **重启 handoff 永不让位给启动**：终端 best-effort，任何失败 abandon 旧 daemon 换新目录，restart 本身必成（daemon.ts:413-414, 418-430）。
- **rename 保 inode 的迁移**：旧版本 daemon 目录搬家后锁仍有效（daemon.ts:433-434 注释）——对"文件锁跟着 inode 走"的精确利用。
- **opaque ≠ deny**：扫描器承认解析边界，把不可分析的命令如实上报为 opaque，让权限层（而非解析器）做决定（scan.ts:14）。
- **持久终端的 ID 信任边界**：`fromID` 严格校验 `pty_persistent_<正整数>`，不合即 throw（index.ts:414-419）——daemon 内部序号不被伪造。


---

## 4. 失败与恢复（Job 恢复矩阵）

**Job 恢复矩阵**（restart.ts boot 扫描，shell 先 subagent 后）：

| recovery 分支 | running（进程死时） | completed/error | cancelled |
|---|---|---|---|
| **shell** | 改判 **cancelled** 补通知（进程输出已失，不可能续）restart.ts:103-110 | 原状态补通知（output/error 原样）restart.ts:111-117 | 通知 "Command cancelled" |
| **subagent** | child 会话是 durable 的：父/子会话仍存在且子不活跃 → `jobs.start` 重跑 `execution.resume(child)` 再 background + wait→notify（restart.ts:160-190）；预算超限（默认 10 次，restart.ts:27）→ error 通知；孤儿（父或子没了）→ 仅清标记 restart.ts:144-147 | 补发完成通知（可按 suspended 决定 resume:false） | 同左 |

共性收尾：每条恢复都以 `completeBackground(notificationID)` 清 KV 标记结束；恢复通知 `resume:false` 防止长命 shell 的空闲 owner 被误唤醒（restart.ts:130-131）。


## 演进注记

- **Job 的 Background schema 把 shell/subagent 两种 recovery 显式编码进同一 Union**（job.ts:14-27），subagent-job 逻辑独立成 `session/subagent-job.ts`——注册表本体保持领域无关。
- **与 PTY 合卷是旧笔记的历史包袱**：旧笔记 4.16（pty+shell tool）+4.22（background-job）在 V2 重建后语义已分家（shell 工具执行不经 Pty），2026-10 笔记层拆分归位。

## 相关篇

- [12_subagent.md](12_subagent.md) §3——Job 的 subagent 侧应用（ID 复用、notify 去重、block vs backgrounded 对父会话的影响）
- [4_session.md](4_session.md)——前台执行循环（Job 的反面：请求内完成的不进账本）
- [16_pty.md](16_pty.md)——拆分姊妹篇（终端面板的 PTY——与本文零代码关联）
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §0.5 第二幕——重启恢复扫描中 Job 的位置
