# OpenCode 全局架构导览（v2.0.30）

> 基于对 `/home/donghe/github/opencode`（分支 `dev`，2026-10-16）的四路并行代码探索整理。
> 仓库：`anomalyco/opencode` — 开源 AI 编程 agent，Bun + turbo monorepo，TypeScript + Effect 全家桶。
> 本目录入口篇；[返回索引](README.md) | [下一篇：包版图 →](2_package-map.md)

---

## 目录

- [1. 分层架构（依赖方向铁律）](#1-分层架构依赖方向铁律)
- [2. core —— 事件溯源的 Session 引擎](#2-core--事件溯源的-session-引擎)
- [3. server / protocol / client / sdk](#3-server--protocol--client--sdk)
- [4. tui / cli](#4-tui--cli)
- [5. 其余版图（按重要性）](#5-其余版图按重要性)
- [6. 开发工作流要点](#6-开发工作流要点)
- [附录 A：请求生命周期（prompt → 落盘）](#附录-a请求生命周期prompt--落盘)
- [附录 B：Session 事件类型清单](#附录-bsession-事件类型清单)
- [附录 C：数据库表清单](#附录-c数据库表清单)
- [附录 D：TUI Provider 层级与事件到屏幕的链路](#附录-d-tui-provider-层级与事件到屏幕的链路)
- [附录 E：核心文件速查表](#附录-e核心文件速查表)

---

## 1. 分层架构（依赖方向铁律）

```
schema（领域契约，Effect Schema）
   ↓
protocol（HttpApi 端点定义）────→ client（生成的 Promise/Effect/Solid 客户端，禁止依赖 core/server）
   ↓                                    ↓
core（运行时 + 持久化）                sdk（组合 client+core+server，嵌入式 host）
   ↓
server（Effect HttpApi handlers + SSE）
   ↓
tui / cli / app / desktop（各端 UI）
```

- 依赖方向：**Schema → Core → Protocol → Server**；client 只依赖 Schema+Protocol，**永不**依赖 Core/Server；`sdk` 组合 Client+Core+Server。
- 改了 protocol/server 的 `HttpApi` 后，必须在 `packages/client` 跑 `bun run generate`（由 `@opencode/httpapi-codegen` 生成，**不可手编**生成文件）。
- 权威规范文档在 `specs/v2/`：`session.md`、`tools.md`、`provider-policy.md`、`event-stream-architecture.md`、`catalog-config-plugin-lifecycle.md`。各关注点的所有权：HTTP 操作归 Protocol、公共形状与 durable 事件归 Schema、运行时行为与持久化归 Core、贡献者红线归根 `AGENTS.md`。

---

## 2. core —— 事件溯源的 Session 引擎

位置：`packages/core/src`。

**心智模型**：一切状态变更 = 发布 durable 事件到 `Bus`（`bus.ts`），`SessionProjector`（`session/projector.ts`）在**同一事务**内把事件折叠进 SQLite（Drizzle ORM）。数据库只是派生读模型。

### 2.1 Session 子系统（`session/`，40+ 文件）

| 模块 | 职责 |
|---|---|
| `session.ts`（顶层） | `Session.Service` 门面：create/fork/prompt/synthetic/shell/compact/move/interrupt/revert/inbox 等 |
| `session/session.ts` | `Session.make()` 工厂 → 按 ID 绑定的 Handle（`forSession(sessionID)`） |
| `session/inbox.ts` | `SessionInbox.Service`：待办队列（`session_inbox` 表）。`admit/steer/queue/cancel/promote` |
| `session/execution.ts` | `SessionExecution.Service`：进程级执行协调器 + write-ahead claim（崩溃恢复标记） |
| `session/run-coordinator.ts` | `SessionRunCoordinator`：合并唤醒、同 Session resume join、不同 Session 并发 |
| `session/runner/llm.ts` | `SessionRunner` 实现：drain 循环、上下文准备、Step 执行、compaction、标题生成 |
| `session/runner/step.ts` | `SessionStep`：一次 LLM 调用尝试（流式、工具执行、重试、溢出恢复） |
| `session/projector.ts` | 事件 → DB 投影（与事件提交同事务） |
| `session/store.ts` | `SessionStore.Service`：Session/消息 CRUD、执行 claim/release |
| `session/sql.ts` | Drizzle 表定义 |
| `session/prompt.ts` | `SessionPrompt.prepare()`：物化文件附件、技能、agent |
| `session/context.ts` | 上下文组装（agent + model + tools + instructions） |
| `session/compaction.ts` | 上下文压缩 |
| `session/revert.ts` | Revert（stage/clear/commit） |
| 其他 | `system-prompt.ts`、`title.ts`、`subagent-job.ts`、`instruction-*.ts`、`move.ts`、`diff.ts`、`shell.ts`、`history.ts`、`message.ts` 等 |

**关键语义**（详见 `specs/v2/session.md`）：

- **准入与执行分离**：`Session.prompt()` 先发布 `session.inbox.enqueued`（投影插入 `session_inbox` 行），再 advisory `SessionExecution.wake()`；`resume: false` 表示只准入不调度。投递发布 `session.inbox.delivered`，其投影在同一事务内消费 inbox 行并插入可见消息。
- **投递模式**：`steer`（默认，在安全步边界按入队顺序插入）vs `queue`（空闲时投递；空闲边界 steer 优先）。
- **幂等**：同 Session 同类型重用 inbox item ID = 第一次准入获胜；跨 Session/跨类型复用则失败。
- **执行进程局部**：`SessionExecution` 只按 Session ID 键控，drain 启动时才通过 `SessionStore` + `LocationServiceMap` 发现放置；runner/model/工具/权限/文件系统都是 Location 作用域。
- **一个 Step = 一次显式 `llm.stream` 调用**，从重载的历史继续；**禁止内存内 tool loop**。一个 Step 可能有多个物理尝试（重试/溢出压缩等）。
- **指令是值增量**：内容寻址（SHA-256），blob 去重存储于 `instruction_blob`；epoch 跨 compaction。
- **恢复边界**：write-ahead claim 标记进程忙碌期；正常终止释放，关机中断/进程死亡保留；启动恢复有界重试，不保证 exactly-once。

### 2.2 其他核心模块

| 模块 | 职责 |
|---|---|
| `agent.ts` | `Agent.Service`：agent 注册表（`State` 响应式），默认 `build`；配置默认 > build > 首个非子代理 |
| `tool.ts` + `tool/runtime.ts` | 工具注册表：作用域注册（最新注册获胜，关闭揭示上一个）、权限门控（`whollyDisabled` 过滤）、CodeMode 集成、输入解码/输出编码/内容规范化（head+tail 截断） |
| `provider.ts` | `Provider.Service`：~50+ 内置 provider 包（`@opencode/ai/providers/*`）+ npm 可装；可用性 = 激活 + 集成连接 |
| `model.ts` | `Model.Service`：模型目录、能力、`default()`/`small()`（压缩/标题用轻量模型） |
| `model-resolver.ts` | 目录模型 → 运行时 `LanguageModel`（凭证注入、variant overlay、包加载） |
| `config/` | 多源加载（优先级低→高）：managed policy → wellknown → 全局补充 → 显式文件 → 直传 → 项目补充（`.opencode/`）→ content → managed settings；文件 watch + 10 分钟 wellknown 刷新 |
| `permission.ts` + `permission/saved.ts` | 权限评估（allow/deny/ask）、pending 请求与用户应答、已存规则持久化 |
| `database/` | SQLite（WAL、bun/node/workerd 三平台实现）、迁移自动应用、按文件路径信号量 |
| `bus.ts` | `Bus.Service`：publish/subscribe + durable 按 aggregate 有序日志（replay + live tail） |
| `mcp/` | MCP 客户端集成 |
| `worktree.ts` | git worktree 管理 |
| `snapshot.ts` | 文件系统快照（diff/revert 用） |
| `plugin/hooks.ts` | `PluginHooks.Service`：aisdk/session/permission/shell/tool 钩子 |
| `state.ts` | `State.create()`：带 transform 重放的响应式注册表状态 |
| `location.ts` | Location（目录+workspace）作用域服务 |
| `v1/` | V1 兼容层 |

### 2.3 工具子系统契约（详见 `specs/v2/tools.md`）

- 一个结构化工具值 `Tool.make({ description, input, output, execute })`；`output` 是给 Code Mode 的机器值，`content` 是模型可见值，`metadata` 是 UI 用紧凑 JSON。
- 每次调用都有持久身份：`context = { sessionID, agent, messageID, callID, progress }`，由 runner 提供。
- 注册按名、作用域化；`execute.after` 钩子在发布前可再变换结果。
- 中断（Effect interruption）不是工具结果；`ToolFailure` 是模型可见的预期失败。
- 注册表负责模型内容 bounding（默认 head+tail 截断，超限文本入托管存储换预览），producer 限制与注册表限制分离。

---

## 3. server / protocol / client / sdk

### 3.1 protocol（`packages/protocol/src`）

- `api.ts`：`makeDefaultApi()` 组装 ~30 个 HttpApiGroup（Server、Location 作用域的 Agent/Model/Provider/MCP/FileSystem…、Session 组、Event 组）+ Authorization/SchemaError 中间件。
- `client.ts`：`ClientApi`（客户端专用中间件标识）、`groupNames`、promise/effect 各自的 `omitEndpoints`。
- `errors.ts`：27+ 带状态码的标签错误（`SessionNotFoundError` 404、`SessionBusyError` 409…）。
- `groups/event.ts`：`GET /api/event` SSE，`OpenCodeEvent` 联合 + `isOpenCodeEvent()` 守卫。
- `simulation.ts`：JSON-RPC 2.0 UI 模拟协议（`ui.type/click/snapshot` ↔ `llm.chunk/tool.attach`…）。

### 3.2 server（`packages/server/src`）

- `routes.ts`：`createRoutes()` 构建全服务 Effect Layer（Database/Bus/Job/Project/Session/…）；`createEmbeddedRoutes()` 供 SDK 进程内嵌。
- `handlers.ts`：合并 28 个 handler 模块。
- `event-feed.ts`：订阅 Bus → SSE 帧（`data: {...}\n\n`），每订阅者 4096 队列，慢消费者溢出断开。
- `handlers/event.ts`：连接先发 `server.connected`，15 秒心跳。
- `process.ts`：`ServerProcess.start()` —— 绑定、鉴权、就绪状态机（starting→ready/failed）、`/api/info`、启动时恢复挂起 session。

### 3.3 client（`packages/client/src`）

- 双风味：`promise/`（fetch，SSE 端点返回 AsyncIterable）与 `effect/`（Effect HttpClient）。
- `solid/data.ts` + `connection.ts`：SSE 事件投影为 Solid store（session/message/model/agent/permission/form），断线重连 + 缓存失效重同步 —— **TUI 与 web 共用这一层**。
- `shared-events.ts`：多订阅者共享一条 SSE 连接。
- `service.ts`：后台服务生命周期 —— 注册文件（XDG state 目录，`{id,version,url,pid,password}`，0600）即完整发现契约；`discover/ensure/stop`（版本不匹配则替换）。
- 生成流程：`script/build.ts` → `compile(ClientApi)` → `emitPromise()` / `emitEffectImported()` / `emitEffectShape()`。

### 3.4 sdk（`packages/sdk/src`）

- `create()` → `EmbeddedHost`（`internal/host.ts`）：进程内 server + `:memory:` SQLite + `ManagedRuntime` + Web handler 直连 fetch。
- 用途：把 opencode 嵌进别的应用；支持插件注册。

### 3.5 function（`packages/function/src`）

Cloudflare Workers + Hono：session 分享（`SyncServer` Durable Object：share_create/delete/sync/poll/data）、GitHub App token 交换、飞书/Discord 桥。

---

## 4. tui / cli

### 4.1 cli（`packages/cli/src`）

- 入口 `index.ts`：Bun 脚本 + effect/cli 命令树，`Runtime.handlers()` 懒加载 handler。
- 命令：默认=TUI、`serve`、`run`（非交互管道）、`mini`、`service start/status/stop/…`、`auth`、`mcp`、`plugin`、`session`、`models`、`stats`、`acp`、`pair`、`upgrade`…
- `commands/handlers/default.ts`：`ServerConnection.resolve()` 决定连接方式 —— 显式 `--server` / `--standalone` 内嵌 / 托管后台服务（discover/ensure）。
- `server-process.ts`：服务端进程三种模式（default/service/stdio）。
- `dev/vite.ts`：Vite 通道开发模式 —— Solid JSX transform + 把 TUI `run` 注册为 Bun 插件模块，支持热重载；`bun run dev:live` 连已安装的后台服务实测。

### 4.2 tui（`packages/tui/src`）

- 技术栈：OpenTUI（`@opentui/core` 渲染器 60FPS、kitty 键盘协议）+ SolidJS。
- `app.tsx`：`run()` 创建 CliRenderer，嵌套 25+ context provider（见附录 D）。
- `context/` 是状态层：`client.tsx`（SDK 客户端+全局事件总线）、`data.tsx`（`createData()`）、`route.tsx`（home/session/plugin 三路由，`reconcile()` 导航）、`keymap.tsx`（`@opentui/keymap` 层/命令/绑定/leader）、`theme.tsx`（暗亮模式、调色板探测、SIGUSR2 刷新）、`local.tsx`（模型/agent/pin 选择）。
- Session 渲染：`routes/session/index.tsx` 转录滚动区（sticky scroll、尾部 40 行优先挂载）+ `rows.ts` 行变换 + `message-parts.tsx` 部件渲染。
- `component/session-frame.tsx`：布局（session 主区 + 侧栏/终端/插件面板 + 拖拽分隔条）。
- 主题**必须用语义 token**（AGENTS.md 硬规则：禁止裸 `theme.hue`、禁止挪用无关 token；缺角色先扩 schema）。
- TUI 故事：`feature-plugins/system/storybook/`，`OPENCODE_STORY=<id> bun run dev:live` 运行。

---

## 5. 其余版图（按重要性）

### 承重包

| 包 | 说明 |
|---|---|
| `ai` | LLM provider 客户端层（~50 provider、流式、Bedrock/Google 认证）；provider 测试用 `http-recorder` 磁带 |
| `codemode` | Effect 原生沙箱 JS/TS 解释器（Acorn），安全执行 AI 生成代码 |
| `plugin` | 插件 SDK（Effect + Promise 双 API） |
| `app` | SolidJS web 应用（app.opencode.ai，也是桌面端渲染器；严格 i18n） |
| `desktop` | Electron 44 桌面壳（主进程 IPC + `@opencode/app` 渲染器） |
| `ui` | 共享组件库（SolidJS + Kobalte + Tailwind 4 + Shiki/KaTeX） |
| `session-ui` | 会话时间线/消息/diff 渲染组件 |
| `gui-extensions` | 桌面/web GUI 扩展 SDK（文件树、变更面板、浏览器等） |
| `theme` | TUI 主题定义与解析 |
| `simulation` | 会话模拟/录制/回放框架 |
| `http-recorder` | VCR 式 HTTP/WS 录制回放（测试磁带） |
| `httpapi-codegen` | HttpApi → 类型化客户端代码生成 |
| `util` | 横切工具（Effect layer、OTel、npm 解析、文件 watch） |

### 边缘包

- **内部运营**：`console/*`（管理台，SolidStart）、`stats/*`（analytics，PlanetScale）
- **文档营销站**：`web`、`posts`、`services/www|changelog|files|update`（Astro + Cloudflare）
- **IDE**：`sdks/vscode`（薄桥接扩展）
- **CI/infra**：`containers`（CI 镜像）、`infra/` + `sst.config.ts`（SST，Cloudflare 为主）、`turbo.json`（transit 缓存测试哈希）
- **TUI 插件**：`merman`（Mermaid）、`latex`（LaTeX 渲染）；`plugins/figma`（Figma MCP，Git 安装不随包发布）

### 空壳（忽略）

`effect-drizzle-sqlite`、`effect-sqlite-node`、`enterprise`、`llm`、`opencode`、`sdk-next`、`slack`、`identity`（仅品牌资产）、`console/{core,mail,support}`。

---

## 6. 开发工作流要点（来自 AGENTS.md）

- **校验**：根目录 `bun run check`（oxlint + ast-grep + turbo typecheck）是权威；聚焦迭代时在包目录 `bun typecheck`（如 `packages/core`），**别跑 tsc**。
- **测试**：只能在包目录跑（root 有 guard）；写测试前先读 `.opencode/skills` 对应 skill（`unit-tests`/`tui-tests`/`app-e2e-tests`/`component-tests`）；尽量不用 mock。
- **风格**：禁 `any`/别名导入/star import/`else`；偏好早返回、内联单用变量、Bun API（`Bun.file()`）、函数式数组方法；Drizzle 字段 snake_case；`src/config` 模块用文件顶部 self-export 模式。
- **git**：分支 ≤3 词连字符（如 `session-recovery`）；commit/PR 用 `type(scope): summary`（feat/fix/docs/chore/refactor/test）；默认分支 `dev`，PR 目标 `dev`，diff 对 `origin/dev`；不用 Changesets。
- **Session Core 红线**（改 core 必读，见 AGENTS.md「Session Core」节 + `specs/v2/`）：durable 事件最小化、准入与执行分离、执行进程局部、一步一流式调用、投递词汇明确（steer/queue）、"turn" 一词保留给未来语义。

---

## 附录 A：请求生命周期（prompt → 落盘）

```
用户 prompt
  → Session.prompt()
      SessionPrompt.prepare() 物化附件/技能/agent
      SessionInbox.admit() 发布 session.inbox.enqueued → 投影插 session_inbox 行
      SessionExecution.wake()
  → SessionExecution（claim 写前标记，发布 session.execution.started）
  → SessionRunner.drain()
      nextPromotable() → promote() 发布 session.inbox.delivered
      （投影：删 inbox 行 + 插 session_message 用户消息，同事务）
      SessionContext.select() 解析 agent+model+tools
  → SessionRunner.runStep()（每步循环）
      溢出检查（可能触发 compaction）
      SessionModelRequest.baseTranscript() 组装 system+history+tools
  → SessionStep.attempt()
      llm.stream()（发布 step.started / step.streamed 增量）
      工具调用：Permission.assert() → 快照 → Tool.Snapshot.execute()
                （beforeExecute → execute → afterExecute → 图片规范化）
      发布 tool.called / tool.success|tool.failed
      step.ended（usage：cost/tokens、diff）
  → 投影更新 session_message / session_v2（usage 累计）
  → 循环直到：无工具调用 / 达到 agent 步数上限 / 用户中断 / 溢出压缩
```

## 附录 B：Session 事件类型清单

来自 `@opencode/schema/session-event`：

- **生命周期**：`session.created` `session.deleted` `session.moved` `session.forked` `session.renamed` `session.viewed` `session.agent_selected` `session.model_selected` `session.metadata_updated` `session.permissions`
- **Inbox**：`session.inbox.enqueued` `session.inbox.delivered` `session.inbox.cancelled` `session.inbox.delivery_changed`
- **执行**：`session.execution.started` `session.execution.succeeded` `session.execution.failed` `session.execution.interrupted`
- **Step**：`session.step.started` `session.step.streamed` `session.step.ended` `session.step.failed`
- **工具**：`session.tool.input.started` `session.tool.input.ended` `session.tool.called` `session.tool.success` `session.tool.failed`
- **文本/推理**：`session.text.started` `session.text.ended` `session.reasoning.started` `session.reasoning.ended`
- **压缩**：`session.compaction.started` `session.compaction.ended` `session.compaction.failed`
- **Shell**：`session.shell.started` `session.shell.ended`
- **技能**：`session.skill.activated`
- **Revert**：`session.revert.staged` `session.revert.cleared` `session.revert.committed`
- **其他**：`session.usage_recorded` `session.message_content_updated` `session.synthetic` `session.instructions_updated` `session.retry_scheduled`

## 附录 C：数据库表清单

| 表 | 定义处 | 用途 |
|---|---|---|
| `session_v2` | `session/sql.ts` | Session 元数据、agent/model、usage、执行 claim、revert 状态 |
| `session_message` | `session/sql.ts` | 全部消息（seq 排序，JSON payload） |
| `session_inbox` | `session/sql.ts` | 未消费的待办工作 |
| `instruction_entry` | `session/sql.ts` | 每会话指令键值 |
| `instruction_blob` | `session/sql.ts` | 指令内容去重（哈希寻址） |
| `instruction_state` | `session/sql.ts` | 指令 epoch（跨 compaction） |
| `event` / `event_sequence` | `event/sql.ts` | durable 事件存储 / aggregate 序列 |
| `worktree` | `worktree/sql.ts` | worktree 记录 |
| `permission_saved` | `permission/sql.ts` | 已存权限规则 |
| `project` | `project/sql.ts` | 项目记录 |

## 附录 D：TUI Provider 层级与事件到屏幕的链路

Provider（外→内）：Log → Exit → Epilogue → TuiApp → ErrorBoundary → TuiPaths → Storage → TuiLifecycle → TuiTerminalEnvironment → TuiStartup → Args → Config → Keymap → Toast → Route → Client → Permission → Data → Location → SessionTabs → SessionTerminals → Theme → Local → PromptStash → Dialog → Frecency → PromptHistory → PromptRef → EditorContext → Attention → UpdateNotification → Panel → Plugin → `App()`

```
SSE 流 → createClientConnection（重连/批量）
  → events.emit(type, event)（全局事件总线）
    → DataProvider（createData() 投影为 Solid store）
      → createSessionRows()（rows.ts 行变换）
        → Session 渲染（虚拟挂载，尾部 40 行优先）
          → SessionRowView → TextPart / InlineToolRow …
```

## 附录 E：核心文件速查表

| 领域 | 文件 |
|---|---|
| Session 门面 | `packages/core/src/session.ts` |
| Session 工厂 | `packages/core/src/session/session.ts` |
| 执行协调 | `packages/core/src/session/execution.ts` |
| Inbox | `packages/core/src/session/inbox.ts` |
| Runner | `packages/core/src/session/runner/llm.ts` |
| Step | `packages/core/src/session/runner/step.ts` |
| 投影 | `packages/core/src/session/projector.ts` |
| 表定义 | `packages/core/src/session/sql.ts` |
| 工具注册表 | `packages/core/src/tool.ts` + `tool/runtime.ts` |
| Agent | `packages/core/src/agent.ts` |
| Provider/Model | `packages/core/src/provider.ts` / `model.ts` / `model-resolver.ts` |
| Config | `packages/core/src/config.ts` + `config/` |
| Bus | `packages/core/src/bus.ts` |
| 权限 | `packages/core/src/permission.ts` |
| API 组装 | `packages/protocol/src/api.ts` / `client.ts` / `errors.ts` / `groups/*.ts` |
| Server 路由 | `packages/server/src/routes.ts` / `handlers.ts` / `event-feed.ts` / `process.ts` |
| 生成客户端 | `packages/client/script/build.ts`（`bun run generate`） |
| Solid 数据层 | `packages/client/src/solid/data.ts` / `connection.ts` |
| 服务发现 | `packages/client/src/service.ts` |
| SDK 嵌入 | `packages/sdk/src/internal/host.ts` |
| CLI 入口 | `packages/cli/src/index.ts` / `commands/commands.ts` |
| 服务连接 | `packages/cli/src/services/server-connection.ts` |
| TUI 入口 | `packages/tui/src/app.tsx` |
| Session 路由 | `packages/tui/src/routes/session/index.tsx` / `rows.ts` |
| 主题 | `packages/tui/src/theme/index.ts` / `context/theme.tsx` |
| 键映射 | `packages/tui/src/context/keymap.tsx` |
| 行为规范 | `specs/v2/*.md` |
| 贡献者红线 | `AGENTS.md` |
