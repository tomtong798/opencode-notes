# PTY（伪终端 / Pseudo-Terminal）

> 关联篇：[websocket.md](8_websocket.md)（PTY 流为什么走 WebSocket）、[js-runtime.md](1_js-runtime.md)（原生模块按运行时分实现的思想）

## 是什么

**PTY 是 Unix 内核提供的一种进程间通道**：一对虚拟设备（master 端 / slave 端），数据经过时带**终端语义**。它让一个程序"假装自己是终端硬件"，让另一个程序"以为自己连着真终端"。

- master 端：被**终端模拟器**（或宿主程序）持有，读写 = 收发终端数据
- slave 端：被 **shell 及你运行的程序**当作 stdin/stdout/stderr 使用
- 伴随终端语义：窗口尺寸（`resize` 会发 `SIGWINCH` 信号）、行/裸模式（raw mode，按键即达不缓冲）、控制信号（Ctrl-C → `SIGINT`）

## 业界类比

- "电话机的线路模拟器"：shell 那头以为自己在跟一台真实终端硬件通话，其实对面是个程序
- 对比普通管道（pipe）：管道只搬字节，**没有终端语义**——这是 PTY 存在的理由（见下）

### 为什么必须有 PTY（没有它会怎样）

程序会检测"标准输出是不是终端"（`isatty()`）。若是普通管道，程序认定"输出给人看没意义"，于是：

- 关掉**彩色输出**（ls/git 的颜色全没了）
- 关掉**交互提示**（password 输入、y/n 确认失效）
- **vim / htop / top 这类全屏程序直接拒绝运行或乱码**（它们依赖终端控制序列与裸模式）

PTY 让 `isatty()` 返回真，一切照常。**这就是"能输命令、跟真终端一样"的机关所在。**

## 三个易混概念

| 概念 | 是什么 | 例子 |
|---|---|---|
| **终端模拟器**（terminal emulator） | 你看到的界面：画字符、滚动、字体、选中、窗口尺寸 | Desktop 终端 app、VS Code 集成终端、iTerm2、xterm.js、opencode TUI 的终端面板 |
| **PTY** | 内核机制：带终端语义的虚拟通道 | `forkpty()`、`openpty()` 系统调用 |
| **shell** | 在终端里跑的命令解释程序 | bash、zsh、fish |

一句话：**模拟器是脸，PTY 是神经，shell 是脑**。VS Code 集成终端 = VS Code 界面 + node-pty + bash。

## 在 OpenCode 中的角色

opencode 在服务进程里替你 spawn 终端，界面（TUI/Web/桌面）远程消费它：

### 实现层（`packages/core/src/pty/`）

按运行时三份实现、统一 `Proc` 接口（`spawn / onData / write / resize / kill` + pid/exitCode）——与 SQLite 驱动同一套"运行时差异隔离"手法：

| 文件 | 运行时 | 底层 |
|---|---|---|
| `pty.bun.ts` | Bun | `bun-pty` 的 `spawn` |
| `pty.node.ts` | Node | `@lydell/node-pty`（路径可经 `OPENCODE_NODE_PTY_PATH` 覆盖；Electron 主进程即此路） |
| `pty.workerd.ts` | workerd | 受限/替代实现 |

Node 侧的 `@lydell/node-pty` 是原生模块，跨运行时有兼容毛边——postinstall 的 `fix-node-pty` 脚本就是为它打的补丁。

### 持久层（`packages/core/src/persistent-pty/`）

**PersistentPty**：PTY 会话由后台守护进程持有、**不随客户端断开而死亡**——关掉 TUI 重开，终端里的进程还活着（tmux 的核心卖点之一）。`binary.{bun,node,workerd}.ts` 同样三平台分实现。

### 服务层（WebSocket 出口）

- server `handlers/pty.ts`：HTTP `ctx.request.upgrade` 把连接升级为 **WebSocket**（按键下行 + 输出上行，真双向——这就是 PTY 流走 WS 而非 SSE 的原因）
- `handlers/pty-socket.ts` 的 `runPtySocket`：单写者队列保证 replay 历史、实时输出、关闭帧的顺序
- **正因需要原生 WS（而非 fetch/SSE），`pty.connect` / `persistentPty.connect` 被排除在生成客户端之外**（protocol 的 omit 清单），由 `2.1_client/solid/pty.ts` 专门消费

### 使用场景

| 场景 | 入口 |
|---|---|
| TUI 会话的**终端面板**（`session-frame` 右侧 TerminalPane） | `packages/tui/src/context/session-terminals.tsx` |
| 桌面端内嵌终端 | Electron（`@lydell/node-pty`，VS Code 同款方案） |
| shell 命令执行（含 AI 发起的） | core `session/shell.ts`（用户 shell 后台执行、完成事件入 inbox） |

## 与其他技术的关系

- **WebSocket**：PTY 数据的传输通道（见 [websocket.md](8_websocket.md)）
- **守护进程架构**：PersistentPty 是"服务进程持有状态、瘦客户端随时重连"的又一体现（见 [../1_architecture/daemon-client.md](../2.6_architecture-styles/2_daemon-client.md)）
- **Electron / VS Code**：同为 node-pty 路线的先例（VS Code 集成终端即用它）
- **js-runtime**："同一能力按运行时拆实现"的范例（见 [js-runtime.md](1_js-runtime.md)）

## 相关源码

- `packages/core/src/pty/`（`pty.ts` 接口、三平台实现、`ticket.ts`）
- `packages/core/src/persistent-pty/`（daemon、三平台 binary）
- `packages/server/src/handlers/pty.ts`、`pty-socket.ts`、`persistent-pty.ts`
- `packages/client/src/solid/pty.ts`（WS 消费端）
- `packages/tui/src/context/session-terminals.tsx`
