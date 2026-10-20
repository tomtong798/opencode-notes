# PTY 与后台作业详解（伪终端 × 进程内 Job 注册表 × shell 解析器）

> 一句话：**PTY 子系统提供"可交互伪终端"两档实现——进程内 `Pty`（server 内存活）与外部守护进程 `PersistentPty`（server 重启也活）；Job 是进程内后台任务注册表，带 durable notification 标记与 shell/subagent 两分支崩溃恢复；shell/ 解析器负责把一条命令安全地拆成"资源前缀 + 目录变更"供权限系统裁决。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19，`wc -l` 全量核对）。
> 权威出处：`packages/core/src/pty.ts` + `packages/core/src/pty/` + `packages/core/src/persistent-pty*` + `packages/core/src/job.ts`（480 行，非传闻的 760+）+ `packages/core/src/shell/`
> [返回 README](README.md) | 前置 [4_session.md](4_session.md)（shell 工具挂在 session 执行链上） | 关联 [12_subagent.md](12_subagent.md) §3（Job 的 subagent 侧应用：Job ID == childSessionID、notify 去重） | [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)（shell turn 的 TUI 侧等待语义）

---

## 0. 模块地图

```
packages/core/src/
├─ pty.ts                    ★ Pty 域服务（进程内 PTY，309 行）
│   └─ create/update/write/attach/remove + 2MB 环形缓冲 + 25 个退出会话保留
├─ pty/                      （平台适配 + 线协议 + 票据，205 行）
│   ├─ pty.ts              Proc/Opts 类型接口（25 行）
│   ├─ pty.bun.ts          bun-ffi + bun-pty 后端，Win 清 Ctrl+C ignore（44 行）
│   ├─ pty.node.ts         @lydell/node-pty 后端，SEA 用 ConPTY DLL（34 行）
│   ├─ pty.workerd.ts      workerd 直接抛错（9 行）
│   ├─ protocol.ts         WS 线协议：原始 chunk + 0x00 元帧（37 行）
│   └─ ticket.ts           PtyTicket：60s TTL 连接票据，容量 1 万（56 行）
├─ persistent-pty.ts         纯 re-export（1 行）
├─ persistent-pty/           ★ 持久 PTY：外部 opencode-pty 守护进程客户端（1208 行）
│   ├─ index.ts            域服务：list/create/write/resize/input/snapshot/read/attach（419 行）
│   ├─ daemon.ts           传输层：spawn 守护进程 + Unix socket 帧协议 + handoff（690 行）
│   ├─ pty-binding.ts      @opencode-ai/pty 二进制绑定描述（7 行）
│   ├─ binary.bun.ts       Bun 静态嵌入二进制（85 行）
│   ├─ binary.node.ts      Node：环境变量/PATH 查找（6 行）
│   └─ binary.workerd.ts   不可用（5 行）
├─ job.ts                    ★ Job 注册表：start/wait/block/background/cancel（480 行）
└─ shell/                    （shell 命令解析，3143 行）
    ├─ parse.ts            扫描入口：legacy(tree-sitter)/portable 双路径 + ARITY 表（403 行）
    ├─ scan.ts             可移植 bash/zsh/posix/PowerShell 手写扫描器（2409 行）
    ├─ select.ts           ShellSelect：$SHELL 发现/兼容性过滤/login 参数（227 行）
    ├─ result.ts           通知/输出格式化（<shell> XML + metadata）（81 行）
    ├─ parser-wasm.node.ts require.resolve 三份 wasm（11 行）
    ├─ parser-wasm.bun.ts  静态 file import 三份 wasm（8 行）
    └─ parser-wasm.workerd.ts 空串占位（4 行）
```

server 侧接缝（core 外，一句带过）：`packages/server/src/handlers/pty.ts`（221 行）把 `Pty.attach` 适配到 `api/pty` 的 WebSocket（票据走 `PtyTicket.issue/consume`，replay 按 `PtyProtocol.chunks` 切 64KB 帧）；`handlers/persistent-pty.ts`（225 行）做同样的事但走 PersistentPty。

