# 守护进程 + 瘦客户端（本地版 C/S 分离）

## 是什么

**客户端-服务端分离**架构在 localhost 上的形态：一个常驻**后台服务进程**（守护进程/daemon）持有状态与引擎，多个**瘦客户端**连接它。

- 服务发现：**XDG 状态目录**里的注册文件（`{id, version, url, pid, password}`，0600 权限）是**完整发现契约**——读这个文件就是客户端需要的全部
- 生命周期：`discover()`（只读探测：文件 + 健康检查 + 版本门）、`ensure()`（幂等拉起：健康复用、版本不匹配替换、缺失则 spawn）、`stop()`（SIGTERM→SIGKILL 升级、PTY 移交）

## 业界类比

- "前后端分离，但 server 在 127.0.0.1"
- 语言服务器协议（LSP）的 server 常驻 + 多编辑器接入模式
- docker daemon ↔ docker CLI 的关系

## 在 OpenCode 中的角色

- **`opencode service`** 即此架构的命令面：`start/status/stop/restart/get password` 等
- 一个服务进程服务多个客户端实例：多窗口 TUI、Web、桌面同时连一个服务，共享会话与实时事件（SSE）
- CLI 连接解析（`ServerConnection.resolve()`）：显式 `--server` URL / `--standalone` 内嵌进程内 server / 默认托管后台服务
- **`dev:live` 工作流**：开发 worktree 的 TUI 连**已安装的**后台服务实测（版本不同时显式 `--server` 警告而不替换）
- 服务进程三种模式：default（前台）/ service（托管后台，写注册文件）/ stdio（JSON 传输，stdin 关闭即停）

## 与其他技术的关系

- **SSE**：客户端与服务间的实时通道（见 [../2.2_server/sse.md](../2.2_server/7_sse.md)）
- **SDK（EmbeddedHost）**：不需要独立进程时，把 server 嵌进同进程（`:memory:` DB + Web handler 直连）——C/S 与单进程两种形态按需切换
- **XDG 规范**：注册/配置/数据文件的位置依据
- **恢复机制**：服务重启时恢复挂起会话（write-ahead claim，见 [../2.3_data/event-sourcing.md](../2.3_data/3_event-sourcing.md)）

## 相关源码

- `packages/client/src/service.ts`、`promise/service.ts`、`effect/service.ts`
- `packages/cli/src/services/server-connection.ts`、`commands/handlers/service/*`、`server-process.ts`
- `packages/server/src/process.ts`、`service-status.ts`
- `packages/sdk/src/internal/host.ts`（嵌入形态）
