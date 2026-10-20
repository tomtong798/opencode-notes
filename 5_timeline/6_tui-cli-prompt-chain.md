# TUI / CLI 提问链：终端侧的提交与消费

> 一句话：**TUI——footer 回车 → `runPromptQueue`（串行队列 + active 期间直发 durable 准入）→ `admitPromptTurn`/`runPromptTurn`（`switchAgent`/`switchModel` → `client.session.prompt`）→ SSE `apply` reducer 投影到 footer commits → `session.wait` idle fence 收 turn；CLI——`runNonInteractivePrompt`（`delivery:"steer"` + `consume()` 事件循环 + 权限/表单自动拒绝 → `session.wait` → `reconcile()` 投影对账补漏）。**
> 代码基线：v2.0.30，`dev` 分支；行号实测（2026-10-19）。
> [返回 README](README.md) | 上一篇 [5_prompt-chain.md](5_prompt-chain.md)（Desktop/Web 链 + 服务端执行/事件回程，本篇不重复）

---

## 0. 三端汇聚图

三条前端路径，同一个后端契约（`session.prompt` 准入 + SSE 事件流 + idle 判定）：

```
Desktop/Web            TUI                      CLI（非交互）
submit.ts submit()     footer.onPrompt          opencode run "…"
sendPrompt             runPromptQueue           runNonInteractivePrompt
data.session.prompt    admitPromptTurn /        client.session.prompt
(乐观 admitLocal)      runPromptTurn            (delivery:"steer")
  │                    (switchAgent/Model→      │
  │                     client.session.prompt)  ├─ consume(): SSE 事件循环
  │                      │                      │  （权限/表单自动拒绝）
  ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────┐
│ POST session.prompt（客户端铸 message id + delivery）      │
│ 服务端：reconcile 幂等 → prepare → admit → wake（秒回）    │
│（详见 5_prompt-chain.md §2-§5）                            │
└─────────────────────────────────────────────────────────┘
  │                      │                      │
  ▼ SSE 事件流            ▼ SSE 事件流            ▼ SSE 事件流
store.produce 投影       apply() reducer →      consume() → stdout /
Solid 细粒度渲染          footer commits          json-lines
  │                      │                      │
  ▼                      ▼                      ▼
执行事件流                session.wait           session.wait
（事件驱动）              （idle fence，权威）    + reconcile() 投影对账
```

三端共同点（与 Desktop/Web 篇一致，不重复展开）：

- **客户端铸 id**：三端都在提交前 `SessionMessage.ID.create()`（TUI 队列 `runtime.queue.ts:151/266`；CLI `noninteractive.ts:82`；Desktop `composer/submit.ts`）。这是服务端幂等 reconcile 的前提。
- **准入/执行分离**：`session.prompt` HTTP 秒回 admitted；执行是异步 drain。终端两端的差别只在"怎么等执行结束、怎么把事件变成字符"。
- **idle 判定不再靠轮询**：两端都用 `session.wait`（`core/src/session/session.ts:265-268`——`get` + `execution.awaitIdle`）作为权威 idle fence；SSE 生命周事件只是展示性元数据（TUI 代码注释原话：*Live lifecycle events remain presentation and best-effort outcome metadata*，`stream-v2.transport.ts:1598-1599`）。旧版 TUI 的 250ms 轮询兜底已删除。

---## 1. TUI：串行队列（runtime.queue.ts）

**`packages/tui/src/mini/runtime.queue.ts`** —— `runPromptQueue`（L46）的队列状态机：

```
footer.onPrompt(prompt) → submit(prompt)（L243）
│
├─ 空文本且无 file part / 已关闭 → 忽略（L244-249）
├─ /exit /quit → footer.close()（L251-254）
│
├─ ★ active 普通 turn 进行中（L256-276）
│   ├ 条件：active 非 shell、prompt 非 shell、
│   │       非技能命令、非 /new、非 /compact、
│   │       队列里也无 /new、/compact
│   ├ 铸 messageID（L266）
│   ├ delivery = prompt.delivery ?? "queue"（L269）
│   └ 串入 admissions 链：
│       等当前 turn admission → input.admit(sent, delivery)
│       ★ 不进本地队列——直接发服务端 durable 队列
│       失败 → onAdmissionError（footer 错误行）
│
└─ 否则 push 本地 queue → drain()（L278-293）
```

