# JS 运行时（JavaScript Runtime）与 Bun

> 本篇回答三个问题：**什么是"运行时"**、**运行时与操作系统的关系**、**运行时里面都有什么**——然后落到 Bun 与 Node.js 的对比，以及 opencode 为什么选 Bun。

---

## 一、什么是"运行时"（Runtime）

### 定义

**运行时 = 执行程序所需的完整软件环境**。对 JavaScript 来说，运行时就是"让 JS 代码真正跑起来的那套程序"：它读入你的代码，编译成机器码，分配内存，调度执行，并向你的代码提供"能力"（API）。

一句话：**语言只是规则，运行时才是让规则落地的机器**。

```ts
console.log("hello")
```

这行代码本身只是一串字符。是运行时做了全部工作：解析语法 → 编译 → 调用"标准库里的 console" → 向操作系统申请 stdout → 输出。

### 为什么需要运行时（对比编译型语言）

| | C / Rust | JavaScript |
|---|---|---|
| 编译产物 | 机器码（`.exe`），CPU 直接执行 | 字节码/源码，**需要宿主程序解释/JIT 执行** |
| 能力来源 | 操作系统 + libc | **运行时提供的 API**（文件、网络、定时器…） |
| 离开宿主 | 可独立运行 | 必须有运行时（浏览器、Node、Bun…） |

JS 生于浏览器：最初**只能**跑在浏览器这个宿主里，能力被限制在 DOM/定时器等。2009 年 Node.js 把 JS 引擎从浏览器里搬出来、接上操作系统能力（文件、进程、网络），JS 才有了浏览器之外的运行时。Bun（2022）、Deno（2020）是这条路的后来者。

### 同一语言的多个运行时

JS 是"一处定义（ECMAScript 规范），多处实现"：

| 运行时 | 引擎 | 出身 | 在 opencode 中的角色 |
|---|---|---|---|
| **浏览器**（Chrome/Safari/Firefox） | V8 / JavaScriptCore / SpiderMonkey | JS 的原生宿主 | Web 端、Electron 渲染进程 |
| **Node.js** | V8 | 第一个服务端运行时（2009） | Electron 主进程、VS Code 扩展宿主、node-pty 等原生模块生态 |
| **Bun** | JavaScriptCore | 三位一体工具链（运行时+包管理+测试） | **主力运行时**：TUI、后端服务、测试 |
| **workerd** | V8 | Cloudflare Workers 的运行时 | 云侧函数；opencode 甚至为它维护 `sqlite.workerd.ts` 驱动 |

**引擎 vs 运行时**：引擎（V8/JSC）只负责"执行 JS 代码"这一个内核；运行时 = 引擎 + 一整套宿主 API（`fs`、`http`、`crypto`…）+ 模块系统 + 事件循环。同款引擎可以出现在不同运行时里（V8 同时驱动 Chrome 与 Node；JSC 同时驱动 Safari 与 Bun）。

---

## 二、运行时与操作系统的关系

### 分层视角（含 CPU 特权级）

**关键修正一个常见误解**：运行时**不是**"用户态与内核态之间的桥梁"——它**完全运行在用户态内部**，是用户态里垫在你代码之下的一层。真正跨越用户态/内核态边界的是**系统调用（syscall）**；运行时只是把这些 syscall **封装**成友好 API，封装这个动作本身发生在用户态。

```
用户态（CPU 低特权级，ring 3）────────────────────────┐
│                                                      │
│  ┌─────────────────────────────────────────┐         │
│  │  你的代码（TS/JS）                        │  ← opencode 源码
│  ├─────────────────────────────────────────┤         │
│  │  运行时（Bun / Node / 浏览器沙箱）        │  ← 引擎+API+事件循环，仍是普通用户程序
│  └─────────────────────────────────────────┘         │
│                        │                              │
│                        │ 系统调用（syscall）─────── 唯一跨态通道（trap 进内核）
├────────────────────────┼─────────────────────────────┤
│  内核态（ring 0）      ▼                              │
│  操作系统（Linux / macOS / Windows）← 真正管硬件       │
├──────────────────────────────────────────────────────┤
│  硬件（CPU / 内存 / 磁盘 / 网卡）                      │
└──────────────────────────────────────────────────────┘
```

三个要点：

