# WeChat iLink Bot 接入 OpenCode 架构方案

> 最后更新：2026-07-30 | 审查人：Sisyphus

## 目录

1. [概述](#1-概述)
2. [整体架构](#2-整体架构)
3. [模块设计](#3-模块设计)
4. [接入点](#4-接入点)
5. [数据流](#5-数据流)
6. [会话管理](#6-会话管理)
7. [控制命令](#7-控制命令)
8. [倾斜命令](#8-倾斜命令)
9. [选择流程](#9-选择流程)
10. [i18n](#10-i18n)
11. [日志与调试](#11-日志与调试)
12. [关键设计决策](#12-关键设计决策)
13. [文件清单](#13-文件清单)
14. [构建与部署](#14-构建与部署)

---

## 1. 概述

### 目标

通过 WeChat iLink Bot API（`https://ilinkai.weixin.qq.com`）将 AI 助手接入微信消息，实现：

- **扫码登录**：TUI 模式下终端 ASCII 二维码 / Desktop 侧栏 QR 码 / standalone 控制台，微信扫码完成认证
- **双向消息**：用户发微信消息 → Bot 调 OpenCode Session API → AI 回复 → 自动发送回微信
- **会话管理**：多用户、多会话，`@新建` / `@关闭` / `@切换` / `@列出` / `@帮助` 控制命令
- **倾斜命令**：`/goal` / `/task` / `/model` 等注册命令，与桌面/TUI 输入框行为一致
- **Agent/Model 选择**：首次连接自动默认配置（中文 + build），支持 `设置` 命令重新选择
- **终端集成**：TUI 内 `/wechat` 命令弹 dialog 展示 QR 码并实时更新扫码状态
- **桌面集成**：Electron utilityProcess 中自动启动，与桌面 App 生命周期绑定

### 运行环境

| 模式 | 进程 | 入口 |
|------|------|------|
| 桌面 (Desktop) | Electron utilityProcess | `packages/desktop/src/main/sidecar.ts` |
| TUI / CLI | Bun Worker | `packages/opencode/src/cli/tui/worker.ts` |
| 独立 (Standalone) | 独立进程 | `packages/channels/wechat/src/standalone.ts` |
| HTTP 服务 | OpenCode Server 内置 | `packages/opencode/src/cli/cmd/serve.ts` |

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        桌面 App                              │
│  ┌─────────────┐    IPC    ┌──────────────────────────────┐ │
│  │  Main Proc  │◄────────►│   Sidecar (utilityProcess)    │ │
│  │             │           │                              │ │
│  │ • show QR   │           │  startWechatAutoDetect()     │ │
│  │ • status    │           │    │                         │ │
│  └─────────────┘           │    ├─ check() 每10s          │ │
│                            │    │   ├─ validateToken()    │ │
│                            │    │   ├─ createClient()     │ │
│                            │    │   └─ createRunner()     │ │
│                            │    │                         │ │
│                            │    └─ Runner                 │ │
│                            │        ├─ doQrAuth()         │ │
│                            │        ├─ createBot()        │ │
│                            │        │   └─ pollLoop()     │ │
│                            │        ├─ onMessage()        │ │
│                            │        ├─ sendToAI()         │ │
│                            │        └─ handleSelection()  │ │
│                            └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
           │                              │
           │ iLink API                    │ HTTP (localhost)
           ▼                              ▼
  ┌─────────────────┐          ┌──────────────────────────┐
  │  WeChat iLink   │          │   OpenCode Server (v2)   │
  │                 │          │                          │
  │  • getupdates   │          │  POST /session           │
  │  • sendmessage  │          │  POST /session/{id}/     │
  │  • get_qrcode   │          │       message            │
  │  • get_config   │          │  POST /session/{id}/     │
  │                 │          │       command   ← 倾斜命令│
  └─────────────────┘          │  GET  /api/command       │
                               │  GET  /agent             │
                               │  GET  /config/providers  │
                               └──────────────────────────┘
```

### 分层结构

```
┌──────────────────────────────────┐
│         Runner (runner.ts)        │  ← 编排层：WechatClient 接口、策略注入
│  • 启动/停止                      │    CredentialStorage、LockStrategy
│  • QR 认证 (pollQrUntilDone)     │    倾斜命令路由 (/goal, /task, ...)
│  • 回调注入 (onQrCode, onReply)   │
├──────────────────────────────────┤
│         Bot (bot.ts)              │  ← 消息层：长轮询、分发、回复
│  • long-poll loop                 │
│  • processMessage (过滤/分派)     │
│  • sendReply (分片/发送)          │
├──────────────────────────────────┤
│     iLink API (ilink/)            │  ← 传输层：纯 HTTP 封装，浏览器安全
│  • auth.ts  (QR/验证/轮询)        │    btoa/atob，无不兼容 Node API
│  • api.ts   (getupdates/send)     │    sendMessage/sendTyping 返回校验
│  • types.ts (接口定义)            │
├──────────────────────────────────┤
│   Session Store (session.ts)      │  ← 映射层：user → session
│  • 控制命令解析 (@新建, @切换..)  │
│  • 多会话管理                     │
├──────────────────────────────────┤
│   i18n (i18n.ts)                  │  ← 展示层：中英文
│  • 字典式 key-value               │
│  • 参数替换 {{param}}             │
├──────────────────────────────────┤
│   SDK Bridge (sdk.ts)             │  ← 桥接层：平台无关的 WeChat SDK 客户端
│  • WechatChannelClient            │    getQrCode / pollQrStatus / getStatus
└──────────────────────────────────┘
```

---

## 3. 模块设计

### 3.1 Bot（`bot.ts`）— 消息引擎

**职责**：与 iLink API 交互，不依赖任何 OpenCode SDK。

**签名**：
```typescript
createBot({ token, onMessage, onControlCommand, onError }): BotInstance
```

**关键设计**：

- `onMessage` 和 `onControlCommand` 是**注入式回调**，Bot 不关心业务逻辑
- 退回策略：基础 1s，翻倍增长，上限 60s，成功后重置
- Token 过期（errcode -14）→ 通知 Runner → 触发重登录
- 长消息自动分片：`splitText(text, 2048)` → 微信单条消息限制
- 使用 discriminated union 提取消息文本（`type: 1` + `text_item`）

### 3.2 Runner（`runner.ts`）— 编排中心（748 行）

**职责**：连接所有组件，管理生命周期，协调 Bot 与 OpenCode 服务端。

```typescript
createWechatRunner({ client, storage, lock?, onQrCode, ... }): WechatRunner
```

#### WechatClient 接口（SDK 解耦）

```typescript
export interface WechatClient {
  session: {
    create(input: { body: { title: string } }): Promise<{ error?: unknown; data?: { id: string } }>
    prompt(input: {
      path: { id: string }
      body: { parts: Array<{ type: "text"; text: string }>; agent: string; model: {...} }
    }): Promise<{ error?: unknown; data?: { parts: Array<{ type: string; text?: string }> } }>
    command(input: {
      path: { id: string }
      body: { command: string; arguments: string; agent?: string; model?: string }
    }): Promise<{ error?: unknown; data?: unknown }>
  }
  command: {
    list(): Promise<{ error?: unknown; data?: Array<{ name: string }> }>
  }
  app: {
    agents(): Promise<{ error?: unknown; data?: Array<{ name: string; mode: string }> }>
  }
  config: {
    providers(): Promise<{ error?: unknown; data?: { providers: Array<{ id: string; name: string; models: Record<string, { name?: string }> }> } }>
  }
}
```

`WechatClient` 是**结构接口**，不依赖 `@opencode-ai/sdk`。consumer（standalone/sidecar/worker/serve）传入真实的 `OpencodeClient` — 它满足此接口的结构约定。

#### 策略注入

```typescript
interface CredentialStorage {     // 凭证存在哪？
  load(): Promise<BotCredentials | null>
  save(creds: BotCredentials): Promise<void>
  delete(): Promise<void>
}

interface LockStrategy {          // 是否防多实例？
  acquire(): Promise<boolean>
  release(): Promise<void>
}
```

- **桌面模式**：`fsCredentialStorage()` + 无锁（sidecar `checking` 互斥）
- **独立模式**：`fsCredentialStorage()` + `flockLock()`
- **浏览器模式**：`custom CredentialStorage`（localStorage / 服务端 API）

#### fsCredentialStorage 凭证校验

```typescript
// 逐字段 typeof 校验，不是 as BotCredentials
const obj = parsed as Record<string, unknown>
if (typeof obj.botToken !== "string" || !obj.botToken) return null
if (typeof obj.userId !== "string" || !obj.userId) return null
return {
  botToken: obj.botToken,
  botId: typeof obj.botId === "string" ? obj.botId : "",
  userId: obj.userId,
  baseUrl: typeof obj.baseUrl === "string" ? obj.baseUrl : "https://ilinkai.weixin.qq.com",
}
```

#### 凭证文件路径

所有模式统一存储在：
```
~/.local/share/opencode/ilink_token.json
```
（`$XDG_DATA_HOME/opencode/ilink_token.json`，若 XDG_DATA_HOME 未设置则 fallback 到 `~/.local/share`）

格式：
```json
{
  "botToken": "xxxxxxxxxxxxxx",
  "botId": "xxxxxxxxxxxxxx",
  "userId": "xxxxxxxxxxxxxx",
  "baseUrl": "https://ilinkai.weixin.qq.com"
}
```
权限：`0600`（仅所有者可读写）。删除此文件即可重新扫码。

### 3.3 iLink API（`ilink/`）— 传输层

#### `auth.ts` — 纯 HTTP 认证（浏览器安全）

**设计原则**：auth.ts 只包含 HTTP 请求逻辑，不含文件 I/O。所有凭证持久化由 runner.ts 的 `CredentialStorage` 接口负责。

| 函数 | 方法 | 说明 |
|------|------|------|
| `requestQrCode()` | GET `/ilink/bot/get_bot_qrcode` | 获取 QR 码数据 |
| `validateToken(token)` | POST `/ilink/bot/getupdates` | 验证 token 有效性 |
| `pollQrStatus(qrSession)` | POST `/ilink/bot/check_status` | 单次轮询 QR 扫码状态 |
| `pollQrUntilDone(qrSession, opts?)` | 轮询循环 | 统一轮询入口，默认 180 次 × 1s 间隔 |
| `waitForScan(qrSession)` | 委托到 pollQrUntilDone | 向后兼容 |
| `generateTerminalQr(base64)` | 本地生成 | ASCII 终端二维码 |

**导出类**：
- `QrExpiredError` — QR 过期时抛出
- `QrTimedOutError` — 轮询超时时抛出

`validateToken` 特殊处理：iLink `getupdates` API 成功时可能不含 `ret`/`errcode` 字段：

```typescript
if (!resp.ok) return false
const json = JSON.parse(text) as { ret?: number; errcode?: number }
if (json.ret !== undefined) return json.ret === 0
if (json.errcode !== undefined) return json.errcode === 0
return true  // ← HTTP 200 且无错误字段 = 有效
```

#### `api.ts` — 消息通道

| 函数 | 方法 | 超时 | 说明 |
|------|------|------|------|
| `getUpdates(cursor, token)` | POST `/ilink/bot/getupdates` | 35s | 长轮询新消息 |
| `sendMessage(msg, token)` | POST `/ilink/bot/sendmessage` | 15s | 发送回复 |
| `getConfig(userId, ctxToken, token)` | POST `/ilink/bot/getconfig` | 10s | 获取配置 |
| `sendTyping(ticket, status, token)` | POST `/ilink/bot/sendtyping` | 10s | 发送"正在输入" |

**返回校验**：`sendMessage` 和 `sendTyping` 都检查 `ret`/`errcode`：
```typescript
const result = await apiPost<{ ret?: number; errcode?: number }>("sendmessage", body, token, 15_000)
if (result.ret !== undefined && result.ret !== 0) throw new Error(`sendmessage failed: ret=${result.ret}`)
if (result.errcode !== undefined && result.errcode !== 0) throw new Error(`sendmessage failed: errcode=${result.errcode}`)
```

**浏览器安全**：`randomWechatUin()` 使用 `btoa()` 而非 `Buffer.from()`。

**公共请求头**：

```
AuthorizationType: ilink_bot_token
Authorization: Bearer <bot_token>
X-WECHAT-UIN: <random uint32 base64>
Content-Type: application/json
Content-Length: <body length>
```

### 3.4 Session Store（`session.ts`）— 会话映射

**数据结构**：

```
sessions: Map<userId, UserSession[]>    ← 每个用户一组会话
active:   Map<userId, sessionId>        ← 每个用户当前活跃会话
```

**UserSession**：

```typescript
interface UserSession {
  userId: string
  sessionId: string         // OpenCode session ID
  contextToken: string       // iLink 上下文令牌 (用于 sendmessage)
  title: string              // 显示名称
  createdAt: number          // 创建时间戳
}
```

**操作**：

| 方法 | 说明 |
|------|------|
| `getActive(userId)` | 获取当前活跃会话 |
| `setActive(userId, session)` | 设置活跃会话（新增或更新） |
| `list(userId)` | 列出所有会话 |
| `remove(userId, sessionId)` | 删除会话（`@关闭`） |
| `updateToken(userId, token)` | 更新上下文令牌（每条消息调用） |

**控制命令解析**：

```
parseControlCommand(text)
  ├─ "@新建" | "@new"      → { type: "new_session" }
  ├─ "@关闭" | "@close"     → { type: "close_session" }
  ├─ "@列出" | "@list"      → { type: "list_sessions" }
  ├─ "@切换 <N>" | "@switch <N>" → { type: "switch_session", target: "<N>" }
  ├─ "@帮助" | "@help"      → { type: "help" }
  └─ 其他                   → null (普通消息)
```

### 3.5 i18n（`i18n.ts`）— 国际化

扁平 key-value 字典，支持 `{{param}}` 占位符。缺失 key 回退：中文不存在 → 查英文 → 返回 key 本身。

### 3.6 日志（`log.ts`）— 浏览器安全

**设计原则**：模块在首次 `log()` 调用时懒初始化，非 Node 环境自动降级为 noop。所有 Node.js 依赖通过 `import()` 动态加载。

```
~/.local/share/opencode/logs/wechat.log    ← 仅文件写（浏览器安全）
```

格式：`[ISO时间戳] [级别] 消息`

- `wechatLog.info(msg)` — 正常流程
- `wechatLog.warn(msg)` — 可恢复异常
- `wechatLog.error(msg, err?)` — 错误

**静默吞错修复**：全项目 11 处 `.catch(() => {})` → `.catch(err => wechatLog.warn(...))`

### 3.7 SDK Bridge（`sdk.ts`）— 平台无关客户端

```typescript
class WechatChannelClient {
  constructor(baseUrl: string, headers?: Record<string, string>)
  getQrCode(): Promise<WechatQrCode>
  pollQrStatus(token: string): Promise<WechatQrStatus>
  disconnect(): Promise<void>
  getStatus(): Promise<{ connected: boolean }>
}
```

基于 `fetch`，无 Node.js 依赖，可在 Electron 渲染进程 / 浏览器 / Node.js 中使用。

---

## 4. 接入点

### 4.1 桌面（`packages/desktop/src/main/sidecar.ts`）

**位置**：Electron 主进程 fork 出的 utilityProcess。

**启动时机**：OpenCode Server 启动完成后，主进程通过 IPC 发送 `start` 消息，携带 `hostname`、`port`、`password`。

**IPC 消息**：
- 下行（主进程 → sidecar）：`{ type: "start" }` / `{ type: "stop" }`
- 上行（sidecar → 主进程）：`{ type: "ready" }` / `{ type: "stopped" }` / `{ type: "error" }` / `{ type: "wechat.qrcode" }` / `{ type: "wechat.status" }`

```typescript
// runner 回调 → 主进程
onQrCode: (base64) => parentPort.postMessage({ type: "wechat.qrcode", base64 })
onStatusChange: (status) => parentPort.postMessage({ type: "wechat.status", status })
```

**关键设计**：

- `checking` 互斥：防止 `setInterval(check, 10000)` 和首次慢 `check()` 并发创建两个 Runner
- `Authorization` 头：`Basic opencode:<password>`
- 回调通过 `parentPort.postMessage` 发送给主进程（渲染进程可消费）

### 4.2 TUI / CLI（`packages/opencode/src/cli/tui/worker.ts`）

**位置**：Bun Worker 线程。

**启动流程**：

```
Worker 线程启动
  │
  ├─ Server.listen({ port: 0 })   ← 自动启动内部 HTTP 服务（随机端口）
  ├─ tryAutoStartWechat()         ← 立即执行 + 每 3 秒轮询
  │   ├─ starting 互斥锁
  │   ├─ fsCredentialStorage().load()
  │   ├─ validateToken()
  │   └─ createWechatRunner({ client, storage })
  │
  └─ 主进程 /wechat 命令
      ├─ 加载 wechat-channel 插件
      ├─ fsCredentialStorage() 检查凭证
      │   ├─ 有效 → 弹 dialog "Already connected"
      │   └─ 过期/不存在 → 弹 dialog 显示 QR 码 (ASCII)
      └─ 扫码流程
          ├─ requestQrCode()
          ├─ generateTerminalQr()
          ├─ pollQrUntilDone(qr, opts)
          └─ storage.save(creds)
```

**与 Desktop 的差异**：

| 方面 | Desktop | TUI |
|------|---------|-----|
| QR 码展示 | 侧栏渲染（通过 IPC 发 base64） | 终端 ASCII + dialog 居中 |
| 并发防护 | `checking` 互斥 | `starting` 互斥 |
| SDK 认证 | `Basic opencode:<password>` | `ServerAuth.header()` |
| 回调 | `parentPort.postMessage` | TUI dialog 内联渲染 |

### 4.3 HTTP Serve（`packages/opencode/src/cli/cmd/serve.ts`）

**位置**：`opencode serve` 命令启动的无头服务。

**启动流程**：服务监听后自动调用 `startWechatAutoDetect(serverUrl)`，加载凭证 → 验证 → 创建 Runner → 启动 Bot。回调实现为 `console.log` / `console.error`。

### 4.4 HTTP API（`packages/opencode/src/server/routes/instance/httpapi/server.ts`）

**路由**：

```
POST /api/wechat/qrcode      → 返回 QR 码 base64
POST /api/wechat/status      → 返回连接状态
POST /api/wechat/send        → 通过 Bot 发送消息
```

复用 `auth.ts` / `runner.ts` 的公共函数（`requestQrCode`、`pollQrStatus` 等），无内联 `ILINK_BASE` 或重复 fetch。

---

## 5. 数据流

### 5.1 消息接收与回复（普通文本）

```
WeChat 用户发送 "你好"
  │
  ▼
iLink getupdates 返回
  {
    msgs: [{
      seq: 36,
      message_type: 1,          ← 用户消息
      from_user_id: "o9cq80-3...",
      context_token: "...",
      item_list: [{ type: 1, text_item: { text: "你好" } }]
    }]
  }
  │
  ▼
Bot.pollLoop → processMessage(msg)
  │
  ├─ message_type !== 1 → skip (过滤 bot 回声)
  ├─ 提取文本 (discriminated union narrowing)
  ├─ parseControlCommand("你好") → null (普通消息)
  │
  └─ onMessage(msg) → Runner.onMessage()
      │
      ├─ updateToken(userId, context_token)
      ├─ seq 去重检查 (seenMessages)
      ├─ 检查 selecting[userId] → 不在选择流程中
      ├─ 检查 isExitText → 不是
      ├─ 检查 "设置" → 不是
      ├─ 检查 userPrefs[userId] → 没有 (首次)
      │
      ├─ fetchAgents() → ["build", ...]
      ├─ fetchModels() → [{ label: "DeepSeek V4 Flash", ... }]
      ├─ 创建 pref = { locale: "zh", agent: "build", model: {...} }
      │
      └─ sendToAI(userId, msg, "你好", pref)
          │
          ├─ parseSlash("你好") → null (非斜杠命令)
          ├─ getActive(userId) → null (无活跃会话)
          ├─ POST /session { body: { title: "WeChat - o9cq80-3" } }
          ├─ setActive(userId, session)
          │
          ├─ POST /session/{id}/message
          │   body: {
          │     parts: [{ type: "text", text: "你好" }],
          │     agent: "build",
          │     model: { providerID: "deepseek", modelID: "deepseek-v4-flash" }
          │   }
          │   → { parts: [{ type: "text", text: "你好！有什么可以帮你的？" }] }
          │
          └─ 返回 responseText → Runner 组装 footer
              → "你好！有什么可以帮你的？\n\n已自动配置：中文 / build / DeepSeek V4 Flash..."
  │
  ▼
Bot.sendReply()
  └─ POST /ilink/bot/sendmessage
       body: {
         msg: {
           to_user_id: "o9cq80-3...",
           context_token: "...",
           message_type: 2,          ← Bot 消息
           message_state: 2,
           item_list: [{ type: 1, text_item: { text: "你好！有什么..." } }]
         }
       }
  │
  ▼
用户微信收到回复
```

### 5.2 倾斜命令路由

```
WeChat 用户发送 "/goal 今天的目标是完成代码审查"
  │
  ▼
Runner.onMessage()
  │
  └─ sendToAI(userId, msg, "/goal 今天的目标是...", pref)
      │
      ├─ parseSlash("/goal 今天的目标是...")
      │   → { command: "goal", args: "今天的目标是完成代码审查" }
      │
      ├─ isExitText → no
      ├─ cmd.command !== "new" → skip /new 分支
      │
      ├─ getKnownCommands()  ← 从 /api/command 获取注册命令列表
      │   → Set { "goal", "task", "model", "agent", ... }
      │
      ├─ known.has("goal") → YES
      │
      ├─ ensure session (不存在则创建)
      │
      ├─ POST /session/{id}/command
      │   body: { command: "goal", arguments: "今天的目标是完成代码审查" }
      │   → { parts: [{ type: "text", text: "目标已设定为: 今天的目标是完成代码审查" }] }
      │
      └─ 返回 responseText
```

**命中规则**：

```
微信文本 → parseSlash(text)
  ├─ null (无 "/")                   → session.prompt()  (普通消息)
  ├─ "/exit" | "/quit" | ":q"        → 清理状态、断连
  ├─ "/new"                          → 强制新建 session
  ├─ "/xxx" + 已知命令(注册过)        → session.command()
  └─ "/xxx" + 未知命令(未注册)        → session.prompt()  (当作普通文本)
```

### 5.3 去重机制

**问题**：iLink 同一消息可能被两次 poll 返回（`seq` 相同，时间差约 3-10s）。

**方案**：`seenMessages: Set<string>` 记录已处理的 `seq`。ES2015 保证 Set 的插入顺序 — `values().next()` 返回最旧条目。

```typescript
const key = String(msg.seq ?? 0)
if (key !== "0" && seenMessages.has(key)) return { text: "" }
if (key !== "0") {
  if (seenMessages.size >= DEDUP_CAPACITY) {  // DEDUP_CAPACITY = 5000
    const oldest = seenMessages.values().next().value
    if (oldest !== undefined) seenMessages.delete(oldest)
  }
  seenMessages.add(key)
}
```

**为什么不用 `message_id`**：iLink 的 `message_id` 超过 JS Number 安全整数范围，`String()` 转换丢失精度。

---

## 6. 会话管理

### 6.1 会话生命周期

```
                   新建会话
                      │
     ┌────────────────┼────────────────┐
     │                │                │
   发消息           @新建            @关闭
     │          (或 /new)              │
     ▼                │                ▼
  自动创建         创建新会话        删除当前会话
  (首次)           setActive()      remove()
     │                │                │
     └──── setActive()                 │
          │                            │
          ▼                            ▼
      活跃会话                    下次发消息
          │                      自动创建新会话
          │
     @切换 <N>
          │
          ▼
     切换到会话 N
```

### 6.2 `sendToAI` 流程（含命令路由）

```typescript
async function sendToAI(userId, msg, text, pref) {
  const cmd = parseSlash(text)

  // Branch 1: /new → 强制新建 session
  if (cmd?.command === "new") {
    sessions.remove(userId, old.sessionId)
    const session = await createSession(userId)
    sessions.setActive(userId, session)
    return { text: "新会话已创建 (xxx)..." }
  }

  // Branch 2: 已知倾斜命令 → session.command()
  if (cmd) {
    const known = await getKnownCommands()  // 缓存
    if (known.has(cmd.command)) {
      const session = await ensureSession(userId)
      const result = await client.session.command({
        path: { id: session.sessionId },
        body: { command: cmd.command, arguments: cmd.args },
      })
      return formatResponse(result)
    }
    // 未知命令 → fall through to prompt
  }

  // Branch 3: 普通文本 → session.prompt()
  const session = await ensureSession(userId)
  const promptResult = await client.session.prompt({
    path: { id: session.sessionId },
    body: {
      parts: [{ type: "text", text }],
      agent: pref.agent,
      model: pref.model,
    },
  })
  return formatResponse(promptResult)
}
```

**设计理由**：`agent` 和 `model` 每次 prompt 传递（而非 session 创建时设定），允许同一 session 内切换 agent/model。

---

## 7. 控制命令

### 7.1 命令表（`@` 前缀）

| 命令 | 别名 | 行为 |
|------|------|------|
| `@新建` | `@new` | 创建新 OpenCode session，设为活跃 |
| `@关闭` | `@close` | 删除当前活跃 session，下次发消息自动创建新 session |
| `@切换 <N>` | `@switch <N>` | 切换到第 N 个 session（`@列出` 中的序号） |
| `@列出` | `@list` | 列出所有 session，★ 标记当前活跃 |
| `@帮助` | `@help` | 显示命令列表和使用说明 |

### 7.2 命令处理流程

```
用户发送 "@关闭"
  │
  ▼
Bot.processMessage()
  ├─ parseControlCommand("@关闭")
  │   → { type: "close_session" }
  │
  └─ onControlCommand(cmd, userId)
      └─ Runner.handleControlCommand()
          ├─ sessions.getActive(userId) → session
          ├─ sessions.remove(userId, session.sessionId)
          └─ return t("wechat.cmd.closed")
```

---

## 8. 倾斜命令

### 8.1 概述

倾斜命令 (`/goal`, `/task`, `/model` 等) 与 TUI/Desktop 输入框行为完全一致。用户通过微信发送后，与在 TUI 输入 `/goal xxx` 效果相同。

**关键设计**：命令通过 **服务端注册命令列表**（`/api/command`）校验。未知命令不会错误拒绝，而是静默降级为普通 prompt（AI 自行解读）。

### 8.2 命令路由

```
微信文本 → parseSlash()
  ├─ /exit /quit :q  → 客户端断连（不调服务端 API）
  ├─ /new             → 客户端强制新 session（不调服务端 API）
  ├─ /goal /task 等   → getKnownCommands()?
  │   ├─ 已注册       → session.command({ command: "goal", arguments: "..." })
  │   └─ 未注册       → session.prompt() （当作普通文本）
  └─ 无 "/"           → session.prompt()
```

### 8.3 缓存

命令列表通过 `fetchCommands()` 获取，首次成功后存入 `Set<string>` 全局缓存。如需刷新，重启 runner 即可。

### 8.4 与 TUI/Desktop 一致

| | TUI/Desktop | WeChat 微信 |
|---|---|---|
| 普通文本 | `session.prompt()` | `session.prompt()` ✅ |
| `/goal xxx` | `session.command()` | `session.command()` ✅ |
| `/new` | 客户端新建 session | 客户端新建 session ✅ |
| `/exit` `/quit` | 客户端退出 | 客户端断连 ✅ |
| Shell 模式 `!xxx` | `session.shell()` | ❌ 不支持（微信无终端） |

---

## 9. 选择流程

### 9.1 状态机

```
┌─────────┐   "设置"   ┌─────────┐   选 1/2   ┌─────────┐   选 N    ┌─────────┐   选 N    ┌──────┐
│  IDLE   │──────────►│  LOCALE  │──────────►│  AGENT   │──────────►│  MODEL   │──────────►│ DONE │
└─────────┘           └─────────┘           └─────────┘           └─────────┘           └──────┘
                           │                     │                     │
                           ▼                     ▼                     ▼
                    pendingLocale          pendingAgent          userPrefs.set()
                       .set()                .set()
```

**状态存储**：

```typescript
const selecting     = new Map<string, "locale" | "agent" | "model" | "done">()
const pendingLocale = new Map<string, "zh" | "en">()
const pendingAgent  = new Map<string, string>()
const userPrefs     = new Map<string, UserPref>()
```

### 9.2 步骤详情

| 步骤 | 提示 | 用户输入 | 下一状态 | 副作用 |
|------|------|----------|----------|--------|
| locale | "请选择语言：1. 中文 2. English" | 1 或 2 | agent | `pendingLocale.set(userId, loc)` |
| agent | "请选择 Agent：1. build 2. compaction ..." | 数字 N | model | `pendingAgent.set(userId, agent)` |
| model | "请选择模型：1. DeepSeek V4 Flash ..." | 数字 N | done | `userPrefs.set(userId, {...})`，清空 pending state |
| done | — | 任意消息 | — | 进入正常对话 |

**i18n**：错误提示使用正确的 locale（`pendingLocale ?? userPrefs.locale ?? "zh"`）。

---

## 10. i18n

### 10.1 Key 清单

| Key | 中文 | English |
|-----|------|---------|
| `wechat.lang.select` | 请选择语言： | Please select language: |
| `wechat.lang.chosen` | 已选择：中文 | Selected: English |
| `wechat.agent.select` | 请选择 Agent（回复数字）： | Select an Agent (reply with number): |
| `wechat.agent.chosen` | 已选择 Agent: {{agent}} | Selected Agent: {{agent}} |
| `wechat.agent.invalid` | 请输入 1-{{max}} 之间的数字。 | Please enter a number between 1-{{max}}. |
| `wechat.model.select` | 请选择模型（回复数字）： | Select a model (reply with number): |
| `wechat.model.chosen` | 已选择：Agent: {{agent}} / 模型: {{model}} | Selected: Agent: {{agent}} / Model: {{model}} |
| `wechat.model.unavailable` | 暂无可用模型，请稍后再试。 | No models available. |
| `wechat.model.invalid` | 请输入 1-{{max}} 之间的数字。 | Please enter a number between 1-{{max}}. |
| `wechat.input.invalid` | 请输入有效数字。 | Please enter a valid number. |
| `wechat.cmd.created` | 已创建新会话: {{id}}... | New session created: {{id}}... |
| `wechat.cmd.create_failed` | 创建会话失败: {{error}} | Failed to create session: {{error}} |
| `wechat.cmd.closed` | 已关闭当前对话。下次发消息将自动创建新对话。 | Session closed. Next message will create a new session. |
| `wechat.cmd.close_no_session` | 当前没有活动对话。 | No active session to close. |
| `wechat.cmd.switched` | 已切换到: {{title}} ({{id}}...) | Switched to: {{title}} ({{id}}...) |
| `wechat.cmd.switch_not_found` | 未找到会话 "{{target}}" | Session "{{target}}" not found. |
| `wechat.cmd.no_sessions` | 暂无会话，发送消息即可创建。 | No sessions yet. |
| `wechat.cmd.list_header` | 你的会话: | Your sessions: |
| `wechat.cmd.help` | 可用命令: @新建 / @关闭 / @切换 / @列出 / @帮助 | Commands: @new / @close / @switch / @list / @help |
| `wechat.first.auto_defaults` | 已自动配置：{{lang}} / {{agent}} / {{model}}... | Auto-configured: {{lang}} / {{agent}} / {{model}}... |

---

## 11. 日志与调试

### 11.1 日志

**设计**：仅文件写。首次调用时懒初始化，非 Node 环境不抛错（module 可被浏览器 import）。

```
~/.local/share/opencode/logs/wechat.log
```

格式：`[ISO时间戳] [级别] 消息`

| 级别 | 场景 |
|------|------|
| INFO | 正常流程：poll 结果、消息收发、session 创建、命令路由 |
| WARN | 可恢复异常：token 失效需重登、空回复、凭证保存失败 |
| ERROR | 需要关注的错误：API 调用失败、SDK 返回错误 |

### 11.2 关键日志点

```
runner start() called                                    ← Runner 开始启动
runner storage.load() result: token=xxx...               ← 凭证加载
runner saved token valid: true                            ← Token 验证
fetchAgents: filtered=build,compaction,...                ← Agent 列表
fetchCommands: goal,task,model,agent                      ← 命令列表
onMessage from o9cq80-3 text=/goal 今天的目标...          ← Runner 接收
sending command to session ses_abc123: /goal 今天...       ← 命令路由
command response: 目标已设定为...                          ← 命令响应
sending prompt to session ses_abc123: 你好                ← 普通 prompt
session created: ses_abc123                              ← Session ID
prompt response: 你好！有什么可以帮你的？                   ← AI 回复
```

### 11.3 启动问题排查

| 症状 | 检查点 |
|------|--------|
| 无 wechat.log | 首次 `log()` 调用是否触发懒初始化 |
| 无 `runner start()` | 桌面：sidecar 是否收到 `start` IPC；TUI：`tryAutoStartWechat` 是否被调用 |
| `token invalid` | QR 是否过期，`~/.local/share/opencode/ilink_token.json` 是否存在 |
| `session create failed` | OpenCode Server 是否运行，端口/密码是否正确 |
| 重复回复 | `sidecar check` 是否并发执行（应只有一条 `bot started OK`） |
| 倾斜命令不生效 | 检查 `/api/command` 返回列表是否包含该命令 |
| 倾斜命令报错 | 检查 command body 字段名是否与 v2 SDK 匹配 |

---

## 12. 关键设计决策

### 12.1 SDK 解耦（WechatClient 接口）

**问题**：直接依赖 `@opencode-ai/sdk` 导致 wechat-channel 包与 OpenCode SDK 版本强绑定，且 TypeScript 类型过于复杂。

**决策**：定义 `WechatClient` 结构接口，只声明实际使用的 5 个方法（`session.create/prompt/command`、`command.list`、`app.agents`、`config.providers`）。consumer 传入真实的 `OpencodeClient`，它满足接口的结构约定。`@opencode-ai/sdk` 从 `package.json` 依赖中移除。

### 12.2 凭证存储统一

**问题**：`auth.ts` 混入文件 I/O（`loadCredentials/persistCredentials/clearCredentials`），模块无法在浏览器中 import。

**决策**：auth.ts 变为纯 HTTP 函数（浏览器安全）。所有凭证操作统一到 `CredentialStorage` 接口（runner.ts），由调用方注入实现（`fsCredentialStorage()` for Node/Bun）。

### 12.3 QR 轮询统一

**问题**：runner.ts、server.ts、wechat-channel.tsx 各有自己的 QR 轮询逻辑，参数不一致。

**决策**：auth.ts 新增 `pollQrUntilDone()`（可配参数，默认 180×1s）+ `QrExpiredError`/`QrTimedOutError`。所有调用方统一使用此函数。`waitForScan` 保留作向后兼容委托。

### 12.4 Agent/Model 每次 Prompt 传递

**问题**：Session 创建时指定 agent/model 还是 prompt 时指定？

**决策**：prompt 时传递。v2 SDK `SessionPromptData` body 支持 `agent` 和 `model` 字段，允许同一 session 中切换 agent。

### 12.5 `seq` 去重（非 `message_id`）

**问题**：`message_id` 超过 JS Number 安全整数 → 去重失效。

**决策**：使用 `msg.seq`（小整数）去重。Set 插入顺序保证淘汰最旧条目。

### 12.6 Sidecar 并发防护

**问题**：`setInterval(check, 10000)` 触发时首次 `check()` 可能仍在执行（网络慢）→ 创建第二个 Runner → 重复消息。

**决策**：`let checking = false` 互斥变量。

### 12.7 日志浏览器安全

**问题**：顶层 `import { appendFileSync } from "node:fs"` 在浏览器/Electron 渲染进程中崩溃。

**决策**：`log.ts` 使用懒 `import()`，首次 `log()` 调用时初始化。非 Node 环境降级为 noop。

### 12.8 静默吞错修复

**问题**：多处以 `.catch(() => {})` 吞掉错误，排查困难。

**决策**：全项目 11 处改为 `.catch(err => wechatLog.warn(...))`，保留上下文信息。

---

## 13. 文件清单

### 13.1 新增文件（`packages/channels/wechat/`）

```
packages/channels/wechat/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              ← 公共导出
│   ├── runner.ts             ← 编排中心 (748 行)
│   ├── bot.ts                ← 消息引擎 (227 行)
│   ├── session.ts            ← 会话映射 + 控制命令解析 (146 行)
│   ├── i18n.ts               ← 国际化字典 (102 行)
│   ├── log.ts                ← 日志模块 (39 行，浏览器安全)
│   ├── sdk.ts                ← 平台无关 WeChat SDK 客户端 (50 行)
│   ├── standalone.ts         ← 独立运行入口
│   └── ilink/
│       ├── auth.ts           ← 纯 HTTP QR 认证 (238 行)
│       ├── api.ts            ← iLink HTTP API 封装 (160 行)
│       └── types.ts          ← 类型定义
```

### 13.2 修改文件（现有项目）

```
packages/desktop/
├── package.json              ← +@opencode-ai/wechat-channel
├── electron.vite.config.ts   ← externalizeDeps.exclude
└── src/main/
    └── sidecar.ts            ← startWechatAutoDetect() + IPC (222 行)

packages/opencode/
└── src/
    ├── cli/tui/worker.ts     ← Server.listen + tryAutoStartWechat (101 行)
    ├── cli/cmd/serve.ts      ← startWechatAutoDetect + console callbacks (68 行)
    └── server/routes/instance/httpapi/
        └── server.ts         ← wechat HTTP 路由 (去重)

packages/tui/
├── package.json              ← +@opencode-ai/wechat-channel
└── src/feature-plugins/system/
    └── wechat-channel.tsx     ← /wechat 命令 + QR dialog (161 行)
```

---

## 14. 构建与部署

### 14.1 本地测试

```bash
cd packages/channels/wechat
bun run src/standalone.ts
# 终端显示 QR 码 → 微信扫码 → 开始对话
```

### 14.2 桌面构建

```bash
cd packages/opencode
./script/build.ts              # 编译 OpenCode 二进制
cd ../desktop
./build.sh                     # 打包桌面 App
```

### 14.3 运行环境要求

| 组件 | 版本 | 说明 |
|------|------|------|
| Bun | ≥1.2 | 运行时 + 编译器 |
| Node.js | ≥22 | Electron 内嵌 |
| WeChat | 最新版 | iLink Bot 扫码 |

### 14.4 凭证文件

**路径**（所有运行模式统一）：

```
~/.local/share/opencode/ilink_token.json
```

或 `$XDG_DATA_HOME/opencode/ilink_token.json`（若 `XDG_DATA_HOME` 已设置）。

**格式**：

```json
{
  "botToken": "xxxxxxxxxxxxxx",
  "botId": "xxxxxxxxxxxxxx",
  "userId": "xxxxxxxxxxxxxx",
  "baseUrl": "https://ilinkai.weixin.qq.com"
}
```

权限：`0600`（仅所有者可读写）。

**操作**：

| 操作 | 方法 | 说明 |
|------|------|------|
| 删除重连 | `rm ~/.local/share/opencode/ilink_token.json` | 删除后下次自动弹 QR 重新扫码 |
| TUI 内断开 | `/wechat` → `Disconnect WeChat` | 调用 `storage.delete()` 删除文件 |
| 微信内断开 | 发送 `/exit` | Runner 清理状态 + 删除本地凭证 |

---

## 15. 安全与可靠性改进（2026-07-30）

### 15.1 WeChat API 路由认证保护

**问题**：`/api/channels/wechat/*` 四个端点通过 `HttpRouter.use()` 注册，未包裹认证中间件。仅依赖 `fenceLayer`（对 GET 请求不校验），存在未授权访问风险。

**修复**（`packages/opencode/.../server.ts`）：
```typescript
const wechatRoutes = HttpRouter.use((router) =>
  Effect.gen(function* () {
    // ... 4 个路由
  }),
).pipe(Layer.provide(authOnlyRouterLayer))  // ← 新增
```

现在 wechat 路由与 `instanceRoutes`/`serverRoutes` 使用相同的 Basic Auth 认证，不再绕过安全检查。

### 15.2 Worker 主动检测 Token 删除并停止 Bot

**问题**：TUI `/wechat disconnect` 仅调用 `storage.delete()` 删除凭证文件，不通知 Worker 停止 Bot。Bot 进程继续运行，直到下次 iLink API 调用返回 `errcode -14` 才停止，期间用户以为已断开但实际上消息仍在处理。

**修复**（`packages/opencode/.../worker.ts`）：

`tryAutoStartWechat()` 重构执行顺序——先检查凭证文件是否存在，若不存在且 Bot 正在运行则立即停止：

```typescript
async function tryAutoStartWechat() {
  if (starting) return
  if (!serverUrl) return

  const creds = await wechatStorage.load().catch(() => null)
  if (!creds?.botToken) {
    // ← 新增：token 被删且 bot 还在运行 → 立即停止
    if (wechatRunner && wechatRunner.status() !== "disconnected") {
      await wechatRunner.stop()
      wechatRunner = null
      Rpc.emit("wechat.status", { status: "disconnected" })
    }
    return
  }
  // ... 原有启动逻辑
}
```

最大延迟从"等 API 报错"降至 ≤3 秒（Worker 轮询间隔）。

### 15.3 Sidecar 即时响应凭证变更

**问题**：Desktop 模式中，扫码确认后 server 保存凭证到文件，但 sidecar 的 `check()` 每 10 秒才执行一次。用户扫码后立刻看到"已连接"，但 Bot 实际要等 0~10 秒才启动。断开连接同理。

**修复**：

server 端（`packages/opencode/.../server.ts`）：在 QR 扫码确认保存凭证后，以及 disconnect 删除凭证后，通过 `process.emit` 通知 sidecar：

```typescript
// 扫码确认后
process.emit("wechat-creds-updated")

// 断开连接后
process.emit("wechat-creds-updated")
```

sidecar 端（`packages/desktop/.../sidecar.ts`）：监听事件并立即触发 `check()`：

```typescript
process.on("wechat-creds-updated", () => {
  if (!checking) void check()
})
```

server 和 sidecar 运行在同一 Electron utilityProcess 中，`process.emit` 是进程内零延迟通信。

**时序对比**：

```
修复前：
  扫码确认 → 存凭证 → Desktop 显示 connected → 等 0~10s → sidecar 启动 Bot

修复后：
  扫码确认 → 存凭证 → process.emit → sidecar 立即 check() → Bot 即时启动
```

### 15.4 Desktop QR 获取加重试

**问题**：`handleConnect()` 中 `getQrCode()` 一次失败就显示 error，无重试。iLink API 偶发超时或返回异常时，用户看不到 QR 码且无明确错误提示。

**修复**（`packages/app/.../settings-wechat.tsx`）：

```typescript
async function handleConnect() {
  // ...
  let data = null
  for (let attempt = 0; attempt < 3; attempt++) {
    if (attempt > 0) {
      const delay = attempt * 1000
      setQrStatus(t("qrcode.retrying", { attempt: String(attempt + 1) }))
      await new Promise((r) => setTimeout(r, delay))
    }
    data = await cl.getQrCode().catch((err) => {
      console.error("WeChat QR error (attempt " + (attempt + 1) + "):", err)
      return null
    })
    if (data?.qrcode) break
  }
  if (!data?.qrcode) {
    setStateWithTime("error")
    setQrStatus(t("qrcode.fetch_failed"))
    return
  }
  // ...
}
```

重试策略：最多 3 次，间隔 1s / 2s。期间 UI 显示 "获取中，正在重试 (2/3)..."，全部失败后显示 "获取二维码失败，请稍后重试"。

**新增 i18n key**：

| Key | 中文 | English |
|-----|------|---------|
| `settings.channels.wechat.qrcode.retrying` | 获取中，正在重试 ({{attempt}}/3)... | Fetching, retrying ({{attempt}}/3)... |
| `settings.channels.wechat.qrcode.fetch_failed` | 获取二维码失败，请稍后重试 | Failed to fetch QR code, try again later |

### 15.5 CODE_REVIEW 已修复项汇总

基于 2026-07-28 审查报告的修复状态：

| # | 问题 | 状态 |
|---|------|------|
| P0-1 | Desktop sidecar 空回调无法展示 QR | ✅ 已修复 — `onQrCode`/`onStatusChange` 通过 `parentPort.postMessage` 推送 |
| P0-2 | server.ts wechatRoutes 重复实现 auth 逻辑 | ✅ 已修复 — 导入 `requestQrCode()`/`pollQrStatus()` 替代内联 fetch |
| P0-3 | 凭证双路径（auth.ts + runner.ts 各自操作文件） | ✅ 已修复 — auth.ts 不再含持久化函数，统一由 `CredentialStorage` 管理 |
| P1-1 | log.ts 顶层 Node 导入导致浏览器崩溃 | ✅ 已修复 — 改为懒 `import()` |
| P1-2 | doQrAuth 尾递归可能栈溢出 | ✅ 已修复 — 改为 `while(true)` + `continue` |
| P2-1 | qrcode 静态导入拖入浏览器 bundle | ✅ 已修复 — `generateTerminalQr` 内 `await import("qrcode")` |
| P2-2 | api.ts Buffer 依赖 | ✅ 已修复 — `btoa()` 替代 `Buffer.from()` |
| P2-3 | 静默吞错 | ✅ 已修复 — 全项目 11 处 `.catch(() => {})` 改为 `.catch(err => wechatLog.warn(...))` |