```
drain()（L85）—— 串行排空
│
└─ while queue 非空且未关闭（L92）
    ├ /new → onNewSession()（L95-126）
    │       （重建 session/stream，footer 复位）
    ├ /compact → sdk.session.compact（L128-144）
    │
    └ 普通 prompt（L146-223）：
        ├ 铸 messageID（L151）
        ├ footer.idle() 等渲染空闲（L170）
        ├ input.run(sent, signal, admission.resolve)
        │ （L181——runner 回调 admitted() 的时机点）
        ├ Promise.race([task, stop.promise])（L186）
        │   关闭 → break
        └ ★ turn 结束后 admission 版本变了（L194-202）
            （说明 turn 期间有新输入 durable 准入）
            → do-while：等 admissions 链排空 → input.settle()
              （session.wait 收尾）直到版本稳定
finally（L203-223）
    ├ admission.resolve()（确保 runner 不被卡）
    └ turn.duration → footer 状态行（L210-221）
```

**与旧笔记（2026-08-03）的语义差异**：旧版"active 期间提交 → 本地 queued，可编辑/移除"；现版改为**直接 `input.admit()` 发服务端 durable 队列**（`delivery:"queue"`），本地不再保留可编辑副本——服务端 inbox 就是队列的事实源。旧文档此节已失效。

**接线**（`runtime.ts:891-1052`）：`admit` = `ensureStream()` 后 `handle.admitPromptTurn(...)`；`run` = `handle.runPromptTurn(...)`；`settle` = `handle.waitForIdle()`；`onNewSession` = 换 session 重建 stream 并复位全部本地状态。

## 2. TUI：单 turn 提交（stream-v2.transport.ts）

**`admitPromptTurn`（`stream-v2.transport.ts:1794-1809`）** —— 仅准入（不等待执行）：

```
admitPromptTurn(next, delivery)
│
├─ shell 模式 → throw "This prompt cannot be queued"（L1795）
├─ waitUntilConnected（L1796——25ms 轮询 state.connected）
├─ 非 command 且有 agent → session.switchAgent（L1798-1799）
├─ 非 command：resolveSelectedModel（L1801）
│   ├ 有 variant 无 model → throw（L1802）
│   └ 有选中 → session.switchModel（L1803-1804）
├─ admitPrompt(next, client, delivery)（L1806 → L1730-1766）
│   ├ prepareAttachments（文件 → files/text 附件）
│   ├ promptAgents / promptSkills（@agent / 技能）
│   ├ ★ 普通 prompt → client.session.prompt(
│   │     {sessionID, id: messageID, text, files,
│   │      agents, skills, delivery})（L1739-1750）
│   └ 命令 → client.session.command（L1754-1765）
└─ mergePending(admitted)（L1807——挂进待提升 pending 表）
```

**`runPromptTurn`（L1815-1848）** —— 完整 turn（准入 + 等 idle）：

```
runPromptTurn(next, admitted?)
│
├─ shell → runShellTurn（L1816-1819，见 §4）
├─ state.wait / shellWait 占用 → throw "prompt already running"
│  （L1820——单 turn 互斥）
├─ command → admitPrompt(delivery ?? "steer") + admitted?.() 即返回
│  （L1826-1831——命令不等待执行）
├─ switchAgent / resolveSelectedModel + switchModel（L1833-1839）
└─ runTurnWait(...)（L1841）
    │
    └ runTurnWait（L1600-1643）
        ├ state.wait = active（L1615）
        ├ signal.abort → session.interrupt(resume:true)（L1616-1620）
        ├ send() = admitPrompt(...)（L1622）
        │   └ admitted → mergePending + onAdmitted?.()
        ├ ★ settleSession(client)（L1625）
        │   （session.wait + projectedMessages 补画 + paintIdle）
        └ 终端错误未渲染 → footer 错误行（L1626-1635）
```

