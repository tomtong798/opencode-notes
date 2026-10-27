# 性能预算（Performance）

> 一句话：**opencode 的性能观是"预算长在结构上"——启动把窗口排在一切之前（让行编排 + 单 chunk + 双侧字节码缓存）、渲染靠 Solid 细粒度信号只重算变化的部分、传输三级批处理（100ms delta → 10ms flush → Solid batch）摊平事件风暴、内存用"永不过期的 Location 图 + 有界 LRU"换稳定、包体积靠"渲染进程不背 Effect runtime"砍掉三分之一启动脚本——每条机制都是对某个实测恐惧的回答。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，直读 + sed 抽查 ≥15 处）。
> 权威出处：`packages/desktop/electron.vite.config.ts`（chunk 策略注释）、`packages/desktop/src/main/index.ts`（让行编排）、`packages/core/src/session/runner/publish-llm-event.ts`（delta 批）、`packages/client/src/solid/connection.ts`（flush 批）。
> [返回 README](README.md) | 关联 [7_observability.md](7_observability.md) §5（marks/bench-startup）/ [5_resource-limits.md](5_resource-limits.md)（限额全景与背压）/ [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md)（启动时序）/ [../2_tech-stack/2.1_client/2.1_solidjs-reactivity.md](../2_tech-stack/2.1_client/2.1_solidjs-reactivity.md)（细粒度渲染）
> 本篇是**性能视角的收口**：启动时序细节在 5_timeline/1，观测在 7_observability §5，限额全景在 5_resource-limits——这里回答"这些机制共同买到了什么预算"。

---

## 0. 恐惧清单（性能上怕什么）

从代码注释与机制反推，desktop/TUI 双端各有一组具体恐惧：

```
性能恐惧（每条对应本篇一节）
├─ 恐惧 1：启动慢——用户看到白屏/无窗口的每一百毫秒都在流失
│    └─ 防线：窗口先行 + 让行编排 + 单 initial chunk + V8 字节码双缓存（§1）
├─ 恐惧 2：长列表卡死——几千条消息的 timeline 全量挂载，首屏秒级冻结
│    └─ 防线：TanStack 虚拟化 + 冷启动尾部优先挂载 + 延迟渲染重内容（§2.2）
├─ 恐惧 3：高频 delta 撕裂渲染——LLM 流式输出每个 token 一条事件，
│    每条事件一轮通知传播 → UI 在 token 粒度上反复重渲染
│    └─ 防线：三级批处理链 100ms→10ms→batch（§3）
├─ 恐惧 4：事件风暴打满主线程——启动期配置/插件/MCP 事件齐发，
│    渲染进程协议处理器与主进程模块求值抢同一条线程
│    └─ 防线：rendererAssetsServed 让行 + 事件落 store 的批合并（§1.5/§3）
├─ 恐惧 5：包体积拖慢首屏——自动 chunk 拆分把初始图打成 ~50 个文件，
│    每个文件一次经主进程的主线程请求往返
│    └─ 防线：initial chunk 不拆分 + 懒加载清单 + renderer 不背 Effect（§4）
└─ 恐惧 6：内存无界增长——消息/缓存/Location 图任一无限膨胀拖垮长会话
     └─ 防线：有界 LRU（500 条/8MiB）+ Location 图"永不过期"的显式取舍（§5）
```

---

## 1. 启动性能机制清单（desktop）

