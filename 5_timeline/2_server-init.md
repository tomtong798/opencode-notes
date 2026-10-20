# 后端服务初始化：`opencode serve --service` → `/api/info` 可答

> 一句话：**CLI 入口编译两个引导节点 → 绑端口并立刻服务 `/api/info`（starting 态）→ boot 效应编译 25 个全局服务节点（Database 先行）→ 装配 HttpApi → `status.ready` 翻门——此后 `/api/info` 返回 200，业务请求开始路由。**
> 代码基线：v2.0.30，`dev` 分支；行号实测（2026-10-16）。
> [返回 README](README.md) | 上一篇 [1_desktop-boot.md](1_desktop-boot.md) | 下一篇 [3_location-lazy.md](3_location-lazy.md)

---

## 0. 全景：两个阶段、一条门

```
阶段 A：进程引导（server-process.ts）          阶段 B：服务启动（server/process.ts）
┌────────────────────────────────┐            ┌────────────────────────────────────┐
│ LayerNode.compile(引导节点)     │            │ listen/bind —— 端口先绑！           │
│ Global 目录创建                 │  ──────►   │ dispatch 挂载（/api/info 特殊路径）│
│ PTY handoff 解码                │            │ boot 效应：                        │
│ ServiceConfig 读取(host/port/pwd)│           │   createRoutes → 25 全局节点编译    │
│ 密码解析/生成 + instanceID      │            │   Database: WAL+迁移（信号量内）    │
│ 在位服务检查（已跑则 no-op）     │            │   HttpApiBuilder 装配 handlers     │
│ 动态 import server/process     │            │   V1Migration 后台 fiber           │
└────────────────────────────────┘            │   会话恢复（write-ahead claim）     │
                                              │ status.ready ★门翻开               │
                                              └────────────────────────────────────┘
```

关键设计：**端口在服务树构建之前就绑定**——`/api/info` 在 `starting` 态即可应答（503+retry-after），发现方（桌面端 `Service.discover`、CLI `service status`）据此轮询就绪，而不是盲等连接拒绝。

---

## 1. 入口：`opencode serve`

**`packages/cli/src/commands/handlers/serve.ts:6-17`**

```
Runtime.handler(Commands.serve)
  L9   校验：--service 与 --stdio 互斥
  L10-15 ServerProcess.run({ mode, hostname, port, cors })
```

三种 mode 的语义：`default`（前台，密码打印到控制台）/ `service`（托管后台，写注册文件）/ `stdio`（JSON 传输，stdin 关闭即停）。

## 2. 进程引导（server-process.ts）

**`packages/cli/src/server-process.ts`** —— `ServerProcess.run` 的执行流：

```
ServerProcess.run(options)
│
├─ L32-45  processEffect 包进
│          LayerNode.compile(group([Global.node, AppProcess.node]))
│          ★ 此刻只引导两个节点：Global 创建全部 XDG 目录；
│            服务树完全未碰
│          └ L37-39 若设 OPENCODE_CONFIG_DIR → Global.layerWith() 替换
│
├─ L48-59  PTY handoff：读 OPENCODE_PTY_HANDOFF 解码
│          PersistentPty.Handoff（服务重启不杀终端的交接）
│
├─ L60-61  获取 Global.Service；service 模式 chdir(global.home)
│
├─ L65-68  service 模式读 ServiceConfig：hostname/port/password/CORS/remote
│           端口/host 优先级：CLI 参数 > 配置 > 默认
│           （host 127.0.0.1；service 端口 ServiceConfig.defaultPort()）
│
├─ L69-73  在位检查：目标 URL 已有受管服务？
│           └ 是 → 直接返回（no-op，幂等）★
│
├─ L77     ★ 动态 import("@opencode/server/process")
│           （HTTP 重模块延迟到此刻——启动敏感路径的懒加载纪律）
│
├─ L78-90  密码解析
│           ├ service：配置密码或随机生成
│           ├ stdio：env（同时从 env 删除，防泄漏给工具子进程）
│           └ default：随机
│
├─ L91     instanceID = randomUUID()（注册文件身份）
├─ L92     获取 WebUi.handler()（内嵌 Web UI 资产变换）
│
├─ L94-156 调 ServerProcess.start(...传 ServerOptions + Lifecycle)
│           └ L139-153 Lifecycle.onListen（service 模式）：
│               密码写入配置 + ServiceRegistration.register()
│               ★ 注册文件在此刻写（§5）
│
├─ L157-180 启动重试：EADDRINUSE 最多等 15 秒
├─ L182-198 配置了 remote tunnel → fork RemoteTunnel.run()
│
└─ L202-206 等待语义分叉
            ├ service → 等 shutdown
            ├ stdio   → 等 stdin 关闭
            └ default → Effect.never
```