**`settleSession`（L909-914）**：`client.session.wait(...)` 等服务端权威 idle → `projectedMessages`（L901，`session.messages` 投影接口倒序拉取）逐条 `renderMessage(message, true)` 补画 → `paintIdle(blockerStatus(view))` → `footer.idle()`。**turn 的完成判定以 `session.wait` 为准，事件流只负责增量**。

## 3. TUI：事件消费（connect/apply）

**`connect`（`stream-v2.transport.ts:1478-1545`）** —— SSE 连接与重连循环：

```
connect()（外层 while，未 abort 且 footer 未关）
│
├─ event.subscribe(...) 异步迭代器（L1486）
├─ 首事件必须 server.connected，否则断线（L1489-1490）
├─ booting 期间事件先 buffer（L1491-1499）
│   ★ 与 hydrate() 竞速（Promise.race，L1501-1509）
├─ hydrate 成功 → 回放 buffered（L1512）→ booting=false
│   → state.connected=true → readyResolve()（L1514-1516）
├─ await consume（正常事件循环）（L1518）
└─ 断线（catch）：
    ├ 状态行 "reconnecting"（L1531）
    ├ input.reconnect() 重新解析 SDK 端点（L1532-1542）
    └ wait(250ms) 重试（L1543）
```

**`hydrate`（L952-1004）** —— 六路并发投影（连接/重连/resize 重放共用）：

```
Promise.all（L955-969）
├ projectedMessages     ← session.messages 投影（倒序拉全量）
├ session.inbox.list    ← pending 输入（→ state.pending → syncPending）
├ permission.list       ← 未决权限
├ session.form.list     ← 未决表单
├ form.list(location)   ← 全局（global 哨兵）表单
└ session.active        ← 活跃 drain 会话集合
→ renderMessage 逐条补画（L980）
→ resolvePermissionSources（L981——权限源工具详情回查）
→ subagents.hydrate（L990——子代理会话）
→ phase: rootActive ? running : idle（L998-1001）
```

**`apply`（L1006 起）** —— SSE 事件 reducer（节选主线）：

| 事件 | 动作 | 行号 |
|---|---|---|
| `session.inbox.enqueued`（user 项） | `mergePending` → 队列 UI | L1040-1049 |
| `session.inbox.delivered` | `promoteWait` + 删 pending + `renderUser` | L1050-1060 |
| `session.inbox.delivery.changed` | 更新 pending 投递模式 | L1061-1067 |
| `session.inbox.cancelled` | 删 pending | L1068-1072 |
| `session.step.started` | 记 stepModel，状态 "assistant responding" | L1073-1077 |
| `session.skill.activated` | 技能行（可 `promoteWait`） | L1078-1085 |
| `session.compaction.*` | 压缩进度/摘要/失败行 | L1086-1124 |
| `session.shell.started/ended` | shell 行 + `shellWait.resolve()` | L1125-1165 |
| `session.text.delta/ended` | `fragments` 对账后增量 `write`（`showTools()` 关时缓存静默文本） | L1166-1209 |
| `session.reasoning.delta/ended` | 同上，`Thinking:` 前缀，受 `input.thinking` 开关 | L1210-1249 |
| `session.tool.input.*` | 工具卡片 streaming 态 | L1250-1272 |
| `session.tool.called/progress` | running 态 + 元数据 | L1273-1308 |
| `session.tool.success/failed` | 终态卡片 | L1309-1360 |
| `session.step.ended/failed` | usage 补状态行 / 错误行 | L1361-1400 |
| `session.execution.started` | phase running | L1401-1407 |
| `session.execution.succeeded/failed/interrupted` | **不直接判 idle**——等 `settleSession` 的 `session.wait`（见下） | L1408-1438 |

