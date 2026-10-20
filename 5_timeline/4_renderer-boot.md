# 前端数据同步：renderer 启动到 store 就绪

> 一句话：**SSE 连接以 `server.connected` 握手 → 三查询并行全局 bootstrap → 每目录 bootstrap 并行拉 12 类资源 → `createData()` 的 store 与事件投影接管增量——"拉"定初始状态，"推"保实时一致。**
> 代码基线：v2.0.30，`dev` 分支；行号实测（2026-10-16）。以 Web/Desktop 共用的 `packages/app` 为主线（TUI 走 `@opencode/tui` 但 client 层同源）。
> [返回 README](README.md) | 上一篇 [3_location-lazy.md](3_location-lazy.md) | 下一篇 [5_prompt-chain.md](5_prompt-chain.md)

---

## 0. 全景：provider 树 → 连接 → 双阶段 bootstrap

```
entry.tsx render
  └ PlatformProvider → AppBaseProviders(Query/Theme/i18n/ErrorBoundary)
      └ AppInterface(ServersProvider → Settings → Router)
          └ Root: TabsProvider → GlobalProvider ★
              GlobalProvider 对每个健康服务连接 ensureServerCtx()
                └ createServerController
                    ├ createServerSdkContext（transport + SSE 连接 + 事件总线）
                    ├ createData()（Solid store，事件投影宿主）
                    └ ServerSyncContext（TanStack Query：bootstrap 与失效重取）

连接就绪后：
  Phase 1 全局 bootstrap：config(占位) ∥ path ∥ projects
  server.connected 事件：active sessions ∥ location ∥ vcs ∥ project
  Phase 2 目录 bootstrap：location.sync() 12 类资源 ∥ bootstrapDirectory()
  （进入会话页时再拉 message/pending）
```

## 1. 入口与 Provider 树

**`packages/app/src/entry.tsx`**

| 行号 | 发生什么 |
|---|---|
| L3 | `import "@/runtime/polyfills"` 最先加载 |
| L52 | `createWebPlatform(pkg.version)` 平台适配器 |
| L77-115 | `render()`：`<PlatformProvider><AppBaseProviders><AppInterface>` 三层 |

**`packages/app/src/app.tsx`**

| 行号 | 发生什么 |
|---|---|
| L24-36 | `QueryClient`：`refetchOnReconnect/WindowFocus/Mount` 全 **false**——**重取策略完全自管**（由事件驱动失效，不靠 Query 默认行为） |
| L48-85 | `AppBaseProviders`：Meta→Font→Theme→Language→i18n→ErrorBoundary→**QueryProvider**→Dialog→ExtensionRoot→FileComponent |
| L87-118 | `AppInterface`：**ServersProvider**（持久化的服务器连接表，registry.tsx:214-301）→Settings→Router；Root=L96：TabsProvider→**GlobalProvider**→CommandProvider |

## 2. 每服务器一个控制器

**`packages/app/src/runtime/server/runtime.tsx`** —— GlobalProvider 的展开流：

```
GlobalProvider
│
├─ L29-31  useServerHealth()（健康监控）
│
├─ L80-85  memo 遍历 server.list
│           └ 每个健康连接 → ensureServerCtx(conn)
│
└─ L54-76  ensureServerCtx
            ├ createRoot（L66，每服务器隔离生命周期）
            └ createServerController(conn, scope, projects, …)
               （构建流见下）
```

**`createServerController`（同文件 L163-266）**——本篇主角的构建流：

```
createServerController(conn, scope, projects, …)
│
├─ L172     createServerSdkContext(conn, scope)
│           （transport + SSE 连接 + 事件总线，见 §3）
│
├─ L174-190 ★ createData()：Solid 数据层
│           ├ api: () => sdk.api
│           ├ event.on / event.listen（事件接入）
│           ├ connection: sdk.connection
│           └ initialMessageLimit：compact 时间线 40，否则 20
│
├─ L192-195 createDesktopData()：包装桌面会话变更
├─ L197     createServerSyncContext(sdk, data)：TanStack Query 同步层
└─ L198-199 createPermissionAutoApprover
            + createServerNotificationState
```