## 1. 心智模型

三件事，故意分开：

- **Pty**：一个**进程内**的伪终端会话注册表。server 死 → PTY 全死。面向 TUI/Desktop 的交互式终端面板。
- **PersistentPty**：同样面向终端面板，但伪终端跑在**外部守护进程 `opencode-pty`** 里，server 重启/热更新后凭 runtime 目录 + handoff 票据找回**同一个**终端进程。与 Job 无关。
- **Job**：**进程内**的后台任务注册表（Effect fiber 的包装），核心问题是"工具调用想先返回、结果后送达"——输出走 synthetic notification 消息，崩溃后凭 KV 里的 recovery 标记补通知。

```
        终端面板(UI)                LLM 工具调用
             │ ws                        │ shell tool
             ▼                           ▼
   ┌─ server handlers ─┐        ┌─ tool/plugin/shell.ts ─┐
   │  pty.ts           │        │  Job.start → block/    │
   │  persistent-pty.ts│        │  background + notify   │
   └───┬────────┬──────┘        └──────────┬─────────────┘
       ▼        ▼                          ▼
   Pty 服务  PersistentPty 服务        Job 服务(job.ts)
   (进程内    (外部 daemon)            (进程内 fiber
    bun-pty/   │ unix socket            注册表 + KV 标记)
    node-pty)  ▼
           opencode-pty 守护进程（detached，server 死它不死）

   交汇点：shell 工具。前台 shell = block 等 Job；
   用户 esc / background:true → Job.background → 通知消息。
   注意：后台 shell 的进程本身在 Shell 服务里（capture 流），
   不在 Pty/PersistentPty 里——那两档只服务终端面板。
```

防重复：Job 在 subagent 场景的语义（Job ID == childSessionID、notify 去重、block vs backgrounded 对父会话的影响）见 [12_subagent.md](12_subagent.md) §3；shell turn 在 TUI 侧的等待/打断语义见 [../5_timeline/6](../5_timeline/6_tui-cli-prompt-chain.md)。本篇只写机制本体。

## 2. PTY 生命周期（进程内档）

`packages/core/src/pty.ts`（309 行），服务 ID `"@opencode/Pty"`（pty.ts:91）。

**创建**（`create`，pty.ts:166）：
- ID 用 `ID.ascending()`；默认 shell 来自 `ShellSelect.resolve({priority:"config"})`（pty.ts:168），login shell（bash/zsh/sh/dash/ksh）自动追加 `-l`（pty.ts:169，meta 表在 select.ts:10-20）。
- env 强制 `TERM=xterm-256color` + `OPENCODE_TERMINAL=1`（pty.ts:181-184）；Windows 再补 LC_ALL/LC_CTYPE/LANG（pty.ts:186-190）。
- spawn 是**同步**的：`yield* Effect.sync(() => spawn(...))`（pty.ts:199）——平台后端由 `lazy(() => import("#pty"))` 延迟选择（pty.ts:17）。

**IO 流**（pty.ts:205-236）：`proc.onData` 把 chunk 三路分发——(1) 推给每个 active subscriber；(2) inactive subscriber 暂存 `pending`；(3) 追加进 `buffer`。buffer 超 `BUFFER_LIMIT = 2MB`（pty.ts:12）就从头部裁掉并推进 `bufferCursor`——这是"保留回放"而非无限累积。`proc.onExit` 置状态、发 `Pty.Event.Exited`，并把 id 推入 `exitOrder`，超过 `EXITED_LIMIT = 25`（pty.ts:15）删最老的——注释明说"退出会话保持可观察直到显式 remove，封顶防止废弃终端无限累积"（pty.ts:13-14）。

