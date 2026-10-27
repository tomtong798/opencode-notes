# 桌面端启动全流程：从点击图标到可交互

> 一句话：**主进程环境准备 → 窗口先行上屏 → 并行（探测/拉起后端服务 + 渲染进程加载）→ MessagePort 交接 → 渲染进程直连服务同步数据 → 可交互。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16）；行号以 `packages/desktop` 现码为准。
> 前置：[1_multi-process.md](../4_desktop/1_multi-process.md)（进程拓扑与通信通道）| [返回 README](README.md)

---

## 1. 全景时序

> **一句话校准版（完整流程）**：主进程带参起锁 → 窗口抢跑上屏 → 探测（只看不动）→ 重模块加载完 → ensure 幂等拉起/复用服务 → 轮询注册文件拿 {url, 密码}（**密码止步主进程**）→ 与 renderer 的 MessagePort 交接 → renderer 直连同步 → 可交互。
> 三个易错点：① "探测"（T5）与"启动"（T9 ensure）是分离的两阶段——探测只读注册文件探活，spawn 发生在 Layer 就绪后；② 服务就绪后**自己写注册文件**，主进程轮询拿到——是"拉"不是"推"；③ 拿到 url/密码 ≠ 界面可用，中间还有 IPC 交接（T10）与首屏数据同步（T11）。

```
用户双击图标
════════════════════════════════════════════════════════════════════
[T0] OS 启动主进程（Electron app）
     │
     ├─ [T1] configureApplication()            lifecycle/configure.ts
     │      userData 定位（按通道分 APP_ID）、compile cache、CLI switches
     │      （函数走读 → §1.5.1）
     │
     ├─ [T2] acquireApplicationLock()          configure.ts:45-50
     │      单实例锁；已在运行 → 新实例把 argv 里的深链转交老实例后退出
     │
     ├─ [T3] registerRendererScheme()          windows/scheme.ts
     │      注册 oc:// 协议（必须在 ready 前）
     │
     ├─ app.whenReady() ──────────────────────────────────────────
     │
     ├─ [T4] createEarlyWindow()  ★窗口先行    windows/early.ts
     │      用户看到窗口骨架（此刻 Effect Layer 一行都没加载）
     │      （函数走读 → §1.5.2）
     │
     ├─ [T5] startSidecarProbe()  ★并行启动    service/sidecar-probe.ts
     │      Service.discover({version})：读注册文件 + 探 /api/info
     │      ├─ 找到兼容服务 → 记为 early，后续直接采用
     │      （函数走读 → §1.5.3）
     │      └─ 没找到 → 等 BackgroundService ensure() 拉起
     │
     ├─ [T6] rendererAssetsServed({quietMs:40, capMs:400})
     │      等 renderer 首批静态资源服务完（同一线程，避免抢 CPU）
     │      （机制 → §1.5.4）
     │
     └─ [T7] import("./desktop")               desktop.ts
            懒加载重组件：IPC Layer、ApplicationLifecycle、
            BackgroundService、DesktopCli、Storage、Extension host
════════════════════════════════════════════════════════════════════
[T8] 渲染进程（与 T4-T7 并行）加载 @opencode/app
     │   ★渲染进程无显式启动代码——它是 T4 的 new BrowserWindow()
     │    的框架副产品（Chromium 自动 fork）；本站的触发链是
     │    T4 loadWindow("index.html") → 渲染进程发 oc:// 请求
     │    → 主进程 protocol.handle 应答 → JS bundle 求值（→ §1.5.7）
     │
     ├─ preload 先行注入（contextIsolation 沙箱内）
     │      window.electron = { windowID, bootstrap, storageSnapshot,
     │                         extensions, getPathForFile }   preload/index.ts
     │      （此刻 MessagePort 还没到手——见 T10）
     │
     └─ 应用启动：显示加载态，等待两件事就绪：
          ① AppAwaitInitialization RPC 的应答（服务 URL）
          ② MessagePort 交接
════════════════════════════════════════════════════════════════════
[T9] 后端服务就绪（主进程侧）              service/background-service.ts
     │
     ├─ early ?? ensure()：采用已发现的服务，或拉起 bundled CLI
     │      命令：[cli, "serve", "--service"]
     │      CLI 二进制：resources → staging 到 userData/cli/<ver>-<arch>/
     │      （spawn 的真实位置与竞争机制 → §1.5.8）
     │
     ├─ 服务写注册文件（XDG state，0600）→ /api/info 可答
     │
     └─ ServerReadyData = { url } 发往 renderer（只有 URL，没有密码）
            密码由主进程持有（sidecar-credentials.ts）
════════════════════════════════════════════════════════════════════
[T10] IPC 交接：did-finish-load 时           main/ipc.ts:66-77
      （post() 走读 → §1.5.5）
     │
     └─ 主进程创建 MessageChannelMain
        port1 留 Main（RpcServer.Protocol 起服务）
        port2 经 ipcRenderer 转交 renderer（preload window.postMessage）
        renderer 侧 ipc-client.ts 直接讲 Effect RPC 线协议
════════════════════════════════════════════════════════════════════
[T11] 数据同步与可交互
     │
     ├─ renderer 内嵌 @opencode/client（与 TUI/Web 同一套）
     │      拉初始数据：会话列表/配置/模型…（HTTP，鉴权头由主进程注入）
     │      订阅事件流：消息/权限/表单实时更新
     │
     ├─ 首屏数据到位 → 界面切换到可用状态
     │
     └─ 此后：深链（second-instance/open-url）、菜单命令、窗口事件
            都经 MessagePort RPC / DesktopEvents 流入 renderer
════════════════════════════════════════════════════════════════════
```

