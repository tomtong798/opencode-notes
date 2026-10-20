# 桌面端启动全流程：从点击图标到可交互

> 一句话：**主进程环境准备 → 窗口先行上屏 → 并行（探测/拉起后端服务 + 渲染进程加载）→ MessagePort 交接 → 渲染进程直连服务同步数据 → 可交互。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16）；行号以 `packages/desktop` 现码为准。
> 前置：[1_multi-process.md](../4_desktop/1_multi-process.md)（进程拓扑与通信通道）| [返回 README](README.md)

---

## 1. 全景时序

```
用户双击图标
════════════════════════════════════════════════════════════════════
[T0] OS 启动主进程（Electron app）
     │
     ├─ [T1] configureApplication()            lifecycle/configure.ts
     │      userData 定位（按通道分 APP_ID）、compile cache、CLI switches
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
     │
     ├─ [T5] startSidecarProbe()  ★并行启动    service/sidecar-probe.ts
     │      Service.discover({version})：读注册文件 + 探 /api/info
     │      ├─ 找到兼容服务 → 记为 early，后续直接采用
     │      └─ 没找到 → 等 BackgroundService ensure() 拉起
     │
     ├─ [T6] rendererAssetsServed({quietMs:40, capMs:400})
     │      等 renderer 首批静态资源服务完（同一线程，避免抢 CPU）
     │
     └─ [T7] import("./desktop")               desktop.ts
            懒加载重组件：IPC Layer、ApplicationLifecycle、
            BackgroundService、DesktopCli、Storage、Extension host
════════════════════════════════════════════════════════════════════
[T8] 渲染进程（与 T4-T7 并行）加载 @opencode/app
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
     │
     ├─ 服务写注册文件（XDG state，0600）→ /api/info 可答
     │
     └─ ServerReadyData = { url } 发往 renderer（只有 URL，没有密码）
            密码由主进程持有（sidecar-credentials.ts）
════════════════════════════════════════════════════════════════════
[T10] IPC 交接：did-finish-load 时           main/ipc.ts:66-77
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

## 2. 五个关键设计点

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