## 3. 服务启动（server/process.ts）

**`packages/server/src/process.ts`** —— `start()` L47-140 的执行流：

```
start(options)
│
├─ L53-54   校验密码存在
├─ L57-58   创建 shutdown Latch + Status.make()（初态 starting；
│           状态机 service-status.ts:5-30：starting→ready/failed，→stopping）
│
├─ L59      ★ listen({hostname, port}) —— 绑端口！
│           ├ 未指定端口：从 4096 起递增试（L142-149）
│           └ bind()（L151-165）：node:http server + NodeHttpServer.make()
│             + finalizer（关连接）
│
├─ L68-78   挂 dispatch 处理器（压缩+CORS+错误日志包裹）
│           ★ 此刻起 /api/info 已可应答（starting → 503 + retry-after:1）
│
├─ L79-85   service 模式 → lifecycle.onListen（→ 注册文件写入，§5）
│
├─ L87-94   fork applicationScope；finalizer：进入 stopping 时
│           清空 application Ref、关全部连接
│
├─ L96-137  ★ boot 效应（服务树构建 → §4）
│           │
│           ├ L97-106  Layer.buildWithScope(createRoutes(...))
│           │          25 全局节点编译 + HttpApi 装配
│           │
│           ├ L107-111 会话恢复（仅 service 模式，gated by lifecycle）：
│           │          installRestartContinuity
│           │          → restart.resumeSuspendedSessions
│           │            （restart.ts:193-231：先恢复 shell 结果，
│           │             再按 write-ahead claim 恢复挂起执行，
│           │             maxAttempts 默认 10 有界重试）
│           │
│           ├ L112     application Ref ← Some(HttpRouter.asHttpEffect())
│           │          （此后业务请求可路由）
│           │
│           └ L113     ★★ status.ready —— 门翻开，/api/info 转 200
│
└─ 失败路径 L123-137：boot 失败 → status.fail + 清 scope + 日志
             + Effect.never（失败态进程不死：/api/info 返 500 供诊断，
             dispatch 仍活着）
```

### dispatch：请求如何被路由（L173-201）

```
每个请求
│
├─ GET /api/info 且未 ready（L188-191）
│    └ 鉴权通过 → infoResponse()（绕过整个应用层！）
│
├─ 其余请求（L192-197）
│    └ 鉴权（pairing 连接 / ticket PTY 连接除外）
│         │
│         ├─ ready   → 委托 app.value（boot 完成的 HttpRouter）（L198-199）
│         └─ 未 ready → unavailable()（503）

infoResponse()（L203-217）：{ version, pid, urls, paths: { tmp } }
  状态码随 status：ready=200 / failed=500 / starting·stopping=503+retry-after
```

## 4. boot 内核：服务树编译（routes.ts）

**`packages/server/src/routes.ts`**

### 4.1 全局服务节点（L52-78）

`applicationServiceNodes` 数组列出 **25 个全局节点**（进程级单例）：

```
Global · Database · Bus · EventLogger · httpClient · Job · Project(+sweep) ·
Worktree · Session · Instance(未绑定占位) · SessionTransfer · SdkPlugins ·
PluginUpdate · PermissionSaved · PtyTicket · PersistentPty · Credential ·
WellKnown · PtyEnvironment · ServerPairing · LocationServiceMap ·
LocationActivity · SessionRestart · Workspace
```

L79 `applicationServices = LayerNode.group(applicationServiceNodes)` 组成图。

### 4.2 替换与编译（L116-161）

- **L116-143 `standard` 替换清单**：给各全局节点注入运行时选项——数据库路径、PersistentPty 配置、Bus 持久化、App 元数据、ModelsDev、文件监听开关、Config 发现参数（project/claude/file/content）、InstructionDiscovery、gitbash、MCP 客户端信息
- **L144-152 `build(overrides)`**：standard + 可选 Instance 替换（私有实例）+ 调用者覆盖 → `AppNodeBuilder.build(applicationServices, replacements)`
- `AppNodeBuilder`（`core/src/effect/app-node-builder.ts:8-27`）：默认把 `Instance.node` 替换为"委托 `LocationServiceMap.Service.get(session.location)`"的节点，再 `LayerNode.compile(root, { replacements: [...] })`
- `LayerNode.compile`（`util/src/effect/layer-node.ts:238-319`）：应用替换 → 递归解析依赖（环检测）→ 共享节点去重 → `Layer.fromBuild` 运行时构建（`provideMerge` 归约）
- L153-161：simulation 模式动态 import `@opencode/simulation/backend` 应用模拟替换

