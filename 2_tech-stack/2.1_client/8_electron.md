# Electron（44）

> 家族导航：本篇讲 **opencode 怎么用 Electron**；框架本体知识（多进程模型原理/安全基线/打包机制）见子篇 [8.1_electron-fundamentals.md](8.1_electron-fundamentals.md)；桌面端进程架构与启动时序的深挖见 [../../4_desktop/1_multi-process.md](../../4_desktop/1_multi-process.md)。

## 是什么

用 **Chromium（渲染进程）+ Node.js（主进程）** 打包跨平台桌面应用的框架。渲染进程跑 Web 页面，主进程提供系统能力（窗口、文件、IPC、托盘），两层经 IPC（本仓库用 MessagePort RPC）通信。

## 业界类比

- "给你一个专用浏览器 + 后端进程，装成桌面软件"
- Tauri 的 heavier-but-mature 对应物（opencode CI 里也维护 tauri-linux 镜像，供其他用途）

## 在 OpenCode 中的角色

- **桌面端（`packages/desktop`）**：Electron 44，渲染器直接加载 Web 应用 `@opencode/app`（一套代码两端复用）
- 主进程 IPC handlers（`src/main/ipc-handlers/*`）提供系统能力；GUI 扩展（`@opencode/gui-extensions`）经 `./main`/`./renderer`/`./sdk` 三面接入
- 打包：electron-builder；自动更新：electron-updater（更新元数据由 `services/update` 提供）
- 内嵌终端：`@lydell/node-pty`（**node-pty** = 伪终端绑定库，VS Code 内置终端同款方案，让程序以交互方式 spawn shell 并读取输出）
- 持久化用本地 SQLite

## 与其他技术的关系

- **@opencode/app（SolidJS + Vite）**：桌面端复用其作为渲染器
- **gui-extensions**：桌面/共享的扩展 SDK（文件树、变更面板、内嵌浏览器等）
- **plugin-browser**：内嵌浏览器的 RPC/代理层
- **服务进程**：桌面端同样连接本机 opencode 服务（或内嵌），架构上仍是"客户端"

## 相关源码

- `packages/desktop/`（主进程 `src/main/`、渲染器 `src/renderer/`）
- 扩展 SDK：`packages/gui-extensions/`