1. **运行时是用户态程序**：Bun/Node 本身就是一个普通可执行文件（`bun`、`node`），和你的代码共享同一个进程地址空间、同样的低特权级。它没有内核权限，崩了就是普通进程崩溃。
2. **跨态的只有 syscall**：`read()`/`write()`/`open()`/`socket()` 这些才是用户态→内核态的通道（CPU trap 指令切换特权级）。运行时做的事 = 在用户态把 syscall 包装成 `Bun.file()`/`fetch()`。
3. **"运行时在 OS 之上"的准确含义**：不是浮在内核旁边，而是"作为用户态进程，被 OS 调度、向 OS 发 syscall"——依赖关系意义上的"之上"，不是空间意义上的"之间"。

**运行时是"你代码与 OS 之间的翻译官"（但住在用户态家里）**：你的代码不直接碰系统调用，一切资源都要经过运行时转手——

> **城市基础设施比喻**：OS 是城市基建（道路/电网/水网）；syscall = 你自己去接线、开总阀；运行时 = 住在你家的水电工——帮你把线接好（封装 syscall）、提供插座开关（高级 API），但不替代市政府（内核），也住在你家里（用户态）。

- `Bun.file("x").text()` → 运行时（用户态）发 `open/read` syscall → 内核干活
- `fetch("https://…")` → 运行时发 socket syscall、走内核 TCP/IP 栈
- `setTimeout` → 运行时经 epoll/kqueue 向内核注册关注
- `new Worker()` → 运行时请求内核 clone 出线程

### 运行时替你做的四类事

1. **执行**：解析 → 字节码 → JIT 热点编译成机器码（V8 的 TurboFan、JSC 的 DFT/FTL）
2. **内存**：自动垃圾回收（分配堆、标记/整理/释放）；你从不手动 malloc/free
3. **并发调度（事件循环）**：JS 单线程执行，但运行时用**事件循环 + 异步 API** 榨出并发——IO 注册给 OS（epoll/kqueue/IOCP），完成后回调排队；重活丢给运行时的后台线程池（libuv 之于 Node；Bun 自带池 + Worker 线程）
4. **能力边界（安全沙箱）**：运行时决定你的代码"能干什么"。浏览器沙箱不给文件系统；workerd 每请求隔离；Bun/Node 给几乎全部系统能力——**能力是运行时给的，不是语言给的**

### 一次 `fetch()` 的完整旅程（感受分层与态边界）

```
你的 fetch() 调用                    ← 用户态：JS 代码
  → Bun API 层（校验、Promise 包装）   ← 用户态：运行时
    → 事件循环注册：DNS（后台线程）→ socket 建连
      → 【syscall: socket/connect/epoll_wait】 ←─ 唯一跨态点（trap 进内核）
        → 内核：驱动网卡、TCP/IP 栈、缓冲区     ← 内核态
          → 数据就绪 → epoll 唤醒（返回用户态）
            → 事件循环回调 → Promise resolve → 你的 .then() 执行  ← 用户态
```

内核负责真正的一切（进程调度、TCP/IP、文件系统），运行时在**用户态内**把内核能力翻译成 JS 世界观；每次真正的 IO 都要穿过一次 syscall 边界。

---

## 三、运行时里面都有什么（解剖 Bun/Node）

运行时 = **JS 引擎 + 标准库 + 事件循环 + 宿主适配** 的用户态组合体。以 Bun/Node 为例，拆开看六大件：

```
运行时 =
├── JS 引擎（解析执行 JS 代码）
│   ├── Bun → JavaScriptCore
│   └── Node → V8
├── 标准库（内置 API）
│   ├── fs（文件系统）/ http（网络）/ path / crypto / …
├── 事件循环（Event Loop）
│   └── 异步任务调度
└── 宿主适配
    ├── 浏览器版 → window / document
    └── 服务端版 → process / global（Bun/Node）
```

### 1. JS 引擎（内核）

- **Bun 用 JavaScriptCore**（Safari 引擎，C++）；Node 用 V8（Chrome 引擎）
- 职责：解析 → 解释执行 → JIT 编译热点代码；执行 GC；实现语言语义（闭包、原型链、异步函数）
- 引擎差异影响启动速度与峰值性能：JSC 启动快（移动端优化出身），V8 峰值吞吐强（桌面/服务端打磨）

### 2. 标准库与平台 API（能力的来源）

| 类别 | 例子（Bun） | 背后映射到 OS |
|---|---|---|
| 文件 | `Bun.file()`、`fs` | open/read/write/close |
| 网络 | `fetch`、`Bun.serve`、`http` | socket/epoll |
| 进程 | `Bun.spawn`、`child_process` | fork/exec/waitpid |
| 加密 | `crypto`（内置 OpenSSL/BoringSSL） | 系统密码学库 |
| 环境 | `process.env`、`os` | environ、uname |