**`fragments` 对账器**（`createFragmentReconciler`，L494 实例化）：`text.delta`/`text.ended` 都先过 `fragments.delta/end` 判重——**以事件 idempotency + 投影文本为对账基准**，重复 delta（重连回放、resize 重放）自动去重，`restore` 支持本地行文本与重放流的 diff 合并（`performResizeReplay` L1645-1723 使用）。

## 4. TUI：shell turn 与权限/表单 blocker

**`runShellTurn`（L1561-1596）**：

```
runShellTurn(next)
│
├─ 互斥检查（L1562）→ waitUntilConnected（L1563）
├─ 铸 messageID + eventID（L1572-1573）
├─ state.shellWait = {resolve, abort}（L1574-1580）
├─ client.session.shell({sessionID, id, command})（L1584-1587）
│   ★ HTTP 同步请求——shell 不走 inbox 准入，服务端自持生命周期
└─ Promise.race([output 渲染完成, SHELL_OUTPUT_GRACE_MS=1500ms])
    （L1588——宽限后放弃等待输出）
```

**中断语义**（`interruptActiveTurn` L1849-1860）：shell turn 在跑 → `abort()` 阻塞请求（服务端命令保留自己的生命周期，只是失去等待者）；普通 turn → `session.interrupt`。

**blocker**（`syncBlockers` L642-668）：权限/表单/全局表单 → `pickBlockerView` 切 footer 视图（权限选择器/表单填写器）；`permission.asked`/`form.created` 事件与 `hydrate` 双通道都会汇入。

## 5. CLI：非交互单发（noninteractive.ts）

**入口**：`packages/cli/src/run/run.ts:137` → `runNonInteractivePrompt`（`noninteractive.ts:76`）。

**准备段**（`run.ts:71-154`）：

```
run(input, options)
│
├─ mergeInput：命令行消息 + piped stdin 合并（L76）
├─ prepareFile ×N：≤10MB、mime 探测、
│   text → <file> 块、二进制 → data:base64（L78, L185-218）
└─ execute（L83）
    ├ OpenCode.make（fetch timeout:false——养长连接 SSE，L88-89）
    ├ resolveSessionTarget（continue/session/fork/model/agent）
    │   （L92-126）
    └ runNonInteractivePrompt(...)（L137）
```

**`runNonInteractivePrompt`（`noninteractive.ts:76-789`）**：

```
准备
├─ event.subscribe 先建流（L78-80——★先订阅后提交，防丢事件）
├─ 铸 messageID（L82）
├─ switchAgent / switchModel（L690-709）
├─ prepareFile(text/plain → <file> 块内联)（L711）
└─ consume() 先启动（L714——消费循环与提交并行）
提交
├─ client.session.prompt({..., delivery:"steer"})(L716-726)
│   失败且 interrupted → 补 interrupt（L727-735）
└─ 提交后即补处理积压 blocker（L740-757）：
    permission.list / form.list / 全局 form.list
    → 逐个 replyPermission / cancelForm
等待与对账
├─ session.wait（L763）
├─ Promise.race([waiting, completed.then(()=>waiting)])（L764）
│   （completed=事件循环；防流断后 wait 悬挂）
├─ finalizing=true（L765）
└─ ★ reconcile()（L766 → L568-677）
    ├ message.list 投影分页拉取（L550-566）
    ├ 已渲染前缀比对：item.text.startsWith(rendered)
    │   → 只补 suffix（防重复输出，L580-593）
    ├ 未渲染工具 → 补画（L617-663）
    └ 未提升（!found）→ 报 "Prompt was not promoted"
      （prePromotionError 优先，L777-782）
```

**`consume()` 事件循环（L209-548）** 的三个非显示性要点：

- **会话所有权**（L143-156）：`ownsSession` 沿 `parentID` 链向上判归属——子代理会话的权限/表单也归本 run；共享服务器上**其他会话（如 TUI 的）不动**。
- **权限自动应答**（L158-186）：非 `--auto` → `decision:"reject"` + 反馈消息（工具以普通失败落进模型可见输出，模型继续）；`--auto` → `decision:"once"` 放行一次。v1 兼容模式下 reject 即中断。
- **表单自动取消**（L188-207）：question 类 → cancel + 反馈消息继续；其他 → cancel 且 `exitCode=1`。

