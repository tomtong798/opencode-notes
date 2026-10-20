# 桌面端多进程架构（Desktop Multi-Process）

> 一句话：**Electron 主进程"窗口先行、服务并行"启动；渲染进程与主进程经 MessagePort RPC 通信（唯一 IPC 通道），与后端服务经 HTTP(+SSE) 直连、由主进程注入鉴权——两条通道各管一边。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16），行号以现码为准。
> [返回索引](README.md) | 下一篇：[../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md)（启动时序，已迁 5_timeline 卷）
> 前置：[../1_architecture/1_overview.md](../1_architecture/1_overview.md)、[../1_architecture/4_build-and-packaging.md](../1_architecture/4_build-and-packaging.md)

---

## 1. 进程拓扑

| 进程 | 职责 | 代码 |
|---|---|---|
| **Main（主进程）** | 唯一 Electron 主进程：跑 Effect Layer，管窗口、IPC、存储、CLI 服务 | `src/main/index.ts`（35 行刻意保持精简）→ 懒加载 `desktop.ts`（Layer 组装） |
| **Renderer（渲染进程 ×N）** | 每个 BrowserWindow 一个；经自定义 `oc://renderer` 协议加载 `@opencode/app`（与 Web 端同一份代码） | `src/main/windows/scheme.ts:20` |
| **GPU/Utility** | Chromium 自动拉起的辅助进程，应用代码不管理 | — |
| **CLI Sidecar（后端服务）** | 外部 `opencode-cli` 二进制，`opencode serve --service` 起 HTTP 服务 | `src/main/service/background-service.ts` |

**preload 不是进程**：它是跑在 renderer 进程内部的桥接脚本（`src/preload/index.ts`，仅 32 行）。

## 2. 启动时序：窗口先行、服务并行

`src/main/index.ts` 的启动顺序是刻意设计的性能优化（原注释："the first window should be on screen before the rest of the main process loads"）：

```
configureApplication()                 ← L15 同步：userData、compile cache、CLI switches
acquireApplicationLock()               ← L17 单实例锁（configure.ts:45-50，抢不到直接 quit）
registerRendererScheme()               ← L18 注册 oc:// 协议（须在 ready 前）
─── app.whenReady() ───────────────────────────────────────────
  createEarlyWindow()                  ← L24 ★立刻开窗（Effect Layer 尚未加载！）
  startSidecarProbe()                  ← L26 ★并行：探测已有兼容服务（sidecar-probe.ts）
  rendererAssetsServed({quietMs:40, capMs:400})  ← 等 renderer 首批静态资源服务完
  import("./desktop")                  ← L33 最后才加载重组件（IPC/Lifecycle/Storage Layer）
```

要点：

- **窗口先上屏，再加载重模块**：用户几百毫秒内看到界面骨架，Layer 初始化在窗口后面跑
- **服务探测与开窗并行**：`sidecarProbe` 在"early window 到 Layer 加载"的空档里先跑 `Service.discover({ version })`——找到已存在的兼容后台服务就直接采用（`early ?? ensure()`），没有才由 `BackgroundService` 走 `ensure()` 拉起 bundled CLI
- 渲染进程起来时服务可能未就绪：它面对"连接中"状态，经 `AppReconnectService` RPC 重连拿新 URL（`renderer/startup/initialization.ts:17-32`）

## 3. 通信通道：旧"四通道"已收敛为两条

### 通道一：Renderer ↔ Main —— MessagePort RPC（唯一 IPC）

```
Main: ipc.ts:66-77 ── did-finish-load 时创建 MessageChannelMain，port2 转交 renderer
Preload: index.ts:11-15 ── ipcRenderer 收到端口后 window.postMessage 转发
Renderer: ipc-client.ts ── 直接讲 Effect RPC 线协议（structured clone，无需 Effect runtime）
```

**RPC 契约**（`src/shared/ipc-rpc/`，7 组 50+ 个 RPC）：

| 组 | 数量 | 代表 |
|---|---|---|
| App | 13 | `AppAwaitInitialization`、`AppReconnectService`、深链消费、onboarding、relaunch |
| Extension | 13 | 扩展调用/订阅/嵌入/管理 |
| Files | 11 | 文件选择器、剪贴板、路径操作 |
| Window | 10 | 焦点/全屏/缩放/标题栏 |
| Storage | 8 | 状态读写 + 草稿（含 blob） |
| Menu | 1 | `MenuRunAction` |
| Events | 1（流式） | `DesktopEvents`：主→渲染推送（深链/菜单/缩放/存储变更/扩展事件等 11 种） |

**preload 桥**：不再有 `window.api` 大接口——`contextBridge.exposeInMainWorld("electron", {...})` 只暴露启动数据（windowID、bootstrap、storageSnapshot、extensions、`getPathForFile`），完整 API 由 renderer 侧在 MessagePort 之上构建（`renderer/api.ts`）。