**attach**（pty.ts:260-307）：订阅者先拿到 `replay`（从请求 cursor 到当前末尾的保留输出；`cursor:-1` 表示从最新处 tail，pty.ts:64-66），随后调 `activate()` 才开始收 live 数据——两段式是为了让客户端先应用 replay 与 cursor 元数据再接流，中间到达的数据在 `pending` 里排队不丢。`write`/`resize` 挂在返回的 Attachment 与 `update`（pty.ts:247）上。

**平台后端**（`pty/` 目录）。Bun 后端（44 行）先处理 Windows 特例再 spawn：

```
$ sed -n 12,20p packages/core/src/pty/pty.bun.ts
if (process.platform === "win32") {
  const library = dlopen("kernel32.dll", {
    SetConsoleCtrlHandler: { args: ["ptr", "i32"], returns: "i32" },
    GetLastError: { args: [], returns: "u32" },
  })
  try {
    // Detached servers start with Ctrl+C ignored, and ConPTY shells inherit it.
    // Clear that attribute once before spawning shells; keep registered handlers.
```

Node 后端（34 行）经 `createRequire` 动态加载，SEA 打包有专门分支：

```
$ sed -n 9,15p packages/core/src/pty/pty.node.ts
const pty = createRequire(import.meta.url)(process.env.OPENCODE_NODE_PTY_PATH ?? "@lydell/node-pty") as {
  spawn: typeof spawn
}

function spawnPty(file: string, args: string[], opts: Opts): Proc {
  const proc = pty.spawn(file, args, process.platform === "win32" && isSea() ? { ...opts, useConptyDll: true } : opts)
```

workerd 后端（9 行）显式失败、不静默降级：

```
$ sed -n 4,9p packages/core/src/pty/pty.workerd.ts
// workerd cannot spawn processes; the Pty service surfaces this as a defect if
// a terminal is ever requested on this runtime.
export function spawn(): Proc {
  throw new Error("Pseudo-terminals are unavailable on the workerd runtime")
}
```

三后端归一到 `pty/pty.ts`（25 行）的 `Proc` 接口：`pid/onData/onExit/write/resize/kill` 六成员——域服务只见这份契约，`lazy(() => import("#pty"))` 按运行时解析到其中一份（pty.ts:17）。

**票据**（`pty/ticket.ts`）：`PtyTicket` 是 Effect `Cache`（非真查询：`noLookup` 故意 die，ticket.ts:27-28）。`issue` 发 UUID + 60s TTL（ticket.ts:9-10），`consume` 用 `Cache.invalidateWhen` **原子地一次性核销**且 scope（ptyID+directory+workspaceID）必须全等（ticket.ts:24）。用途：WS 直连端点无法继承 HTTP 鉴权，客户端先走 HTTP 拿票再带票建 WS（见 `handlers/pty.ts:120-156`）。

**线协议**（`pty/protocol.ts`）：出站帧 = 原始 UTF-8 终端 chunk；控制帧 = `0x00 + JSON{cursor}`，只在 replay 结束后发一次，让客户端记住绝对输出游标以便断线重连（protocol.ts:5-7）。replay 按 `REPLAY_CHUNK = 64KB`（protocol.ts:13）分帧。入站非法 UTF-8 直接丢弃（protocol.ts:31-38）。

## 3. Persistent PTY

`packages/core/src/persistent-pty/index.ts`（419 行，服务 `"@opencode/PersistentPty"`，index.ts:117）+ `daemon.ts`（690 行传输层）。Windows 不可用（`binary.node.ts:2` 的 `available`），workerd 不可用。

**架构**：伪终端不住在 server 进程里，而住在独立守护进程 `opencode-pty daemon --runtime-dir <root> --name <id>` 里（daemon.ts:228-262，`detached:true, stdio:"ignore"`，spawn 后 `unref()`）。server 通过 `<root>/<id>/service.json`（Registration：instance_id/pid/protocol/socket/token，daemon.ts:23-29）发现它，用 Unix socket + 4 字节长度前缀 JSON 帧通信（`MaxFrameBytes = 8MB`，daemon.ts:9；编码/解码器在 daemon.ts:655-687）。协议版本 `ProtocolVersion = 7`（daemon.ts:8），不匹配即拒连（daemon.ts:210-218）。