## 1.5 关键函数走读（T 序承重函数下钻）

全景时序只点了函数名；这里把六个承重函数按调用序展开。行号实测于 `packages/desktop`（2026-10-19）。

### 1.5.1 `configureApplication()`（T1，`lifecycle/configure.ts:16-43`）

同步执行于 ready **之前**（注释原话：命令行开关只在 ready 前生效、单实例锁按 userData 划定、early window 需要 userData 找窗口状态）：

```
configureApplication()
│
├─ :18-19   process.chdir(homedir())——从哪启动都不影响相对路径
├─ :21      OPENCODE_DISABLE_EMBEDDED_WEB_UI=true
│           （桌面端禁用 CLI 内嵌 web UI，走自己的 renderer）
├─ :23-27   app.setName/setAppUserModelId（任务栏身份按通道 APP_ID）
│           + 命令行开关：代理绕回环、崩溃报告带 JS 调用栈
├─ :29-30   开发模式：远程调试端口 9222
├─ :32-39   ★ userData 定位（createTestRoot :52-73）
│           ├ 正常：appData/<APP_ID>（按通道分库）
│           └ 测试：OPENCODE_DESKTOP_TEST_ROOT / onboarding 临时根
│               （整套 XDG_* 重定向 + :memory: DB——E2E 隔离的关键）
└─ :42      enableCompileCache(userData/compile-cache)
            ★主 bundle 的 V8 字节码跨启动复用（对应 renderer 的代码缓存）
```

### 1.5.2 `createEarlyWindow()`（T4，`windows/early.ts:36-93`）

"抢跑窗口"——见 §2.1。关键在**它只接上首请求前必需的最小接线**：

```
createEarlyWindow()
│
├─ :37-41  读持久化窗口 ID 列表 + 状态文件（位置/尺寸还原，默认 1280×800）
├─ :44-63  new BrowserWindow({show:true})
│           ├ 持久化背景色充当主题占位（renderer 加载后才应用真主题）
│           └ 平台样式全镜像正式窗口（macOS 红绿灯位 / Win 无边框 overlay）
├─ :65     manageWindowState（移动/缩窗持续落盘）
├─ :67-71  ★收养前关窗 = app.quit()（见 §2.1 注 3）
├─ :73-85  挂 pending（模块级 let :24）等待收养
└─ :88-92  最小接线四件套 + loadWindow(index.html)
            renderer 协议 / 权限 / 导航策略（临时 openExternal，
            layers 起来后换正式版）/ 响应头
```

收养现场在 `windows/index.ts:126-135`（`restore()`）：`takeEarlyWindow()`（:128）取走 pending → 若窗口没被销毁则替换 `create()` 里的全新创建 → 已在 `list` 之外的 early 窗口直接 `destroy()`。

### 1.5.3 `startSidecarProbe()`（T5，`service/sidecar-probe.ts:13-21`）

利用"窗口已上屏、主 bundle 还没加载"的**空档**做服务探测（文件头注释：主线程此刻在等 renderer 的资源请求，这个槽够查明兼容服务是否已在跑）：