### 通道二：Renderer/Main ↔ 后端服务 —— HTTP（+SSE 事件流），鉴权由主进程注入

- **连接管理**：主进程经 `@opencode/client/service` 的 `discover()/ensure()` 发现或拉起服务；命令 `[cli, "serve", "--service"]`（background-service.ts:53-58）
- **鉴权代理**：密码只存主进程（`sidecar-credentials.ts`）；`windows/security.ts:45-58` 用 `webRequest.onBeforeSendHeaders` 给 renderer → 服务的请求注入 `Authorization: Basic` 头——**渲染进程永远见不到密码**，它只拿到 `{ url }`（`ipc-contract.ts:2-4`），且 CORS-simple GET 不触发预检
- **数据流**：渲染进程内嵌的 `@opencode/client`（与 TUI/Web 同一套）拉初始数据、订阅事件流

### 已消失的通道（对照早期资料）

| 早期说法 | 现状 |
|---|---|
| Main↔Sidecar 走 postMessage | ❌ 已无——sidecar 纯 HTTP 服务 |
| Main↔daemon 走 execFile | ❌ 已无——统一为 `@opencode/client/service` 的 ensure/discover |
| `window.api` 大桥接 | ❌ 已无——`window.electron` 仅启动数据 + MessagePort RPC |
| 四条通道 | ✅ 收敛为两条：MessagePort RPC（系统能力）+ HTTP（业务数据） |

## 4. Sidecar 的形态与托管

- **形态**：完整 CLI 二进制（打包在 `extraResources`，运行时 staging 到 `userData/cli/<version>-<arch>/`；版本按 `size:mtime` 缓存避免反复解包 200MB，旧版本目录由 `cli-stages.ts` 清理）
- **三种模式**：内置（默认，打包产物）/ Dev（`OPENCODE_DESKTOP_CLI_DEV` 指向源码目录跑 `bun run src/index.ts`）/ 隔离（`OPENCODE_DESKTOP_ISOLATED_SERVER=1`，独立 state 与端口）
- **单实例语义**：全机一个后台服务（注册文件即发现契约）——桌面端启动时优先采用已存在的兼容服务，版本不匹配才替换

## 5. 安全配置

所有窗口统一（`windows/early.ts:56-62`）：

```ts
webPreferences: {
  contextIsolation: true,   // 隔离 preload 与页面世界
  nodeIntegration: false,   // 渲染进程无 Node 能力
  sandbox: true,            // Chromium 沙箱
  preload: "…",
}
```

配合 `windows/security.ts` 的导航策略（拦截非 oc:// 跳转）与权限处理。深链 `opencode://` 协议注册于 `lifecycle/environment.ts:40-41`（second-instance/open-url 两路事件进 `lifecycle/index.ts:79-94`）。

## 6. 桌面端自有存储（独立于后端服务）

`userData/drafts.sqlite`（`storage/database.ts`：`node:sqlite` + Drizzle，WAL）三张表：state（键值）、document（草稿）、blobs（SHA-256 寻址二进制，60s 孤儿回收/15 分钟宽限）。写入走 write-behind（state 250ms / drafts 500ms）。另有 `window-state-<id>.json`（去抖 100ms 持久化窗口位置）与 settings 同步 JSON store。

**职责边界**：后端会话数据在全局 XDG SQLite（服务的库）；桌面私有数据（草稿/窗口/扩展状态）在 userData——两库互不相干。

## 7. 关键文件速查

| 域 | 文件 |
|---|---|
| 启动 | `main/index.ts`（时序）、`lifecycle/configure.ts`（锁/userData）、`windows/early.ts`（先行窗口） |
| IPC | `main/ipc.ts`、`ipc-transport.ts`、`shared/ipc-rpc/*.ts`（契约）、`renderer/ipc-client.ts` |
| 服务 | `service/background-service.ts`、`sidecar-probe.ts`、`sidecar-credentials.ts`、`desktop-cli.ts`（staging） |
| 窗口 | `windows/index.ts`（239 行，创建/恢复）、`window-state.ts`、`recovery.ts`（无响应恢复） |
| 存储 | `storage/index.ts`、`drafts.ts`、`state.ts` |
| 生命周期 | `lifecycle/index.ts`（198 行：窗口恢复/深链/关机）、`shutdown.ts` |

## 相关篇

- [../2_tech-stack/2.1_client/8_electron.md](../2_tech-stack/2.1_client/8_electron.md) + [8.1](../2_tech-stack/2.1_client/8.1_electron-fundamentals.md) —— Electron 框架本体知识
- [../1_architecture/4_build-and-packaging.md](../1_architecture/4_build-and-packaging.md) —— CLI 二进制与桌面打包
- [../2_tech-stack/2.6_architecture-styles/2_daemon-client.md](../2_tech-stack/2.6_architecture-styles/2_daemon-client.md) —— 后台服务发现/ensure 语义
- [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) —— 点击图标到可交互的完整端到端时序