**退出码**：`session.execution.interrupted` + user → 130（L534-536）；failed / step.failed 未被吸收 → 1；正常 `succeeded` → return（L546）。

## 6. 三端对比（速查）

| 维度 | Desktop/Web | TUI（mini） | CLI（run） |
|---|---|---|---|
| 提交入口 | `composer/submit.ts` `submit()` | `runPromptQueue` → `admitPromptTurn`/`runPromptTurn` | `runNonInteractivePrompt` |
| 铸 id | ✓（admission.id） | ✓（队列/turn 两处） | ✓（L82） |
| 默认 delivery | steer | turn 内 `?? "steer"`；active 期间补发 `"queue"` | `"steer"`（L723） |
| 会话切换 | UI 先建/选会话 | `switchAgent`/`switchModel` 每次同步 | `resolveSessionTarget` + `switch*` |
| 幂等 | outbox + 服务端 reconcile | 服务端 reconcile（pending 表镜像） | 服务端 reconcile + `reconcile()` 投影对账 |
| 事件消费 | store.produce（Solid） | `apply` reducer → footer commits | `consume()` → stdout / json-lines |
| 去重 | store 按 id 写入 | `fragments` 对账器 + 各 id 集合 | `renderedText`/`renderedTools` 前缀比对 |
| idle 判定 | 执行事件流 | ★`session.wait`（idle fence）+ `settleSession` 补画 | ★`session.wait` + `reconcile()` |
| 权限/表单 | 弹窗 | footer blocker 视图 | 自动 reject/cancel（`--auto` 则放行一次） |
| 中断 | `session.interrupt` | `interruptActiveTurn`（shell 特殊：abort 请求） | SIGINT → interrupt + exit 130 |
| 附件 | UI 附件管线（20MB/8 并发物化） | `prepareAttachments`（路径 → files） | `prepareFile`（10MB、text 内联 `<file>` 块） |

## 7. 与旧笔记（gitee 2026-08-03）的差异清单

供从旧笔记迁移阅读时校准，旧版描述已失效处：

| 旧笔记说法 | 现状（2026-10-19 实测） |
|---|---|
| TUI 代码在 `packages/opencode/src/cli/cmd/run/` | 已迁至 `packages/tui/src/mini/`（队列 `runtime.queue.ts`、传输 `stream-v2.transport.ts`） |
| TUI 用 `prompt_async` 端点异步提交 | 端点已删除；统一 `client.session.prompt`（含 `delivery` 参数） |
| TUI active 期间提交 → 本地 queued 可编辑 | 直发服务端 durable 准入（`input.admit`，delivery `"queue"`）；本地 `pending` 表仅是镜像 |
| TUI 250ms 轮询 `session.status` 兜底 | 已删除；权威 idle = `session.wait`（`session.ts:265`） |
| `runPromptTurn` 三种发送（shell/命令/普通） | shell 拆出 `runShellTurn`（HTTP 同步 + 1500ms 输出宽限）；准入拆出 `admitPromptTurn` |
| CLI `sdk.session.prompt` + 事件→stdout | 语义未变，实现大改：`session.wait` + `reconcile()` 投影对账、权限/表单自动应答、子代理会话所有权判定 |
| `stream.transport.ts` | 现名 `stream-v2.transport.ts`（旧文件名已不存在） |

## 相关篇

- [5_prompt-chain.md](5_prompt-chain.md)——Desktop/Web 提交链 + 服务端准入/执行/事件回程全链（本篇的服务端半部与回程基础设施）
- [4_renderer-boot.md](4_renderer-boot.md)——Desktop/Web 的 store 与 SSE 连接层
- [2_server-init.md](2_server-init.md) / [3_location-lazy.md](3_location-lazy.md)——服务端前置
- `specs/v2/session.md`——投递词汇（steer/queue）、准入语义的权威出处