**同一段 JS 在不同运行时能力不同**：`Bun.file()` 只在 Bun 有；`fs` 在浏览器没有；`window` 只在浏览器有。写跨运行时代码 = 只依赖交集（或做条件适配——opencode 的 SQLite 驱动正是这么干的：bun/node/workerd 三份实现）。

### 3. 事件循环（Event Loop）

JS 并发的心脏。"单线程 + 非阻塞 IO"的实现机制：

```
┌──────────────── 同步代码执行 ────────────────┐
│ 待处理定时器队列 │ 微任务队列 │ IO 回调队列 …  │
└───────────────────┬─────────────────────────┘
                    ▼ 循环轮询
        向 OS 询问"谁就绪了"（epoll/kqueue）
```

- 宏任务（setTimeout、IO 回调）与微任务（Promise.then）的执行顺序由循环决定
- Bun 与 Node 的事件循环实现不同（Node 是 libuv；Bun 自实现），语义兼容但时序细节有差异

### 4. 模块系统

- ES Module（`import/export`，语言标准）vs CommonJS（`require`，Node 历史）
- 运行时负责解析模块说明符 → 查找文件/包 → 加载执行 → 缓存
- **Bun 的关键差异**：原生执行 TS/JSX——`import "./app.ts"` 直接跑，因为加载器内置了转译（转译≠类型检查；类型检查另跑 `bun typecheck`）

### 5. 包管理器（Bun 独有的三位一体）

传统分工：运行时（node）+ 包管理（npm/pnpm）+ 测试（jest）是三个程序。**Bun 把三件事合成一个二进制**：`bun install`（lockfile 兼容策略自定）、`bun test`（Jest 兼容 API）、`bun run`。这是"工具链"而不仅是运行时——详见 [bun.md](2_bun.md)。

### 6. 附加能力（Bun 的扩展面）

- `Bun.serve()`：内置 HTTP 服务器（opencode 后端用它绑端口）
- `Bun.spawn()`：子进程（opencode 跑 shell 工具/PTY 用）
- Worker 线程、`Bun.password`、SQLite（`bun:sqlite`）等
- 原生插件钩子（`bun plugin`：opencode 的 Vite 通道就用 `build.module()` 把 TUI 入口注册为模块——运行时层面的模块替换）

---

## 四、Bun vs Node.js：opencode 为什么选 Bun

| 维度 | Node.js | Bun | opencode 的取舍 |
|---|---|---|---|
| 引擎 | V8（峰值吞吐强） | JavaScriptCore（启动快） | CLI 工具重启动速度——TUI/命令秒开 |
| TS 支持 | 需 `tsx`/编译 | **原生执行 TS** | 全仓"源码即运行"，无 build 步骤 |
| 工具链 | node+npm+jest 三件套 | 一体（install/test/run） | 单一依赖、单一版本约束（`packageManager: bun@1.4.2`） |
| API | 回调时代遗产 + 新 API 并存 | 现代优先 + Node 兼容层 | 新代码直接用现代 API（仓库约定：能用 Bun API 就用） |
| 生态 | 原生模块（node-pty 等）全兼容 | 兼容但偶有毛边 | 需要时打补丁：postinstall `fix-node-pty` |
| 稳定性 | 15+ 年 | 较新 | 桌面端 Electron 主进程仍是 Node；扩展宿主天然是 Node |

**兼容不是替换**：opencode 是"主路径 Bun、节点按需 Node"的混合——Electron（Node）、VS Code 扩展（Node 宿主）、workerd（云）各自保留，运行时差异被隔离在驱动层（如 `sqlite.bun.ts` / `sqlite.node.ts` / `sqlite.workerd.ts`）与 postinstall 补丁里。

---

## 五、快速记忆

- **引擎**执行代码（V8/JSC），**运行时** = 引擎 + 标准库 + 事件循环 + 宿主适配
- 运行时**完全在用户态内**：它是你代码与 OS 之间的翻译官，但跨态的只有它封装的 syscall
- 同一语言可以有多运行时（浏览器/Node/Bun/workerd），**API 集合各不相同**
- Bun = JSC 引擎 + 现代工具链三位一体；opencode 用它换"源码即运行 + 快启动"，Node 留在它不可替代的位置

## 相关源码

- 运行时差异隔离示例：`packages/core/src/database/`（三平台 SQLite 驱动）
- Bun 插件钩子用法：`packages/cli/dev/vite.ts`（`build.module` 注册 TUI 入口）
- postinstall 补丁：`packages/core` 的 `fix-node-pty`
- 根 `package.json`（`packageManager`、`bunfig.toml`）