客户端可发的请求 op（request/subscribe 两通道）与响应（WireResponse Union，daemon.ts:47-106）：

```
op: ping | own | create | write | resize | control | input | snapshot
    | read_rows | terminate | list | shutdown | prepare_handoff | subscribe
resp: pong | owned | created | ok | terminals | handoff | snapshot | rows
      | attached | resized | exited | controller_changed | title_changed
      | foreground_process_changed | error
```

注意 `write`（不带 attachment，单向写）与 `input`（带 attachmentID+尺寸，控制器输入并同步 resize）是两个 op；`snapshot` 回文本+checkpoint+光标，`read_rows` 回定行数屏幕行——两者都依赖 daemon 内置的终端模拟状态，这是进程内档没有的能力。

**连接生命周期**（daemon.ts:177-345）：
1. `discover`：读 service.json → 校验协议版本 → one-shot `ping` 并核对 instance_id/pid。
   ```
   $ sed -n 210,218p packages/core/src/persistent-pty/daemon.ts
   if (decoded.protocol !== ProtocolVersion)
     return yield* Effect.fail(
       new DaemonError({
         kind: "protocol",
         message: `opencode-pty protocol mismatch: daemon=${decoded.protocol}, client=${ProtocolVersion}`,
         pid: decoded.pid,
       }),
     )
   ```
2. 不存在则 `spawnDaemon`：起子进程，5s 内每 50ms 轮询 discover；失败 SIGTERM 收尾。
3. `claim`：长连 socket 发 `{op:"own", instance_id, ticket}`，daemon 回 `owned` 才算取得所有权（daemon.ts:448-483）。锁定失败（"unreachable daemon 还握着目录锁"）→ `abandon()`：换一个全新 UUID 目录重试一次（daemon.ts:282-290）。
4. 所有请求经 `request()`：registration 类错误自动重连一次（daemon.ts:322-334）。

**会话重连怎么找回同一个 PTY**——三个机制叠加：
- **ID 稳定**：PersistentPty 的 ID 是 `pty_persistent_<daemon 内部序号>`（`toID`/`fromID`，index.ts:410-419）；daemon 活着，终端就在，ID 不变。
- **游标重放**：`attach(id, {cursor, attachmentID, role, takeover, onEvent})`（index.ts:326-345）→ daemon `subscribe`。首帧 `attached` 带 `replay`（requestedOffset/availableOffset/endOffset/truncated/data，daemon.ts:75-86）——**服务端只保证从 availableOffset 起**，请求更早则 truncated。随后二进制 output 帧格式 `0x00 + u64 start + u64 end + data`（daemon.ts:590-600）。role 为 `controller`（可写）或 `observer`（只读多路旁观）；controller 变更广播 `controller_changed` + generation。
- **重启 handoff**：旧 server 关闭前 `handoff()` 发 `prepare_handoff` 拿 `{ticket, expiresAt}`（daemon.ts:358-377）；新 server 启动时带 `Handoff` 选项 `makeDaemonTransport(..., options.handoff)`（index.ts:128-133），继承方校验 expiresAt/instance_id、必要时 `migrate`（把旧版本临时目录里的 service.lock/json **rename** 到新 root——rename 保 inode，还活着的 daemon 仍握着它的锁，daemon.ts:434-448），再凭 ticket `claim`。任何失败都只降级（"persistent terminals will start fresh"，daemon.ts:420-430），**不阻塞 server 启动**（注释：daemon.ts:413-414）。

**与普通 Pty 的差异**：