```
startSidecarProbe()
│
├─ :14  仅打包模式（app.isPackaged）——开发模式直接跳过
├─ :27-32  bundledVersion()：读 resources/opencode-cli.version
│           （bundled CLI 的版本号，作为兼容性判断基准）
└─ :18-20  probe = 动态 import @opencode/client/service
            → Service.discover({version})   ★只探测，不拉起
            （注释：需要拉起的服务等 layers——layers 设置 CLI 期望的环境）
            失败 → undefined（catch 吞掉，探测是尽力而为）
```

结果经 `sidecarProbe()`（:23-25）供 `background-service.ts` 的 `early ?? ensure()` 消费——**能复用全机已有服务就不重复拉起**。

### 1.5.4 `rendererAssetsServed({quietMs:40, capMs:400})`（T6，`windows/protocol.ts`）

同线程让行：renderer 的 HTML/首屏 chunk 由**本线程**的协议处理器供出，bundle 的 import 若抢跑会拖慢 ready 本身（index.ts:22-23 注释）。此调用等"资源请求静默 40ms 或封顶 400ms"再放行重组件加载——**同一线程上的启动编排**，避免主进程和 renderer 抢 CPU。

### 1.5.5 IPC 交接：四段接力（T10）

T10 = renderer↔主进程那条"专用电话线"（MessagePort 双端口通道）的架设时刻。之后所有 RPC（存储/CLI/窗口/菜单，7 组 50+）都走这一对端口，不再有零散 `ipcRenderer.invoke` 通道。端口对要跨三层（主进程 → preload 沙箱 → 页面），每层只做"转交"：

```
① 主进程：造端口，一端自留、一端递过去        main/ipc.ts:66-76
│
│ win.webContents.on("did-finish-load", post)  :73
│ └─ post()  :66-71
│     ├ new MessageChannelMain()               :68  一对端口诞生
│     ├ handoff.bind(webContents, channel.port1) :69
│     │   port1 留主进程：绑到 RPC 服务器（见④）
│     ├ webContents.postMessage(               :70
│       IpcTransportPort, null, [channel.port2])
│     │   port2 随消息"转交"renderer（transferable）
│     └ :76  ★兜底：early window 可能在 layers 存在
│         前就已 load 完（!isLoading() && getURL()
│         → 立即 post）
│
② preload：从 Electron 通道捞 port2，转发进页面  preload/index.ts:11-14
│
│ ipcRenderer.on(IpcTransportPort, (event) => {
│   const port = event.ports[0]
│   window.postMessage(IpcTransportPort, "*", [port])
│ })
│   ★preload 在 contextIsolation 沙箱里，无法直接
│    把对象塞给页面——只能借 window.postMessage
│    把 port 再转交一次（"交接"的第二跳）
│
③ renderer：页面里捞到 port，接管              renderer/ipc-client.ts:42-56
│
│ window.addEventListener("message", (event) => {
│   if (event.source !== window || event.data !== IpcTransportPort) return :44
│   const value = event.ports[0]               :45
│   value.addEventListener("message",          :50  收 RPC 响应
│     (m) => receive(m.data))
│   value.start()                              :51  ★启动端口
│   resolve(value)                             :52  交付给全体调用方
│ })                                           （Promise 形态，先到的
│                                              调用自动排队）
│
④ 主进程 port1 侧：RPC 服务器绑端口            main/ipc-transport.ts:26-70
│
│ IpcServerProtocolLive：RpcServer.Protocol.make
│   ├ 每窗口一个 binding（:55 bind）
│   ├ ★同 webContents 重复 bind → 先 disconnect 旧
│   │  binding（:55-58）——did-finish-load 每次
│   │  导航都触发，页面重载后天然换线重接
│   ├ disconnect 清全套监听 + port.close（:43-51）
│   └ 消息过端口走 structured clone
│      （ipc-transport.ts:22-25 注释：无序列化层，
│       二进制保持二进制；MessagePortMain 只能
│       transfer 端口，字节负载双向 clone）
```

三个设计要点：

1. **renderer 讲裸协议，不背 Effect 运行时**——`ipc-client.ts:8-12` 注释（原文 *"worth about a third of its startup script"*）：renderer 直接讲 Effect RPC 线协议，省掉 Effect runtime + 契约 schema **约 1/3 的启动脚本体积**；不解码的前提是"主进程可信"（沙箱 renderer 只信这一个对端）。
2. **契约单处定义**——两端共享 `src/shared/ipc-rpc/*.ts` 的类型（不共享运行时）；renderer 侧 `request()` 按 id 配对 pending Map（ipc-client.ts:29）。
3. **端口生命周期跟窗口走**——`browser-window-created`（ipc.ts:82）给每个新窗口 wire；`registerIpcHandlers` 本身在 Layer 里，首窗口靠 ① 的 :76 兜底补上。