启动的全时序走读见 [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5/§2.1，启动计时与 bench-startup 的 17 段阶段拆分见 [7_observability.md](7_observability.md) §5。本节只收口：**每条机制解决什么测量出的什么问题**。

### 1.0 机制地图

```
packages/desktop/
├─ electron.vite.config.ts ........ 149 行 initialChunk 插件：初始图单 chunk（§1.4）
├─ scripts/bench-startup.ts ....... 17 阶段冷启动基准（§1.7）
└─ src/main/
   ├─ index.ts ....................  35 行入口：窗口先行 + 让行编排（§1.1/§1.5）
   ├─ lifecycle/configure.ts ......  enableCompileCache：主 bundle 字节码缓存（§1.2）
   ├─ lifecycle/marks.ts ..........  11 个启动计时点（§1.7）
   ├─ windows/scheme.ts ...........  oc:// privileges：codeCache:true（§1.3）
   ├─ windows/protocol.ts .........  rendererAssetsServed 让行（§1.5）
   └─ desktop.ts ..................  主 bundle 入口：500ms 延迟装非关键件（§1.6）
```

### 1.1 窗口先行（index.ts:18-35）

主入口 `desktop/src/main/index.ts` 全文 35 行，注释就是设计宣言（原文节选）：

> "Electron holds the ready event until the entry module has finished, and the first window should be on screen before the rest of the main process — a few hundred milliseconds of module evaluation and layers — loads."

执行序（`index.ts:26-34`）：`whenReady` → `marks.ready` → **`createEarlyWindow()`（先出窗）** → `marks.window` → `rendererAssetsServed` 让行 → `import("./desktop")`（后进主包）。解决的问题是 bench-startup 里被切成独立阶段的 "ready → window shown"——窗口可见被钉死在所有重活之前，剩余阶段（bundle/logging/storage/layers）全部与渲染进程加载**并行**。落到阶段图上：

```
旧世界观（顺序）：ready → 主包求值(百ms级) → layers → … → 窗口
新世界观（并行）：ready → 窗口（立刻）──┐
                        主包求值 ← 让行(≤400ms) ← 渲染资产请求突发
                        layers/logging/storage 与渲染进程加载并行推进
bench-startup 阶段：window→served / served→bundle / … / shell→idle 各自计中位数
```

### 1.2 主进程 compile cache（lifecycle/configure.ts:42）

```typescript
// V8 bytecode for the main bundle survives between launches, like the renderer's code cache.
enableCompileCache(path.join(app.getPath("userData"), "compile-cache"))
```

`node:module` 的 compile cache 让主 bundle 的 V8 字节码跨启动存活——对应 bench-startup 阶段 "main bundle load + evaluate"。冷启动（`--fresh` 模式）与热启动的差值里，这段就是被缓存吃掉的部分。

### 1.3 oc:// 协议 codeCache（windows/scheme.ts:45-53）

自定义协议注册 privileges 时显式 `codeCache: true`（`scheme.ts:51`），注释原文："Let Chromium keep V8 bytecode for the renderer bundle between launches."——渲染进程侧的同款优化：**主进程 enableCompileCache + 渲染进程 oc:// codeCache，双侧字节码缓存**。没有这层，每次启动渲染 bundle 都要重新 parse/compile 全量 JS。

### 1.4 初始 chunk 不拆分（electron.vite.config.ts:35-51）

vite 配置里的自定义 `initialChunk` 插件，注释把动机说得极明白（`:35-38` 原文）：

> "Every module the entry reaches through static imports lands in one chunk. Automatic splitting otherwise fragments the initial graph into ~50 files shared with lazy routes, and each file costs the renderer a main-thread request round trip through the main process before first paint."

机制：`renderStart` 从入口沿静态 import 闭包收集 `initialGraph`（`:48-49` 利用 Set 迭代器"边迭代边加成员"的语义走全图），输出时 `codeSplitting.groups` 把 initial 图内的模块强制归入名为 `app` 的单 chunk、priority 10（`:141-143`）。**测量出的问题**：oc:// 协议处理器跑在主进程主线程上（见 §1.5），每个 chunk 文件 = 渲染进程经主进程的一次串行请求往返——50 个文件就是首屏前 50 次往返。收口成 1 个，往返数坍缩。

### 1.5 rendererAssetsServed 让行（windows/protocol.ts:26-51 + index.ts:30）

窗口先出、但主 bundle 的 `import("./desktop")` 不能立刻求值——因为**渲染进程的 HTML/JS 是同一个主线程在服务的**。`protocol.ts:26-29` 注释原文：

> "The entry module holds the main bundle back until the renderer's initial burst of asset requests has been answered, because this handler runs on the main thread and a 100 ms module evaluation would otherwise sit between the renderer and its HTML."

`rendererAssetsServed({ quietMs: 40, capMs: 400 })`（`index.ts:30`）：轮询 inflight 计数与 lastRequest，请求静默 40ms 或总计到 400ms 封顶才放行（`protocol.ts:36-51`）。这是**恐惧 4（事件风暴打满主线程）的启动期形态**：两件事抢一条线程，用"让行 + 有界等待"而不是队列排序解决。完整走读见 [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5.4。

### 1.6 配套让行：500ms 延迟装的非关键件

desktop.ts（主 bundle 入口）里还有一个启动后段的让行（`desktop.ts:44-48`）：

```typescript
// The first window's renderer now has its IPC port and is hydrating its stores over it. The crash
// reporter (spawns a process) and the context menu (a dependency tree) are not worth answering late.
yield* Effect.sleep("500 millis")
const logging = yield* DesktopLogging.Service
yield* logging.startCrashReporter
yield* installContextMenu
```

崩溃报告器（要 spawn 进程）与右键菜单（一整棵依赖树，动态 import `electron-context-menu`，`lifecycle/environment.ts:13`）被显式推迟到渲染进程 hydrate 完 store 之后——同一让行哲学的启动尾段形态：**任何"不值得让首屏等"的模块，排在首屏之后**。GUI 扩展同理（`desktop.ts:34-36` 注释）：主扩展随窗口启动（喂首屏的 server 列表和菜单），但"their heavy modules load lazily on first use"。

### 1.7 启动收口：marks 与 bench

`lifecycle/marks.ts:5-7` 定义 11 个计时点（entry/ready/window/served/bundle/onboarding/logging/crash/storage/init/layers），`scripts/bench-startup.ts:224-241` 的 `phaseOrder` 把启动切成 **17 个中位数阶段**（electron native init → … → shell → idle），里程碑来自主进程日志、渲染进程 performance timeline、CDP DOM 轮询、以及**屏幕像素采样**（"What is on screen is sampled from the screen itself"——因为"渲染进程报告 painted"不等于"用户看到了"）。这套测量是 §1 全部机制的存在前提：先量、后优化、每次改动跑 `--compare` 双构建交替对拍。

---

## 2. 渲染预算

### 2.1 细粒度响应式的性能含义

机制全貌见 [../2_tech-stack/2.1_client/2.1_solidjs-reactivity.md](../2_tech-stack/2.1_client/2.1_solidjs-reactivity.md)。性能视角只收一句：**组件函数只跑一次，更新发生在信号级**——数据变化时重跑的是订阅该信号的局部表达式，不是组件树。推论：

- 没有 vdom diff 这一层预算开销，更新成本与"变化了多少信号"成正比，与"树有多大"无关；
- `<For>` keyed 列表里引用未变的项零开销（2.1 篇 §4"key 的意义就是细粒度更新"）——这正是长列表能扛住的前提，虚拟化负责不挂载，细粒度负责挂载的部分更新便宜；
- 组件体里写初始化逻辑天然只执行一次（2.1 篇 §5"组件只跑一次的推论"），不需要 memo 化防御；
- produce/setStore 的路径级写入让"改一条消息的一个字段"只通知订阅该路径的表达式——timeline 流式更新走的就是这条路（结合 §3 的批 3，一批事件 = 至多一批路径级信号触发 = 一次合并渲染）。

### 2.2 timeline 长列表虚拟化（app/src/session/timeline/）

虚拟化基于 `@tanstack/solid-virtual`（`virtualizer.tsx:8`），但 opencode 在其上叠了三层**实测出的挂载策略**：

**① 冷启动尾部优先（coldBottomMount/initialTail）**。会话历史很长时，首屏用户只看底部。`virtualizer.tsx:102`：`const coldBottomMount = !initialMeasurements?.length && input.pinned()`——无缓存测量且钉在底部时进入 initialTail 模式；`rangeExtractor`（`:143-177`）在该模式下从底部截一段"有界廉价后缀"先挂载，注释原文："A large tail still mounts alone before estimates expose earlier history"，重内容（未知高度的工具调用）之前的边界由 `canRenderImmediately` 判定。早前历史等测量估算就位后再展开——**首屏挂载量与历史总长解耦**。冷尾模式的行进：

```
initialTail = true（无缓存测量 + pinned）
├─ rangeExtractor：从 range.startIndex-2 向尾部扫，遇到第一个"不可立即渲染"行
│   （大内容/未知高度，canRenderImmediately 判定）即收边界
│   → 只挂 [first, range.count) 的尾部窗口 + active/anchored 行
├─ 流式期间：pinColdBottom 持续钉底（直写高度 + scrollToEnd）
├─ settleColdBottom（microtask）：等 pendingMarkdown/待测元素清零
│   → 逐项 resizeItem 补测 → 条件齐备后 setRendering("initialTail", false)
└─ 退出冷尾 → 恢复常规 overscan:2 的 defaultRangeExtractor
```

**② deferred 行延迟渲染**。哪些行算"重"由路由显式声明（`message-timeline.tsx:449-455`）：edit/write 工具调用的结果内容 deferred——首挂载先出廉价骨架，`VirtualRow` 里 `ready` 信号（`virtualizer.tsx:778`）初始为 `size <= fallbackItemSize || !deferred`，就位后再揭示真内容。

**③ sticky scroll（pinColdBottom）**。流式输出期间钉底不抖：`virtualizer.tsx:541-557` 的 `pinColdBottom` 直接写 `virtualContent.style.height` 后 `scrollToEnd`，注释点明"scrollToEnd computes its target from the DOM, not the new size cache"——绕过尺寸缓存竞态；`settleColdBottom`（`:564-600`）在 microtask 里等 markdown/pending 测量全部落定才退出 initialTail 模式，防止流式会话首屏跳动。另有按会话键缓存的测量缓存（`:97-100` cache entry），tab 切回时免重测。

### 2.3 按 session 键的测量缓存（tab 切回免重测）

虚拟化器还有一层跨挂载的测量缓存（`virtualizer.tsx:97-100`）：以 `ownerSessionKey` 取 cache entry，`presentationKey`（呈现模式）一致则复用 `measurements`/`toolOpen`/`patchGroupKeys`。tab 切走再切回时行高不需要 ResizeObserver 重测，`coldBottomMount` 判定（`:102` `!initialMeasurements?.length`）也因缓存存在而走常驻路径。配套细节：`observe-element-offset.ts` 与 `:246` 注释（引用 TanStack/virtual#1183）处理"deferred 内容后到导致高度漂移"的测量坑——**测量缓存省的是切 tab 的重复布局，虚拟化省的是挂载，两层各管一段**。

### 2.5 TUI 60FPS 预算

TUI 侧的帧预算在渲染器入口显式声明——`packages/tui/src/app.tsx:241`：`targetFps: 60`（OpenTUI 渲染器选项，见 [../2_tech-stack/2.1_client/3_opentui.md](../2_tech-stack/2.1_client/3_opentui.md)）。TUI 的"渲染"是终端 cell diff，OpenTUI 的渲染器按目标帧率调度重绘；配合 Solid batch（TUI 层 `app.tsx:633`、`ui/dialog.tsx:312/327` 等"一个用户动作改多个状态"处主动包 batch，见 2.1 篇 §引用），把 N 次状态变更合并成一帧。

---

## 3. 事件流水线的批处理链

传输侧的背压与限额全景在 [5_resource-limits.md](5_resource-limits.md) §2b（订阅者 4096 dropping 队列等）。本节收口**延迟视角**：从 provider 到像素，delta 要过三道批，每道理由不同。

```
provider 流式 token
│
├─ 批 1：server 侧 100ms delta 批
│    core/src/session/runner/publish-llm-event.ts:106  const deltaBatchInterval = 100
│    append() (:190-205)：chunk 累进 current.pending，首次到达 fork 定时 fiber，
│    循环 "sleep(100) → publishDelta"（:197-203），publishDelta 先清空 pending 再发
│    （:183-185 注释："New chunks can arrive while the timer is publishing this batch"）
│    理由：按 token 发事件 = 总线/SSE/DB 三层全在 token 粒度工作；
│    100ms 批把事件量压到 ~10/s 且低于人眼对流式文字的敏感阈值。
│    边界纪律：block start 不批（:300 注释 "Deltas are batched, but block starts are not"），
│    terminal 事件前先 flush pending（保证 ordinal 单调、片段不丢）。
│
├─ 批 2：client 侧 10ms flush 批
│    client/src/solid/connection.ts:64-74  publish() 把事件压入 pending，
│    setTimeout(flushInterval ?? 10) 后一次性交给 onEvent
│    理由：SSE 一帧可能带多条事件（批 1 的产物 + 消息/权限事件混流），
│    10ms 微批合并"同一帧到达的多事件"，摊薄每事件的处理摊销；
│    同时它是 4096 dropping 队列的 upstream 節流（5_resource-limits §3 失效表）。
│
└─ 批 3：Solid batch（connection.ts:72）
     batch(() => events.forEach(options.onEvent))
     理由：没有它，每个 onEvent 的 store 写各自触发一轮通知传播——
     N 个事件 = N 轮渲染；batch 把写与通知解耦，批尾合并去重成一次。
     （机制展开见 2.1 篇 §"batch 把写和通知解耦"）
```

**总延迟预算**：最坏 ~110ms + 一帧渲染，换事件处理成本从"每 token 一轮全链路"降到"每 100ms 一批、每帧一次渲染"。三道批各自独立可调（100 是 publisher 内常量、10 是 `flushInterval` 选项、batch 是 Solid 原语），但没有一道是为了另一道存在的——server 批压源头、client 批合帧、Solid 批合渲染，缺任何一层，下一层都会退化为逐事件工作。

### 3.1 传输帧本身：SSE 的成本账

事件到 client 走 SSE（`server/src/handlers/event.ts:23-30`）：`text/event-stream`，每事件一帧 `data: {json}\n\n`（`event-feed.ts:29-31` 的 `frame()`），15s keepalive 注释帧防中间层缓冲（响应头显式 `X-Accel-Buffering: no`、`no-transform`，`event.ts:27`）。编码失败的处理是"宁断不腐"——任一事件编码失败即断开**全部**订阅者（`event-feed.ts:41-46`，见 5_resource-limits §3 失效表），因为一条腐帧会污染所有下游的折叠。客户端 idle 检测（`connection.ts:33-37`：`defaultIdleTimeout = 45_000`，前台回归阈值 20s > 15s keepalive）保证半开连接被换掉而不是干等。

### 3.2 批参数的注入点与测试耦合

三道批里只有 client flush 是**选项**（`connection.ts:19` `flushInterval?: number`）——因为它是唯一需要随测试与宿主环境调整的：单测传 0（`client/test/solid-connection.test.ts:56`）压扁时间维度。server 的 100ms 是 publisher 常量（`publish-llm-event.ts:106`），Solid batch 是框架原语——两者都不留旋钮，**延迟预算是产品决定，不交给配置**。另外批 2 与连接生命周期的耦合被测试钉死：keepalive 帧不触发 flush、强制 resync 立即换流不重放（`solid-connection.test.ts:94,115`），即批的存在不能改变"事件恰好一次到达 store"的语义——批是性能机制，不是语义边界。

---

## 4. 包体积治理

### 4.0 为什么包体积是启动问题

desktop 的渲染资产不走 http 而走 oc:// 自定义协议——**协议处理器在主进程主线程上跑**（§1.5）。传统 web 的"并行六连接下载"模型在此不成立：每个 chunk 文件都要经主进程应答，而主进程同时还在求值自己的 bundle、起 layers。所以包体积在这里的直接计价单位是**主线程请求往返次数**（§1.4 的 ~50 文件问题）与**初始图模块量**（§4.1 的 1/3 问题），而不是下载字节数——本地文件没有网络传输成本，只有线程时间成本。

### 4.1 renderer 不背 Effect runtime（ipc-client.ts:7-13）

`desktop/src/renderer/ipc-client.ts`（203 行）开篇注释是全篇最重的一句性能声明（原文）：

> "…the renderer needs neither the Effect runtime nor the contract's schemas to talk to it. Keeping them out of the renderer's initial module graph is worth about a third of its startup script."

渲染进程直接说 Effect RPC 的**线格式**（MessagePort + structured clone，二进制保持二进制），不引 Effect、不引 schema——初始模块图砍掉约 1/3 启动脚本。这与 §1.4 的单 chunk 策略互补：chunk 策略管"怎么打包"，裸协议管"打包进什么"。promise 客户端同哲学（`client/src/promise/service.ts:13-15` 注释："intentionally implemented with Node APIs so Promise clients do not need Effect or @effect/platform-node at runtime"）。

### 4.2 动态 import 清单（实测）

启动敏感路径上的重模块全部 `await import` 懒加载（与仓库 AGENTS.md 风格规则一致："Prefer dynamic imports for heavy modules… especially in startup-sensitive entrypoints"）：

```
主进程（desktop/src/main/）
├─ index.ts:33                  import("./desktop")            主 bundle 让行后求值
├─ extension/index.ts:73        import("./host")               扩展宿主首用加载
├─ extension/manager.ts:85,89   import("./archive")/electron   装扩展时才引 zip.js
├─ service/sidecar-probe.ts:18  import("@opencode/client/service")
├─ service/background-service.ts:42  同上（后台服务生命周期）
├─ lifecycle/environment.ts:13  import("electron-context-menu")（desktop.ts 里
│                               sleep 500ms 后才装——"not worth answering late"）
└─ native/logging.ts:296        import("@zip.js/zip.js")       日志压缩时才引

渲染进程
├─ renderer/desktop-app.tsx:32  lazy(() => import("./migration-status"))
└─ app/src/shell/routes/routes.tsx:14-26  File/DraftRoute/SettingsScreen/
    ConnectServerScreen/TargetSessionRouteContent 全部 lazy——
    session 路由 preloadRoute 与 File.preload() 并行（:16）
```

懒加载的判据从清单里读得很清楚：**重模块 + 选定路径才需要**——zip.js 两处（装扩展/压日志）、client/service 两处（后台服务生命周期）、context-menu（依赖树）、migration-status（仅迁移期）、五个路由（仅导航到时）。反向约束同样成立：启动热路径上的东西**不许**动态 import（`index.ts` 头部的静态 import 全是轻配置件），否则首窗等 import 完成才 ready——这正是 AGENTS.md 风格规则"Keep branch-specific imports inside the branch that needs them"的性能成因。

主进程还有打包级配合：Effect 家族 + drizzle 显式 bundle 进主进程（`electron.vite.config.ts:86-90` exclude externalize），"Bundle the Effect family together"——主进程吃一份完整 runtime 换运行时一致性，renderer 一点不背——**两端各取所需**。preload 更是极简到 32 行（见 [6_security.md](6_security.md) §3.2）：桥面小既是安全面也是加载面。

构建契约本身有测试钉住（`electron.vite.config.test.ts`）：minify 只在 build、sourcemap 必开、Effect 家族 bundled 而不重复 external——**打包策略是回归测试保护的性能预算，不是一次性配置**。preload 更是极简到 32 行（见 [6_security.md](6_security.md) §3.2）：桥面小，既是安全面也是加载面。

构建契约本身有测试钉住（`electron.vite.config.test.ts`）：minify 只在 build、sourcemap 必开、Effect 家族 bundled 而不重复 external——**打包策略是回归测试保护的性能预算，不是一次性配置**。

---

## 5. 内存预算

### 5.1 Location 图：永不过期的内存换稳定

Location 服务图（每目录一份 Layer 编译产物）的 LayerMap 显式 `{ idleTimeToLive: Duration.infinity }`（`core/src/location-services.ts:56`）。这是**明知取舍**而非遗漏：域模型篇的结论（[../6_modules/15_location.md](../6_modules/15_location.md) §4.1/§7 引述）——"LayerMap 自身的 idleTimeToLive 是 infinity，逐出决策权在 activity 这一层显式行使"，即空闲逐出走 `location-activity.ts` 的显式通道（能先中断会话再逐出），缓存层自己绝不过期。**换到什么**：正在借图运行的会话永远不会遭遇"缓存 TTL 到期被腰斩"；**付出什么**：活跃目录的图常驻内存。长会话稳定性 > 常驻内存，这是一次显式定价。

### 5.2 desktop 扩展缓存：500 条 / 8MiB

渲染进程持久化层读缓存（`app/src/runtime/persistence/storage.ts:51-53`）：

```typescript
const CACHE_MAX_ENTRIES = 500
const CACHE_MAX_BYTES = 8 * 1024 * 1024
```

`cachePrune()`（`:69-76`）插入后按 **Map 迭代序（即 LRU 序，`cacheGet` 每次 delete+set 提升）** 淘汰最旧项直到双上限满足；单条超过 8MiB 直接不缓存（`:82-84`）。同款纪律的邻居：文件内容 LRU `content-cache.ts:5-8`（40 条 / 20MiB，字节估算函数 `approxBytes` 连 diff/hunk 行都按 2 字节/字符计）。扩展包本体另有限额：安装包 ≤100MiB / 4096 条目（`extension/archive.ts:6-8`）——超限 `tooLarge` 拒装，这是内存恐惧在安装边界的形态。

### 5.3 store 增长边界（实测）

实测结论：**消息列表没有客户端裁剪**——timeline 的 `model.ts`/`projection.ts` 不设条数上限，全量进内存；增长边界全部由**渲染层虚拟化**（§2.2：只挂载视口附近的行）承担，导出走服务端分页（`session/commands/export.ts:16` `limit: 200`）。事件侧历史有界：connection 历史环 50 条（`connection.ts:37` `connectionHistoryLimit = 50`）。即：**内存随会话长度线性增长是被接受的，被治理的是"增长如何不转化为渲染成本"**。

### 5.4 SSE 订阅者的容量账

server 侧每个 SSE 订阅者持一条 dropping 队列，容量 4096 帧（`server/src/event-feed.ts:8` `SubscriberCapacity = 4_096`）。满时**只断开该订阅者**（`:64-68`：`Queue.offerUnsafe` 失败 → 从 subscribers 摘除 → 该队列 fail），发布者与其余订阅者不受影响——慢客户端的内存上界是 4096 × 帧大小，且代价由它自己承担。这是内存预算与背压的交点，容量语义详见 [5_resource-limits.md](5_resource-limits.md) §2b。

### 5.5 内存预算的域外邻居

内存治理不只在 app 层：附件物化 20MB 硬拒（server `prompt.ts:203`，限额语义见 5_resource-limits §1 表）、日志文件 50MiB 上限就地压缩（`observability/logging.ts:9-11`）、drafts 按时间清退（`desktop/src/main/storage/drafts.ts` 的 touched_at 扫描）。共同模式：**每条内存/磁盘预算都贴在生产者边界上**（谁生产谁截断），没有一个集中式内存管理器——这与 5_resource-limits 的"限额是局部不变量"结论完全同构，内存只是其中一个维度。

---

## 6. 失效场景与预算冲突

| # | 冲突/场景 | 机制表现 | 取舍裁决 |
|---|---|---|---|
| 1 | 批延迟 vs 实时性 | delta 批 100ms + flush 10ms：流式文字最坏多 ~110ms 才上屏 | 接受——低于人对"在打字"的感知阈值；但 block start 不批、terminal 前 flush（publish-llm-event.ts:300），结构事件零延迟 |
| 2 | 让行 capMs vs 启动 | rendererAssetsServed 等资产静默，最坏 400ms 才求值主 bundle | 有界等待——宁可主包晚 400ms 也不让渲染进程 HTML 排在 100ms 模块求值后面 |
| 3 | 单 chunk vs 懒加载缓存 | initial 图全进一个 `app` chunk，首访即全量下载（oc:// 本地文件，无网络成本） | 本地协议下往返成本 > 体积成本；远程分发场景该权衡会反转 |
| 4 | 缓存 vs 内存 | Location 图 infinity TTL：活跃目录图永驻 | 显式定价：会话稳定 > 内存；逐出权显式交给 activity 层 |
| 5 | 裸协议 vs 类型安全 | renderer 甩掉 Effect+schema 后，RPC 契约靠 main 进程可信 + 手写 wire 解码 | main 进程 trusted（同源沙箱内），契约漂移由 IPC contract 测试兜住 |
| 6 | 虚拟化冷尾 vs 搜索 | initialTail 模式只挂底部，Ctrl+F 搜早前历史需先退出冷尾展开 | settle 后退出 initialTail；搜索走 search-controller 投影而非全量挂载 |
| 7 | 慢客户端 vs 事件流 | 4096 dropping 满则断该订阅者（event-feed.ts:64-68） | 宁断不腐——细节在 5_resource-limits §2b/§3 |
| 8 | 消息无界 vs 长会话 | §5.3：全量进内存，仅虚拟化治理渲染成本 | 已接受的债务；超长会话的内存形态由 provider-memory-benchmark 监控 |

---

## 7. 测试覆盖

```
预算域                    测试（实测存在）
────────────────────────────────────────────────────────────────
启动                      desktop/electron.vite.config.test.ts（minify/sourcemap/
                          Effect 同捆/native external 化的构建契约）
                          app/e2e/performance/unit/desktop-startup.test.ts
                          （里程碑行→阶段识别、MAD 统计、主题内联）
                          desktop/scripts/bench-startup.ts（17 阶段冷启动基准，
                          --compare 双构建交替、屏幕像素采样）
delta 批（100ms）         core/test/session-runner-tool-events.test.ts
                          :362 "batches text deltas and flushes pending text
                          before the terminal event"、:395 批间 flush、
                          :461 reasoning delta 同款——ordinal/不丢片段被钉死
flush 批（10ms）          client/test/solid-connection.test.ts（idle 替换/
                          keepalive/强制重同步，flushInterval 可注入 :56）
渲染/内存基准             app/e2e/performance/（Playwright 基准套件：
                          timeline/ 首导航/切 tab/流式/搜索基准、
                          provider-memory-benchmark.spec.ts——CDP
                          Runtime.getHeapUsage 显式 GC 后量 retained heap）
扩展缓存/限额             extension/storage.test.ts、manager.test.ts；
                          app content-cache-accounting.test.ts（LRU 字节记账）
虚拟化                    timeline/controller-projection.test.ts、
                          observe-element-offset.test.ts、
                          e2e timeline-stability/（视觉稳定性）
```

值得单记：`provider-memory-benchmark` 把 GC 当**测量步骤**而非优化（spec 注释原文："GC is an explicit retained-heap measurement, not an application optimization"）——内存预算的回归检测是显式量出来的，不是猜的。bench-startup 的 `--compare` 交替启动设计（注释："alternates launches of a second build so machine drift affects both equally"）同理：性能数字的敌人是机器漂移，对拍消掉它。

基准套件的覆盖面（`app/e2e/performance/` 实测目录）：timeline 域有首导航/切 tab/流式/搜索/projection/滚动稳定性六组 spec（`first-navigation-benchmark`、`session-tab-switch-benchmark`、`session-timeline-stream-probe`、`session-timeline-search-benchmark`、`timeline-projection-benchmark`、`timeline-stability/`），启动域有 `devex/desktop-startup-benchmark.spec.ts`，另有 markdown 渲染、composer 写批（`composer-write-batch-benchmark`）、补丁组（`patch-groups/`）、终端（`terminals/`）等专项。通用骨架是 `benchmark.ts` 的 fixture 扩展：每个 spec 恰好 `report` 一次指标、`benchmarkResult` 钩子把结果按 schemaVersion 2 的 JSON 落盘——**基准是流水线产物，不是手抄数字**。

---

## 演进注记

对照旧卷：启动时序（5_timeline/1）、观测计时（7_observability §5）、限额全景（5_resource-limits）各有其篇；本篇为**新增域**——首次把启动/渲染/传输/内存/包体积五类机制按"恐惧 → 预算 → 冲突裁决"收口成性能视角总图，并引入此前未覆盖的实码：initial chunk 插件的往返成本注释（electron.vite.config.ts:35-38）、ipc-client 的 1/3 启动脚本声明（:7-13）、三级批处理链的延迟账（publish-llm-event.ts:106 + connection.ts:64-74）、virtualizer 冷尾挂载策略（:102-177）、persistence 双上限 LRU（storage.ts:51-53）。v1 时代无 desktop 包、无 Solid 客户端，批处理链与 chunk 治理均为 v2 desktop 重构的产物。

与相邻篇的分界：5_timeline/1 写"启动时发生了什么"（因果链），本篇写"为什么这样安排更快"（预算账）；5_resource-limits 写"量超了会怎样"（限额与背压），本篇写"延迟与体积花在哪"（预算的支出面）；7_observability §5 写"怎么量"，本篇各节引用其结果作为机制存在的前提。三篇合起来才是完整的性能叙事：**量出来（obs）→ 防住（limits）→ 定价（本篇）**。

## 相关篇

- [7_observability.md](7_observability.md) §5 —— marks 启动计时与 bench-startup 阶段拆分（本篇 §1.7 的观测面）
- [5_resource-limits.md](5_resource-limits.md) —— 限额全景与事件背压（本篇 §3 的容量视角）
- [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5/§2.1 —— 启动时序全走读（本篇 §1 只收性能口）
- [../2_tech-stack/2.1_client/2.1_solidjs-reactivity.md](../2_tech-stack/2.1_client/2.1_solidjs-reactivity.md) —— 细粒度响应式机制（本篇 §2.1 的功能面）
- [../6_modules/15_location.md](../6_modules/15_location.md) —— Location 域模型（本篇 §5.1 的缓存语义出处）