### 4.3 HttpApi 装配（L163-192）

```
L164   services = Layer.succeedContext(构建出的上下文)
L165-177 requestServices：从上下文挑 6 个服务（Database/Credential/PermissionSaved/
       PluginUpdate/Project/WellKnown）+ ServerInfo 合入请求作用域
L178-189 HttpApiBuilder.layer(Api, { openapiPath: "/openapi.json" })：
       handlers（含 CORS）→ formLocation → sessionLocation → location 中间件 →
       authorization → schemaError → auth → HttpRouter.provideRequest(...)
L190   Layer.merge(api, V1Migration.layer.pipe(Layer.provide(services)))
```

**注意 L190**：V1 迁移不是全局节点，而是路由层装配时 merge 进来的 `Layer.effectDiscard`（`v1-migration.bun.ts:505-522`，`startImmediately: true` 后台 fiber）——旧库转换与 HTTP 服务并行跑，进度存 KV。

### 4.4 Database 初始化（节点构建时）

**`core/src/database/database.ts`** —— 数据库节点的构建流：

```
Database.node 构建
│
├─ L67-81  layer(options) 解析路径
│           ├ 相对路径基于 Global.data
│           └ restrictToOwner（L85-98）：
│               同步建文件 0600 → chmod db / -wal / -shm（Windows 跳过）
│
├─ L33-53  databaseLayer
│           ├ L37   makeDatabase 经 SqlClient 开连接
│           ├ L39-45 ★ WAL 调优（守卫 supportsTuningPragmas）：
│           │        journal_mode=WAL · synchronous=NORMAL
│           │        busy_timeout=5000 · cache_size=-64000
│           │        wal_checkpoint(PASSIVE)
│           ├ L47   PRAGMA foreign_keys = ON
│           └ L48-49 ★ 按数据库文件路径加信号量（L57-65 locks Map，
│                    防多进程并发 bootstrap），锁内跑迁移：
│
└─ migration.ts L22-52  DatabaseMigration.apply
                ├ 已有 session / session_v2 表 → applyOnly 增量迁移
                │   （L54-146：迁移日志表 → 补种 __drizzle_migrations
                │     → 逐个 pending 迁移在事务内执行）
                └ 全新库 → schema.up 一次建全表
                    + 迁移日志表标记全部完成
```

## 5. 注册文件与心跳（发现契约）

**`packages/cli/src/services/service-registration.ts`** —— 注册的生命周期流：

```
register(options)
│
├─ L22-31  组装 { id, version, url, pid, password }
│
├─ L39     写 temp 文件（mode ★0600）→ 原子 rename 到目标
│           （XDG state 下 service.json）
│
├─ L40-67  ★ 心跳 fiber：每 5 秒
│           重读注册文件 → 校验所有权
│           （id/version/url/pid/password 全匹配）
│           └ 失守 → 触发 shutdown
│             （被新版本服务顶替时，旧进程自杀让位）
│
└─ L68-71  清理时：若仍持有 → 删除注册文件
```

## 6. 就绪门语义汇总

| 时刻 | `/api/info` | 业务 API | 注册文件 |
|---|---|---|---|
| 绑端口后（boot 前） | 503 + retry-after | 503 | 已写（onListen） |
| `status.ready`（L113） | 200 | 经 HttpRouter 路由 | 心跳中 |
| boot 失败 | 500 | 503 | 心跳中（进程不死） |

**发现方视角**：桌面端 `sidecar-probe`/CLI `service status` = 读注册文件 → 轮询 `/api/info` 到 200 → 版本门通过即采用。

## 相关篇

- [1_desktop-boot.md](1_desktop-boot.md)——谁拉起了这个服务
- [3_location-lazy.md](3_location-lazy.md)——ready 之后第一个带目录的请求发生什么
- [../1_architecture/5_patterns.md](../1_architecture/5_patterns.md)——LayerNode/LayerMap 模式详解
- [../2_tech-stack/2.3_data/1_sqlite.md](../2_tech-stack/2.3_data/1_sqlite.md)——WAL 语义