### 1.5.6 入口编排（`main/index.ts:15-31`）

```
configureApplication()                :15  同步，ready 前
│
acquireApplicationLock()              :17  锁失败 → 深链转交老实例后退出
│
registerRendererScheme()              :18  oc:// 协议，必须 ready 前
│
app.whenReady().then:                 :21
├─ marks.ready                        :22  计时点（性能基准可断言）
├─ registerStorageSnapshotHandler()   :23
├─ createEarlyWindow()  ★T4           :24
├─ marks.window                       :25
├─ startSidecarProbe()  ★T5           :26
├─ rendererAssetsServed ★T6           :29  静默 40ms / 封顶 400ms
└─ import("./desktop")  ★T7           （后续）
   懒加载：IPC Layer、ApplicationLifecycle、
   BackgroundService、DesktopCli、Storage、Extension host
```

入口模块**刻意保持小**（文件头注释：Electron 会 hold ready 直到入口模块求值完毕，窗口必须抢在"几百毫秒的模块求值 + layers"之前上屏）。

### 1.5.7 渲染进程如何"启动"（T8 的触发链，T4 的续篇）

代码里**没有** `spawn renderer` 式的显式启动调用——渲染进程是 Electron 框架的副产品。完整因果链：

```
T4: new BrowserWindow({...})              early.ts:44
 │   【框架内部，非应用代码】
 │    Chromium 为这个窗口自动 fork 渲染进程
 │    （sandbox:true + contextIsolation:true 的独立进程）
 │    此刻只是空白壳——还没有任何页面
 │
 ├─ preload 声明                          early.ts:57
 │   preload: ../preload/index.cjs
 │   渲染进程加载页面前先执行 preload（沙箱内装 window.electron）
 │
T4: loadWindow(win, "index.html")         early.ts:92 → scheme.ts:11-25
 │   生产：win.loadURL("oc://renderer/index.html")
 │   开发：ELECTRON_RENDERER_URL（vite dev server）
 │
 ├─ 渲染进程向主进程发 oc:// 协议请求
 │   │
 │   ▼ 主线程 protocol.handle 接住          protocol.ts:56-67
 │      （early.ts:88 registerRendererProtocol 注册，
 │       rendererRoot = ../renderer 即构建产物目录）
 │      ├─ host 校验（只答 renderer/extensions，:83-87）
 │      ├─ 路径逃逸防护（rel.startsWith("..")，:91）
 │      └─ net.fetch 读磁盘文件应答 + Range 支持（:99）
 │          ★主进程就是 oc:// 的"HTTP 服务器"——
 │           这就是 T6 rendererAssetsServed 让行的原因：
 │           资源请求由主线程应答（protocol.ts:30-35 计数）
 │
 └─ 渲染进程收到 HTML → JS bundle 求值
     → @opencode/app 启动 → Solid 应用挂载
        这才是 T8 说的"渲染进程加载 @opencode/app"
```

三个容易误读的点：

1. **`new BrowserWindow()` ≠ 只是开个窗**——窗口对象背后就是一对新进程（渲染进程 + GPU 合成器参与），Electron 没有也不需要独立的"启动渲染进程"API。
2. **渲染进程的"入口"在主进程代码里**——preload 路径（early.ts:57）与页面 URL（early.ts:92）都是 BrowserWindow 构造参数/加载调用指定的；渲染进程自己没有任何启动逻辑。
3. **oc:// 是 privileges 协议**（scheme.ts:40-60 `registerSchemesAsPrivileged`，必须 ready 前注册——即 T3）：`codeCache:true` 让 Chromium 为 renderer bundle 保留 V8 字节码（对应主进程侧的 `enableCompileCache`），`corsEnabled` 允许扩展资产跨源加载。

### 1.5.8 后端服务的诞生（T9 下钻：spawn 的真实位置）

桌面代码里**搜不到后端进程的 spawn**——`background-service.ts` 只决定"用什么命令、什么版本门槛、要不要启动"；真正的 `spawn()` 在 `@opencode/client` 的 Service.ensure 里。完整两层链：