| | Pty（pty.ts） | PersistentPty |
|---|---|---|
| 进程宿主 | server 进程内 | 外部 daemon（detached） |
| server 重启 | 全部死亡 | 终端存活，handoff/重新 discover 找回 |
| 回放 | 内存 2MB 环形缓冲 | daemon 侧持久输出 + offset 游标 |
| 控制权 | attach 即可写 | controller/observer 角色 + generation + takeover |
| 快照 | 无（只有原始流） | `snapshot`（文本+checkpoint+光标）与 `read_rows`（按行读屏） |
| 平台 | 全平台 | 非 Windows、非 workerd |

另有两个细节：`read(sessionID, lines?)`（index.ts:265-309）按 session 记"当前终端"（`current` Map：controller 活动才选中，observer 读不算，index.ts:137），供 shell 工具把终端屏幕喂给模型；`attach` 收到 `exited` 时自动 `removeVisibleExit`（index.ts:311-324，去重防止并发 remove）。

## 4. Job 注册表（job.ts，480 行）

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

## 5. 后台 Shell

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

## 6. Shell 解析器

shell 工具在 spawn 之前必须回答权限系统两个问题：这条命令将动哪些**资源前缀**（permission.assert 的 `resources`/`save`）、会不会**换目录**（cd 类要授权目标目录）。这就是 `shell/` 的职责——它不是"危险命令拦截器"，而是**权限证据提取器**（拦截决策在 11_permission）。

**入口** `ShellParse.scan`（parse.ts:164-173）：

```
scan(command, shell, cwd, {portable})
  ├─ portable || wasm.bash === ""  → scanPortable（手写扫描器 scan.ts）
  └─ 否则                          → scanLegacy（web-tree-sitter）
```

选择发生在 shell.ts:135-137：config `experimental.portable_shell_scanner`，缺省在 wasm 不可用（workerd）或 local/dev channel 时 portable。

**legacy 路径**（parse.ts:175-207）：加载 tree-sitter（bash 或 powershell，按 `ShellSelect.ps(shell)` 选），遍历 `command` 节点：首 token 在 `CWD` 集合（cd/chdir/popd/pushd/push-location/set-location，parse.ts:16）→ 解析出目录参数；否则整条命令（含重定向父节点）作 `resource`，`save` 前缀由 ARITY 表决定。

**ARITY 表**（parse.ts:23-161）：命令名 → 参与权限前缀的词数（`git: 2`、`docker compose: 3`、`cat: 1`……），`prefix()` 截取后拼 `"git config *"` 这样的保存规则。`PREFIX_LENGTH` 取全表最大值（parse.ts:162）。

**portable 路径**（parse.ts:209-266 → scan.ts 2409 行手写扫描器）：纯 TS 的 bash/zsh/posix/PowerShell 词法+结构分析。这是全模块最重的文件，要点：
- 返回 `scanned(commands)` 或 `opaque(reason)`——**opaque 是"解析受限"而非"拒绝"**（scan.ts:14 注释），九种原因：command-substitution/compound-command/dynamic-command-name/dynamic-execution/heredoc/invalid-redirect/invalid-structure/unterminated-escape/unterminated-quote（scan.ts:3-13）。
- 防御上限：`MAX_INPUT_LENGTH = 64KB`（超长即 opaque，scan.ts:89,209）、`MAX_SUBSTITUTION_DEPTH = 32`（嵌套替换深度，scan.ts:90）、`remaining = 64KB×32` 的工作量配额（scan.ts:210）——保证扫描器自己不被恶意输入卡死：
  ```
  $ sed -n 208,215p packages/core/src/shell/scan.ts
  export function scan(input: string, dialect: Dialect = "posix"): Result {
    if (input.length > MAX_INPUT_LENGTH) return { kind: "opaque", reason: "invalid-structure" }
    const context: BashContext = { remaining: MAX_INPUT_LENGTH * MAX_SUBSTITUTION_DEPTH, dialect }
    const result = scanBash(input, 0, 0, context)
    if (result.kind === "opaque") return result
    if (context.boundDollar && context.boundOpen && context.variableSink)
      return { kind: "opaque", reason: "dynamic-execution" }
  ```