## 3. 连接层：SSE 循环

**`packages/app/src/runtime/server/client.tsx:62-122`**（`createServerSdkContextBase`）：

| 行号 | 发生什么 |
|---|---|
| L77 | `createServerTransport({http, fetch})`——生成的 API 客户端 |
| L89 | `createOpenCodeEventSource()` = `createGlobalEmitter<OpenCodeEventMap>()` 本地事件总线 |
| L92-105 | **`createClientConnection(transport.api, {onEvent: (e) => events.publish(e), flushInterval: 16, pageLifecycle: true, …})`** |

**`packages/client/src/solid/connection.ts`** —— 重连状态机的运行流：

```
构造 createClientConnection(api, options)
│
├─ L41-48  master AbortController + 状态 store
│           （connecting → connected → reconnecting）
│
└─ L241    构造即 start() → runStream(active)
    │
    └─ L147-184  ★ 重连循环（while 未中止）
        │
        ├ 状态置 connecting（首轮）/ reconnecting（重试轮）
        │
        ├ connect()（L76-145）
        │   ├ L97   api.event.subscribe() 取 SSE 迭代器
        │   ├ L98   iterator.next() 等首个事件
        │   ├ L106-107 ★ 校验必须是 server.connected（握手协议）
        │   ├ L114  发布首个事件（进入 connected 态）
        │   └ L117-134 事件循环：逐个 touch() + publish()
        │
        ├ L157 连接存活 ≥ 1 个 delay → attempt 归零
        │      （长连成功可重置退避）
        │
        └ wait(reconnectDelay)（L33：1_000ms）→ 下一轮

关键参数与钩子：
  L35   defaultIdleTimeout = 45_000ms（无字节即中止流）
  L68-73 publish()：10ms 批量 → Solid batch（防抖渲染）
  L219-240 页面生命周期：pagehide 停 / pageshow 起 /
          后台回前台空闲 → resync()（L212：强断立即重连）
```

**对照服务端**：握手事件与 15s 心跳来自 `handlers/event.ts:8-36`（`server.connected` 先发、`Stream.tick("15 seconds")` 心跳、慢消费者队列 4096 溢出断开）。

## 4. 双阶段 Bootstrap

### Phase 1：全局（每服务器一次）

**`packages/app/src/runtime/server/sync.tsx:110-123`** —— bootstrap Query（`enabled: connected()`）：

**`global-sync/bootstrap.ts:81-108` `bootstrapGlobal`** 三查询 `runAll` 并行：

| 查询 | 行号 | 内容 |
|---|---|---|
| config | L91 | `loadGlobalConfigQuery`——**当前是占位返回 `{}`**（L55 有 TODO：V2 客户端暴露 config API 后恢复） |
| path | L92 | `loadPathQuery` → `api().location.get()` |
| projects | L93-104 | `loadProjectsQuery` → `api().project.list()`（归一化入 store） |

### `server.connected` 事件处理器

**`packages/client/src/solid/data.ts:633-666`**：连上即拉——active sessions（`api().session.active()`，与水合期收到的生命周期事件对账）∥ location 信息 ∥ `location.vcs.sync()` ∥ `project.sync()`。

### Phase 2：每目录

**`sync.tsx:190-229` `bootstrapInstance(directory)`** 的执行流：

```
bootstrapInstance(directory)
│
├─ L201  children.ensureChild(directory)
│         （每目录子 store）
│
└─ L202-219 ★ 两路并行：
    ├ data.location.sync({directory})
    │   └ data.ts:1934-1950 —— 12 类资源 Promise.all：
    │     location.syncInfo ∥ vcs ∥ agent ∥ command ∥ integration
    │     ∥ mcp.server ∥ mcp.resource ∥ model ∥ provider
    │     ∥ reference ∥ skill ∥ shell ∥ form(global)
    │     ★ 这正是触发 Location 惰性编译的那批请求——
    │       首目录 bootstrap ≈ 首次编译的计时窗口
    │       （见 3_location-lazy.md）
    │
    └ bootstrapDirectory()（bootstrap.ts:127-179）
        ├ 项目 ID 查表入 store → 种子配置
        ├ status 置 "partial"
        ├ waitForPaint()（50ms + rAF，让位首帧）
        └ 无错 → status "complete"
```