```
第一层：desktop 主进程（Layer 就绪后）
│
BackgroundService.layer                 background-service.ts:18-30
└─ connect("initial")                   :32-92
   │
   ├─ :38   desktopCli.resolve
   │        bundled CLI 二进制：resources → staging 到
   │        userData/cli/<ver>-<arch>/（desktop-cli.ts 管 staging）
   │
   ├─ :44-61 ★组装 ensure 订单（此刻还没启动）
   │        client.Service.ensure({
   │          command: [cli..., "serve", "--service"],
   │          version,        // 版本门槛（bundledVersion）
   │          onStart,        // 仅日志钩子
   │        })
   │        isolated 模式固定端口 0x0c0c（:51-52 注释：
   │        固定端口让第二个竞争者 bind 失败退场，
   │        port 0 永不冲突 → 可能双服务对同一个库）
   │
   ├─ :65   early = T5 探测结果
   │
   └─ :67-68 ★三态分派
       ├─ 命中 early → 直接采用；ensure() 仍后台跑一遍
       │   （:63-64 注释：为副作用——terminal handoff 收尾，
       │    不挡 renderer 路径）
       └─ 未命中 → yield ensure() → 进入第二层
           ▼
第二层：@opencode/client（TUI/CLI 共用同一套）
│
Service.ensure                           client/src/effect/service.ts:49-127
语义是"确保在跑"而非"无脑启动"
│
├─ :72   读注册文件（XDG state，0600）
│        文件头注释（:16-20）："注册文件就是全部发现契约"
│        ——url/pid/version/password，探活只需 HTTP 探 /api/info
│        └─ :92 decide 三决策
│            ├─ reuse    健康且兼容 → 直接返回（:94-97）★零 spawn
│            ├─ replace  版本不匹配 → stop 旧进程
│            │   （SIGTERM→SIGKILL 升级，:182-197）→ 继续等新的
│            └─ 缺失     → 往下招募 contender
│
├─ :110-112 shouldRecruit → spawnContender（:60-70）
│   │   （PtyHandoff.environment 先接持久终端票据，:63）
│   │
│   └─ spawnServiceContender      client/src/service-contender.ts:14-55
│       └─ :19-24 ★★★ 后端进程在这里诞生
│           spawn(cli, ["serve", "--service"], {
│             detached: true,    // 脱离父进程——桌面退出服务不死
│             stdio: [ignore, ignore, pipe],  // 只留 stderr 尾巴 8KB
│             windowsHide: true,
│           })
│           child.unref()        // 主进程不持引用、不等它退出
│
├─ :61    默认命令就是 "opencode serve --service"
├─ :116-121 轮询循环：spaced(pollInterval) + upTo(promiseTimeout)
│           反复重读注册文件，直到新 contender 写出 url → 返回 endpoint
└─ contenderPool（service-contender.ts:61-118）
    "竞争者池"——多客户端竞速不双启的机制：
    ├─ 最多同时 2 个 contender（:104-108 注释：
    │   一个候选 + 一个锁探针；失败后不补员防错被掩盖）
    ├─ 谁先注册成功谁赢（:94-96 reuse 分支）
    ├─ 干净退出（exitCode 0）= 别人赢了 → spawnDelay ×2 退避（:95-96）
    └─ 无响应 3 次超时（:75-88）→ 清 PTY + terminate + 重新招募
```

spawn 出来的 CLI 进程自己走 `opencode serve --service`——即 [2_server-init.md](2_server-init.md) 整篇的启动链；就绪后**自己写注册文件**，ensure 轮询读到它返回 `{url, password}`，主进程经 `SidecarCredentials.set`（background-service.ts:89）持有密码、只把 URL 发给 renderer（凭据边界见 §2.3）。

三个关键认知：

1. **spawn 在 client 包不在 desktop 包**——这个分层让 TUI/CLI 的托管服务（`opencode service` 命令族）与桌面走完全相同的 ensure 语义；desktop 只贡献"命令 + 版本门槛"两个参数。
2. **`detached + unref` = 服务与桌面不同生命周期**——桌面退出服务继续跑（下次启动 T5 探测命中 → 热启动零成本）；服务崩溃由注册文件失效表达，客户端重新 ensure 自愈。
3. **ensure 是幂等的协调协议**，不是简单的 start：reuse/replace/compete 三路 + 竞争者池 + 终端票据（PtyHandoff）交接，保证"任意时刻全机至多一个（每版本）服务"且多客户端并发冷启不双启。