- 方言知识密集：bash 重定向/列表/case 操作符表（scan.ts:29-31）、绑定型 builtin（declare/export/read/…，scan.ts:33-54）、zsh 预命令修饰（noglob/exec/…，scan.ts:61-62）、PowerShell 参数语法。`dynamic-execution` 由 bound `$` 变量触发的动态执行检测（scan.ts:212-214）。
- parse.ts:224-266 把 portable 结果适配回 legacy 形状（跳过声明语句、PowerShell foreach 语句头特判、只认字面量引号为静态目录——"只有字面量引号解析成静态目录"，parse.ts:265）。

**wasm 三平台适配**（parser-wasm.*.ts）：node 用 `require.resolve` 解析 `web-tree-sitter/tree-sitter.wasm` + bash + powershell 三份（可被 `OPENCODE_TREE_SITTER_*_WASM_PATH` 覆盖）；bun 用 `import ... with {type:"file"}` 静态嵌进 CLI 产物；workerd 返回空串——正是 scan 入口 `wasm.bash === ""` 判 portable 的原因。

**ShellSelect**（select.ts）：`$SHELL` 发现 + `META` 表（fish/nu deny 不兼容；powershell/pwsh 走 ps 分支加 `-NoLogo -NoProfile ...`，select.ts:88）+ Windows git-bash 重定向 + login 参数决策；它同时被 pty.ts、persistent-pty/index.ts:166（daemon 默认程序）、shell 工具复用。

## 7. 不变量与设计取舍（从源码注释与结构提取）

- **Pty 服务 transport-free**：protocol.ts:4-5——"PTY 域服务不含传输层；server 路由用这些 helper 把 Pty.attach 适配到 websocket，使所有面对外一种协议"。
- **两段式 attach**（activate 前 pending 不丢）：保证 replay→cursor→live 流的顺序不被竞态打断（pty.ts:260-307 结构）。
- **有界保留**：2MB 输出缓冲、25 个退出会话、64KB replay 帧——处处封顶，废弃终端不泄漏。
- **票据必须原子核销**：`Cache.invalidateWhen` 而非 get+remove；且 `noLookup` 故意 die 以防误用（ticket.ts:26-28）。
- **Job 是进程本地的，durable 只覆盖"通知义务"**：start 注释（job.ts:172-177）明确了什么落盘（recovery+notificationID）什么不落（前台结果、fiber 本身）。崩溃后**不恢复执行**，只恢复**通知**——shell 进程死了就补 cancelled 通知，subagent 才有 resume。
- **消费历史按首次消费排序**（consume，job.ts:185 注释），不是 start 时间也不是后续读取时间。
- **重启 handoff 永不让位给启动**：终端 best-effort，任何失败 abandon 旧 daemon 换新目录，restart 本身必成（daemon.ts:413-414, 418-430）。
- **rename 保 inode 的迁移**：旧版本 daemon 目录搬家后锁仍有效（daemon.ts:433-434 注释）——对"文件锁跟着 inode 走"的精确利用。
- **opaque ≠ deny**：扫描器承认解析边界，把不可分析的命令如实上报为 opaque，让权限层（而非解析器）做决定（scan.ts:14）。
- **持久终端的 ID 信任边界**：`fromID` 严格校验 `pty_persistent_<正整数>`，不合即 throw（index.ts:414-419）——daemon 内部序号不被伪造。

## 8. 失败与恢复