### 进入会话页时

`data.session.message.sync(sessionID)`（message.list，data.ts:1690）与 `data.session.pending.sync(sessionID)`（inbox.list，L1465）。

## 5. store 结构与事件投影

**`packages/client/src/solid/data.ts`** —— `createData()`（L231）：

### Store 形状（L113-134）

```ts
{
  session: { info, family(根→成员索引), active, message[], messageCursor,
             messageLoading, pending[], permission[], form[] },
  project: { info, permission[] },
  location: Record<目录, { info, vcs, agent, command, config, integration,
                           mcpServer, mcpResource, model, provider,
                           reference, websearch, shell, skill }>
}
```

### handleEvent——中心投影器（L633-1341）

代表性映射（与 `SessionProjector` 服务端投影**同构**，两端必须一致）：

| 事件 | 行号 | 投影动作 |
|---|---|---|
| `server.connected` | L635 | 拉初始态（见上） |
| `session.created` | L671 | 移出 outbox、sync info |
| `session.inbox.enqueued` | L816 | 移出 outbox、`admitLocal()` 立即渲染 |
| `session.inbox.delivered` | L792 | 移出 pending、消息入转录尾 |
| `session.step.started` | L881 | 建/改 assistant 行 |
| `session.text.delta` | L952 | `message.editText(..., t => t.text += delta)` |
| `session.tool.called/success` | L983/996 | 工具行 running→completed |
| `session.execution.*` | L1052-1068 | active 状态 + idle 标记 |

### 同步协调器 createSync（L174-227）

`run(key, load)` 每失效周期只跑一次（并发去重）；`invalidate(key)` 标脏；`complete(key)` 免取标记。事件驱动失效示例（L1252-1340）：`provider.updated`→provider.invalidate+sync、`integration.updated`→三表联动失效。

### 断线语义（L2002-2005）

```ts
createEffect(() => { if (connected) return; sync.invalidate() })  // 断线 → 全部 sync 键失效
```

配合 `sync.tsx:172-188` `createConnectionSync`：断线时失效该服务器全部 Query；重连后 refetch bootstrap + 活跃目录重排队——**断线期间事件缺口由"失效重取"填补**，不靠事件补发。

## 6. 设计取舍

- **拉推分工**：bootstrap"拉"出一致快照，事件"推"保增量；断线用"失效重取"对账——避免实现"事件回放补发"的复杂度（服务端 durable log 仅 `sessions.log` 消费者使用）
- **Query 全关自动重取**：失效完全由事件驱动（`provider.updated` 等），否则事件与重取会互相打架产生抖动
- **乐观 UI 的基础设施在此**：outbox 集合（未确认 admission）+ `admitLocal()`——服务端回声（`session.inbox.enqueued`）到达前界面已渲染，失败才回滚（详见 [5_prompt-chain.md](5_prompt-chain.md) §1）
- **每服务器 createRoot 隔离**：多服务器（本地+远程）各自一套 store/连接/生命周期，互不泄漏

## 相关篇

- [5_prompt-chain.md](5_prompt-chain.md)——store 就绪后，回车发生什么
- [3_location-lazy.md](3_location-lazy.md)——bootstrap 请求的服务端代价
- [../2_tech-stack/2.1_client/2_solidjs.md](../2_tech-stack/2.1_client/2_solidjs.md)——fine-grained reactivity 如何让 store 投影自动驱动渲染
- [../2_tech-stack/2.2_server/7_sse.md](../2_tech-stack/2.2_server/7_sse.md)——事件通道设计