### 2.1 窗口先行（T4 < T7）

`index.ts` 原注释：*"the first window should be on screen before the rest of the main process — a few hundred milliseconds of module evaluation and layers — loads"*。重组件（Effect Layer、IPC、存储）全部懒加载在窗口之后——**感知启动速度优先于架构整洁**。计时点用 `lifecycle/marks.ts` 记录（ready/window/served），性能基准测试可断言。

### 2.2 服务探测与开窗并行（T5）

"early window 到 Layer 加载"的空档被用来探测服务：`sidecar-probe.ts` 在 Layer 就绪前先跑 `Service.discover({ version })`，Layer 就绪后 `background-service.ts` 走 `early ?? ensure()`——**能复用全机已有的服务就不重复拉起**（版本不匹配才替换）。

### 2.3 凭据不跨进程边界（T9/T11）

渲染进程只拿到 `{ url }`；它发出的请求经主进程 `webRequest.onBeforeSendHeaders`（`windows/security.ts:45-58`）自动注入 `Authorization: Basic` 头。设计成 CORS-simple GET（无自定义头）正是为了让注入可行——**渲染进程被攻破也拿不到服务密码**。

### 2.4 IPC 一次性交接（T10）

MessagePort 在 `did-finish-load` 时创建并转交，之后 Renderer↔Main 全部流量走这对端口（structured clone，无 JSON 序列化开销）。preload 里没有 `window.api` 式的大接口——**契约集中在 `src/shared/ipc-rpc/*.ts` 一处定义**（7 组 50+ RPC），两端共享类型。

### 2.5 断线与重连

服务重启/升级时：renderer 经 `AppReconnectService` RPC 拿新 URL（`renderer/startup/initialization.ts:17-32` 的 `createSidecarResolver`）；client 库的事件流自带重连 + 缓存失效重同步——桌面端不特殊处理，复用与 Web/TUI 相同的连接语义。

## 3. 冷启动 vs 热启动

| 场景 | 差异 |
|---|---|
| **冷启动**（服务未跑） | T5 探测失败 → ensure() 拉起 CLI（含 200MB 二进制 staging，版本按 `size:mtime` 缓存，二次启动跳过） |
| **热启动**（服务已跑，如 TUI 先开过） | 探测命中 → 直接采用，无 spawn；窗口 + 数据同步为主耗时 |
| **第二实例** | T2 锁失败 → 深链转交老实例后退出（`lifecycle/index.ts:79-88`） |

## 4. 启动失败与恢复路径

- **renderer 无响应/崩溃**：`windows/recovery.ts`（179 行）接管——采样无响应、崩溃对话框、恢复 UI
- **服务起不来**：连接状态机（`background-service-state.ts`）→ UI 呈现失败态；`AppReconnectService` 支持手动重试
- **深链冷启动**：`opencode://console/authorized?window=<id>`（`lifecycle/deep-link.ts`）进 argv → 锁失败路径转交，或冷启动时经 `AppConsumeInitialDeepLinks` RPC 补投递

## 5. 关键文件对照表

| 阶段 | 文件 |
|---|---|
| T1-T3 准备 | `lifecycle/configure.ts`、`lifecycle/environment.ts`、`windows/scheme.ts` |
| T4 窗口 | `windows/early.ts`、`windows/index.ts` |
| T5 探测 | `service/sidecar-probe.ts` |
| T7 重加载 | `desktop.ts`（Layer 组装） |
| T9 服务 | `service/background-service.ts`、`desktop-cli.ts`（staging）、`cli-stages.ts`（清理） |
| T10 IPC 交接 | `ipc.ts`、`ipc-transport.ts`、`preload/index.ts`、`renderer/ipc-client.ts` |
| T11 数据 | `renderer/startup/initialization.ts`、`@opencode/client` |

## 相关篇

- [../4_desktop/1_multi-process.md](../4_desktop/1_multi-process.md) — 进程拓扑与两条通信通道（本篇的前置）
- [../2_tech-stack/2.1_client/8_electron.md](../2_tech-stack/2.1_client/8_electron.md) — Electron 框架本体
- [../1_architecture/4_build-and-packaging.md](../1_architecture/4_build-and-packaging.md) — CLI 二进制从哪来（T9 的前置）
- [../2_tech-stack/2.6_architecture-styles/2_daemon-client.md](../2_tech-stack/2.6_architecture-styles/2_daemon-client.md) — 服务发现/ensure 全局语义