**PTY 进程死**：
- **进程内档**：onExit → status=exited、广播 `Pty.Event.Exited`、进 exitOrder LRU；attach 对 exited 会话抛 `ExitedError`（pty.ts:261-262）。server 死 = 全部 teardown（kill running 子进程，pty.ts:131-148）。

  环形缓冲的裁剪逻辑（onData 尾部，pty.ts:216-222）：
  ```
  $ sed -n 216,223p packages/core/src/pty.ts
        session.buffer += chunk
        if (session.buffer.length <= BUFFER_LIMIT) return
        const excess = session.buffer.length - BUFFER_LIMIT
        session.buffer = session.buffer.slice(excess)
        session.bufferCursor += excess
  ```
  `bufferCursor` 记录"buffer[0] 对应的绝对输出偏移"——attach 的 replay 切片（pty.ts:288-295）靠 `from - start` 换算，请求早于 start 的部分自然拿不到（返回空串），客户端应感知截断。

- 持久档：daemon 侧 `exited` 事件带 exit_code + finalOffset；index.ts 收到后 `removeVisibleExit` 自动清终端。**daemon 死/失联**：request 的 connect 错误降级为 `UnavailableError`；list 走 `requestIfRunning`——daemon 不在就返回空列表而不是失败（index.ts:141-146）；需要时会 spawn 新 daemon（fresh 目录），旧终端不可寻回。
- 协议不匹配（升级后 daemon 旧版）：discover 直接 fail protocol 错误 → start 重试新目录。

**Job 恢复矩阵**（restart.ts boot 扫描，shell 先 subagent 后）：

| recovery 分支 | running（进程死时） | completed/error | cancelled |
|---|---|---|---|
| **shell** | 改判 **cancelled** 补通知（进程输出已失，不可能续）restart.ts:103-110 | 原状态补通知（output/error 原样）restart.ts:111-117 | 通知 "Command cancelled" |
| **subagent** | child 会话是 durable 的：父/子会话仍存在且子不活跃 → `jobs.start` 重跑 `execution.resume(child)` 再 background + wait→notify（restart.ts:160-190）；预算超限（默认 10 次，restart.ts:27）→ error 通知；孤儿（父或子没了）→ 仅清标记 restart.ts:144-147 | 补发完成通知（可按 suspended 决定 resume:false） | 同左 |

共性收尾：每条恢复都以 `completeBackground(notificationID)` 清 KV 标记结束；恢复通知 `resume:false` 防止长命 shell 的空闲 owner 被误唤醒（restart.ts:130-131）。

## 演进注记

- **persistent-pty 是新增子系统**：`git log --diff-filter=A` 显示目录由 `bc1f67e "feat(cli): embed persistent PTY service binaries (#44970)"` 引入（含 `@opencode-ai/pty` 0.2.0 二进制嵌入，package.json:109）。旧笔记 4.16 时代的"pty"只有进程内 `packages/core/src/pty.ts` + server 侧 shell 工具那一档；如今的进程内档反而只剩终端面板用途，**shell 工具的命令执行走 Shell 服务（capture），不经 Pty**——旧笔记把两者混在"pty + shell tool"一篇的框架已不成立。
- `pty.ts` 本体近年经历 `expose pty layer directly (#45733)` 与 `drop forwarding shim modules (#51670)` 两次瘦身，平台后端从单文件拆成 `pty/` 目录的 bun/node/workerd 三份 + protocol/ticket。
- Job 的 Background schema 把 shell/subagent 两种 recovery 显式编码进同一 Union（job.ts:14-27），subagent-job 逻辑独立成 `session/subagent-job.ts`——注册表本体保持领域无关。

## 相关篇

- [4_session.md](4_session.md)：shell 工具所处的 session 执行链与 synthetic 通知
- [12_subagent.md](12_subagent.md) §3：Job 的 subagent 侧应用（ID 复用、notify 去重、block vs backgrounded）
- [11_permission.md](11_permission.md)：`permission.assert` 如何消费 §6 产出的 resource/save
- [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)：shell turn 的 TUI 侧等待/打断/1500ms 宽限
- [../2_tech-stack/2.2_server/1_js-runtime.md](../2_tech-stack/2.2_server/1_js-runtime.md)：bun/node/workerd 三运行时与 node-pty 生态背景
