# MCP 集成详解（客户端生命周期 → 能力发现 → Tool/资源物化 → 调用转发的外部能力源接入层）

> 一句话：**MCP 是 opencode 作为客户端接入外部能力源（工具/资源/提示词/服务器指令）的完整管道——配置声明 Server（local stdio / remote streamable-HTTP）→ Location 级 `Mcp.Service` 按服务器粒度管理连接生命周期（连接、list 物化、会话过期恢复、OAuth 集成）→ `McpTool` 把发现的工具物化为内部 Tool.Info 单 transform 注册 → `callTool` 经 timeout 分层与进度保活转发到远端。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：`packages/core/src/mcp/`（index/client/stdio/oauth/instructions）、`packages/core/src/tool/mcp.ts`、`packages/core/src/tool/plugin/mcp-resource.ts`、`packages/schema/src/mcp.ts` + `config/mcp.ts`
> [返回 README](README.md) | 前置 [9_tool.md §5](9_tool.md)（MCP→Tool 桥接概览） | 关联 [../2_tech-stack](../2_tech-stack/)（配置面）

---

## 0.5 通俗完整描述（先读这段再看细节）

**MCP（Model Context Protocol）= 外部能力的插座标准**：你配置几个 MCP 服务器（比如文件系统、数据库、浏览器工具），opencode 作为客户端连上去，把对方提供的工具拉进自己的工具清单——模型用这些工具时和内置工具无感同权。opencode 不用为每种外部能力写适配代码，"加能力=配置一个服务器"。

两种连接形态：本地（stdio——起个子进程，stdin/stdout 通信）和远程（HTTP 流式）。连接是**随目录懒建立的**（打开项目才连它配置的服务器），每服务器一把锁防并发重连，OAuth 凭证换了自动重连。

工具进来时**schema 零转换直传**（对方给什么 JSON Schema 就用什么——信任边界明确但意味着清单质量取决于服务器方）；名字冲突用下划线替换+最新注册胜；执行超时 12 小时上限。工具调用时可携带会话 ID 的带外通道（`_meta`），对方能用它区分"哪个会话在调用"（比如 per-session 的资源隔离）。

资源（resources）和提示词（prompts）也接入：资源经 mcp-resource 工具按需读取；elicitation（服务器反过来问用户问题）限定在所属目录范围内。

一句话：**"工具的 USB 标准"——配置即接入、懒连接、schema 直传、与会话联动的带外标识——外部能力生态的接入口**。
---

## 0. 模块地图

`packages/core/src/mcp/`（5 文件，2,131 行）+ 桥接层与插件。Schema 形状在 `packages/schema/src/mcp.ts`（约 130 行）与 `config/mcp.ts`（32 行，纯 re-export）。

```
packages/core/src/
├─ mcp/
│   ├─ index.ts ★ Mcp.Service：服务器注册表 + 生命周期 + 能力缓存 + Interface（846 行）
│   │   ├─ ServerEntry 状态机（config/status/startup Latch/scope/client/tools/prompts）
│   │   ├─ startServer/stopServer/replaceServer/removeServer/reconcile
│   │   ├─ recovering：SessionExpiredError → 断线重连后重放一次请求
│   │   ├─ elicitation：服务器反向请求用户输入 → Form（"global" 哨兵 session）
│   │   └─ register：remote 服务器自动注册为 OAuth Integration
│   ├─ client.ts ★ 协议层：SDK Client 封装、传输选择、重试、能力门控（446 行）
│   │   ├─ connect()：local→McpStdio / remote→StreamableHTTPClientTransport
│   │   ├─ Connection 接口：tools/prompts/resources/callTool + 5 个 onChange 回调
│   │   └─ toCallToolResult：SDK content → {text|media} 归一化
│   ├─ stdio.ts   自研 stdio 传输：经 Environment.spawner 拉起子进程（196 行）
│   ├─ oauth.ts   OAuth：发现/授权/refresh 去重/credential 映射（532 行）
│   └─ instructions.ts McpInstructions：服务器 instructions 按权限过滤注入提示（111 行）
├─ tool/
│   ├─ mcp.ts ★ McpTool：MCP 工具 → Tool.Service 单 transform + debounce reload（152 行）
│   └─ plugin/
│       └─ mcp-resource.ts  list/read_mcp_resource 两个内置工具（101 行）
└─ plugin/
    └─ mcp-codemode-defaults.ts  已知服务器 CodeMode 默认值（41 行）
```

Schema 侧：`mcp-event.ts` 定义 3 个 ephemeral 事件（ToolsChanged/ResourcesChanged/StatusChanged）；`config.ts:82` 把 `ConfigMCP.Info`（`timeout` + `servers` record，config/mcp.ts:23-26）挂为可选配置键。

## 1. 心智模型

MCP server = 外部能力源（工具 + 资源 + 提示词 + instructions）；opencode 始终是**客户端**。它把三类能力分别物化：tools → Tool 注册表（LLM 可调用）；resources → 两个内置工具（模型主动 list/read）；prompts → 斜杠命令（plugin/command.ts:57-62）。instructions 不物化为工具，而是按权限过滤后注入系统提示。

```
config mcp.servers 声明（2_config）
        │
        ▼ State.reconcile（index.ts:537-575，diff 旧/新配置）
┌────────────────────────── Mcp.Service（Location 级）──────────────────────────┐
│  per-server KeyedMutex（index.ts:169）                                        │
│  ┌─ startServer（:427）──────────────────────────────────────────────────┐   │
│  │ status=pending → connectProvider(OAuth) → McpClient.connect          │   │
│  │   local: McpStdio（经 Environment 拉起子进程，stdio.ts:38-46 注释）    │   │
│  │   remote: StreamableHTTP + authProvider + ?codemode=false            │   │
│  │ → initialize 握手（协议协商/能力声明）→ connection.tools()            │   │
│  │ → status=connected，watch() 挂 5 个回调                                │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│  能力缓存：entry.tools / entry.prompts（list-changed 通知刷新）               │
└──────┬──────────────────────┬──────────────────────┬────────────────────────┘
       │ McpEvent.ToolsChanged│                      │
       ▼                      ▼                      ▼
  McpTool（tool/mcp.ts）  McpResourceTools       plugin/command.ts
  editor.add → Tool.Info （list/read 工具）      prompts→斜杠命令
       │
       ▼ Snapshot.execute（9_tool §5）
  permission.assert → mcp.callTool → SDK client.callTool → 服务器
```

关键分层：`mcp/index.ts` 管**生命周期与状态**，`mcp/client.ts` 管**协议与传输**，二者只通过 `Connection` 接口（client.ts:114-138）耦合——注释明言该接口的目的是"keeps the SDK Client out of the rest of core"。

## 2. 配置面

声明位于 `config.mcp.servers`（record：名字 → ServerConfig），schema 是 tagged union（schema/mcp.ts:89）：

```jsonc
// opencode.json（示意）
{
  "mcp": {
    "timeout": { "startup": 30000 },        // 全局默认，可被单服务器覆盖
    "servers": {
      "postgres": {                          // local：stdio 子进程
        "type": "local",
        "command": ["npx", "-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/db"],
        "environment": { "PGPASSWORD": "..." },
        "codemode": false                    // 可选：不进 Code Mode
      },
      "github": {                            // remote：Streamable HTTP
        "type": "remote",
        "url": "https://api.githubcopilot.com/mcp/",
        "headers": { "Authorization": "Bearer ..." },
        "oauth": false                       // 已带 token，跳过 OAuth 集成
      }
    }
  }
}
```

- **local**（stdio）：`{ type: "local", command: string[], cwd?, environment?, disabled?, codemode?, timeout?, protocol? }`（schema/mcp.ts:38-48）。command 是数组（argv 形态，非 shell 字符串）；cwd 相对路径从 workspace 目录解析（client.ts:253 `path.resolve(directory, config.cwd)`）。command[0] 为 `opencode` 时自动注入 `BUN_BE_BUN=1`（client.ts:255，让嵌套 opencode 服务器走 Bun 执行）。
- **remote**（Streamable HTTP）：`{ type: "remote", url, headers?, oauth?, disabled?, codemode?, timeout?, protocol? }`（schema/mcp.ts:71-80）。`oauth: false` 关闭 OAuth 集成（匿名/自备 header 连接）；`oauth` 对象可给 `client_id`/`client_secret`/`scope`/`callback_port`/`redirect_uri`/`auth_server_metadata_url`（服务器不发布 protected resource 元数据时手工指认授权服务器，schema/mcp.ts:65-69）。
- **protocol**（schema/mcp.ts:33-37）：`"legacy"`（默认，initialize 握手、修订至 2025-11-25）| `"auto"`（探测 2026-07-28 修订，失败回退）| `"2026-07-28"`（强制，不支持即失败）。client.ts:421-425 `negotiation()` 把它翻成 SDK 的 `VersionNegotiationOptions`；不支持时错误消息直接列出服务器支持的版本并指引用户改配（client.ts:401-405）。

实测**仅这两种传输**：client.ts:246-293 的分支只有 `local` 与 URL 两种，没有独立 SSE 传输类型——旧式 SSE 回退由 SDK 的 StreamableHTTPClientTransport 在协议协商中处理，opencode 配置面不感知。

三段超时（schema/mcp.ts:14-26 + client.ts:34-36 默认值）：

| 键 | 语义 | 默认 |
|---|---|---|
| `timeout.startup` | 建连+initialize | 30s |
| `timeout.catalog` | tools/list 等发现请求 | 30s |
| `timeout.execution` | callTool/getPrompt/readResource | **12 小时** |

凭证/env：local 的 `environment` 记录**只含声明的变量**，宿主 env 由 spawner 侧 `extendEnv` 合并（stdio.ts:26-33 注释：workspace 驱动用沙箱自身 env，宿主变量不跨越 seam）；remote 凭证不入 config——每个 remote 服务器自动注册为 OAuth Integration，credential 存全局库（index.ts:173-175 注释）。

运行期修改：`Mcp.Service` 本身是 `State.Transformable<Editor>`（index.ts:113），`/mcps` UI 可 add/connect/disconnect/remove；`reconcile` 对配置做 deep-equal diff 后按服务器增删替换（index.ts:563-573）。

## 3. 连接生命周期

### 3.1 状态机

每个服务器的 `ServerEntry.status`（index.ts:78-89）走五态：

```
                reconcile/add 首见
                      │
                      ▼
                 ┌─────────┐  disabled 配置 / disconnect()
      startServer│ pending │──────────────────────────▶┌──────────┐
                 └────┬────┘                           │ disabled │
        connect 成功 │ │ connect 失败                   └────┬─────┘
        +首拉 tools  │ │  (NeedsAuth→needs_auth, 否则 failed)  │ connect() / 配置重新启用
                      ▼ ▼                                        │
               ┌───────────┐   onClose（进程崩溃/断连）    ┌──────────┐
        ┌─────▶│ connected │─────────────────────────▶    │  failed  │
        │      └─────┬─────┘  stopServer 后 startServer   └────┬─────┘
        │            │                                          │
        └────────────┘  ◀───────────────────────────────────────┘
          SessionExpiredError/凭证切换 → stop+start（recover/reconnect）
                 needs_auth：等用户 OAuth 登录 → Credential.Switched → reconnect
```

所有转移都广播 `McpEvent.StatusChanged`，TUI `/mcps` 页据此刷新（schema/mcp-event.ts:22-26 注释："observers can refresh status without polling"）。

### 3.2 启动时机

**随 Location**。`Mcp.node = configured()`（index.ts:774-782）是 Location 级服务节点；首次 `reconcile` 把配置里的服务器全部建 entry，然后**逐个 fork 异步连接**——注释（index.ts:550）："Initial connections stay asynchronous so one slow server does not block Location startup"。不是懒连接：Location 起来就连，除非 `disabled`。共享同一 URL 的 remote 服务器经 `endpointLoads` KeyedMutex 串行启动（index.ts:95-96 注释："shared remote endpoints should not receive concurrent startup bursts"、:453）。

### 3.3 启动幂等门

每个 entry 带 `startup: Latch`，`startServer` 以 `Effect.ensuring(entry.startup.open)` 收尾（index.ts:479）——即使注册失败或中途被替换也放开，`callTool`/`prompt`/`resources` 等 reader 先 `entry.startup.await`（index.ts:671、708、730），不会挂死在慢服务器上（index.ts:520 注释："Settle startup even when registration fails...so readers cannot hang"）。

### 3.4 连接失败

`startServer` 把整个 connect+首拉 tools 包成 `Effect.exit`（index.ts:452）。失败时关闭 scope、按错误类型落 status：`NeedsAuthError → needs_auth`，其余 → `failed`，并广播 StatusChanged（index.ts:469-477）。**状态机层不做自动重试**——重试只在协议层内部（§3.6）；恢复路径是：用户从 `/mcps` 重连（unavailable 提示语 index.ts:65-76），或 OAuth 凭证切换事件自动 reconnect（index.ts:579-598：`Credential.Event.Switched` → 匹配 owned integration → stop+start）。

### 3.5 传输层重试

`retrying`（client.ts:171-182）对**瞬态错误**（408/429/5xx≠501、网络 TypeError，client.ts:52-58 注释：过载或重启中的服务器通常 1 秒内恢复）做最多 2 次指数重试（250ms、1s），受 startup deadline 约束；remote 连接还带 `?codemode=false` 首次失败（400/404）时用原 URL 重试一次的兜底（client.ts:285-291）。

### 3.6 关闭

三层。① scope 关闭触发 finalizer（client.ts:297-317）：legacy 会话先 `terminateSession()`（1s 超时，注释：close 会 abort 掉 DELETE 共享的 signal，所以必须先 terminate）再 `client.close()`；② stdio 子进程随 scope 被 kill 整个进程组（stdio.ts:38-46 注释，子孙进程一并回收）；③ `stopServer` 清空 entry 缓存并广播 ToolsChanged/ResourcesChanged/PromptsChanged（index.ts:483-494），让下游注册表即时缩表。

### 3.7 elicitation（服务器反向请求用户输入）

服务器可通过 `elicitation/create` 向用户要输入。form 挂在哨兵 session `"global"` 上（index.ts:91-93 注释：elicitation 是 Location 域而非 Session 域，服务器无法归属到持久 session 行）；JSON schema → Form.Field 的转换在 `toElicitationField`（index.ts:786-844），含机器生成 title 的正则识别（index.ts:788，如 "string with format email" 退回用 description/key 当标题）。URL 模式的 `notifications/elicitation/complete` 通知仅为 legacy 兼容（client.ts:103-111 注释：2026-07-28 起客户端靠重试得知结果，但 legacy 服务器仍发，能让 form 不用重跑工具就落定）。

## 4. 能力发现与物化

### 4.1 能力协商先行

SDK 对未声明能力的 list 请求会 console.debug + 空结果，污染 stdout；所以 client.ts:333-334 用 `advertised(capability)` 先查 `getServerCapabilities()`，未声明就直接返回 `Effect.succeed([])` 不发请求（client.ts:330-332 注释："the SDK answers a list for an unadvertised capability with a console.debug line...Negotiate first and leave the request unmade"）。

### 4.2 tools/list → 内部 Tool.Info

桥接细节总览见 9_tool §5，此处补协议侧。`startServer` 在 connect 成功后**立刻**拉一次 tools（index.ts:437-438 注释：失败要标 failed，不能"连着但工具空"且无恢复路径）；`toTool` 打上 `server` 与可选 `codemode` 标记（index.ts:304-308）。list 请求本身经 `list()` 包装（瞬态可重试，client.ts:324-329）并带 `timeout.catalog`。

**name 冲突**：不做改名——`McpTool.namespace()`（tool/mcp.ts:16-17）把服务器名/工具名里非 `[a-zA-Z0-9_-]` 字符替换为 `_` 作前缀（如服务器 `my.server` → 命名空间 `my_server`）。两个服务器理论上可碰撞出同一 effective name，此时按 Tool 注册表的"最新有效注册胜"规则解决（tool/AGENTS.md："The latest valid active registration for the same effective name wins"）。

**schema 转换**：无转换——SDK 的 `inputSchema`/`outputSchema` 直接 as JsonSchema 透传（tool/mcp.ts:47-48），缺省 fallback `{type:"object",properties:{}}` / `{}`。

### 4.3 动态漂移（server 加工具后客户端何时看到）

```
server: notifications/tools/list_changed
  → SDK (autoRefresh:false, client.ts:157-161) → changed.tools()
  → whenLive 守卫 (index.ts:328-336: entry.client === connection 才提交)
  → refreshTools (index.ts:310-315) → entry.tools 更新
  → bus: McpEvent.ToolsChanged
  → McpTool: sliding PubSub(1) → 100ms debounce (tool/mcp.ts:133-143)
  → reconcile: mcp.tools() + tools.reload()
  → 下一次 model request 捕获的 definitions 快照即含新工具
```

MCP 持有的是**一个稳定 transform**，reload 重放读最新 `mcp.tools()`，保住后注册 plugin 覆盖 MCP 工具的优先级（tool/AGENTS.md 明文："MCP owns one stable tool transform...MCP refresh therefore preserves the precedence of later plugin overrides"）。debounce 的理由（tool/mcp.ts:131-132 注释）："Servers announce tools in bursts and each read loads the whole catalog, so settle and refresh once"——bus 订阅保持 eager，只 debounce 已开的订阅。

### 4.4 resources 与 mcp-resource.ts

按需拉取而非常驻缓存。`resourceCatalog()` 对每个已连接服务器并发 `loadCatalog`（resources + templates，后者失败容错为空——index.ts:372 注释"有些服务器声明了 resources 但不实现 template 列举"），`mergeCatalogs` 排序合并（index.ts:756-772）。模型侧入口是 `mcp-resource.ts` 的两个工具（均 namespace "opencode"、codemode true）：

- `list_mcp_resources`（mcp-resource.ts:18-48）：列全部或单服务器；描述词明确引导模型"用户提到非本地文件的东西（自定义 scheme 的 URI）时用它"，templates 的 uriTemplate 占位符要先填再读。
- `read_mcp_resource`（mcp-resource.ts:49-97）：按 server+uri 读；图片/PDF blob 直接转 `data:` URI 作为 file content 呈给多模态模型（mcp-resource.ts:81-85），其余为文本输出。

### 4.5 prompts：消费

`refreshPrompts` 连接后异步拉取并广播 `PromptsChanged`（index.ts:466，prompts 拉取失败静默为空数组 :319）；`plugin/command.ts` 订阅该事件，把每个 prompt 物化为 `mcp_<server>_<name>` 形态的斜杠命令（command.ts:57-62 调 `mcp.prompt()` 渲染参数化 prompt——MCP prompts 即用户可触发的服务器侧模板）。

### 4.6 instructions

`Connection.instructions` 来自 initialize 握手（client.ts:338 `getInstructions()`）；`McpInstructions.load`（instructions.ts:69-100）按 Session 权限过滤——某服务器的所有工具都被 deny、或 codemode 开启但 `execute` 动作被拒时，整个服务器的 instructions 不注入（instructions.ts:86-89 注释："Instructions are useful only when this Session can reach at least one server tool"）；渲染为 `<mcp_instructions>` XML 块，codemode 服务器附带"用 execute + tools[namespace] 调用"的引导（instructions.ts:19-25），并做增量 diff 更新（纯新增/移除渲染小 delta，变更则全量重述，instructions.ts:34-52）。

## 5. 调用转发

```
McpTool.execute (tool/mcp.ts:49)
  │ ① permission.assert({action: `${ns}_${tool}`, resources:["*"], save:["*"]})
  │ ② mcp.callTool (index.ts:669)
  │     requireServer → entry.startup.await → recovering 包装
  │ ③ 协议层 client.callTool (client.ts:369-380)
  │     args ?? {} ──无编码转换──▶ SDK
  │     sessionID → _meta["ai.opencode/sessionID"]
  │     { signal, timeout: execution, onprogress: ()=>{} }
  │ ④ toCallToolResult (client.ts:427) 归一化 content
  ▼
桥接层 (tool/mcp.ts:78-110)：isError→ToolFailure；output = structured ?? JSON.parse(text) ?? text
```

要点：

- **参数编码**：直传 `Record<string, unknown>`，不做任何转换；缺失 schema 的空对象兜底。
- **超时**：`timeout.execution`（默认 12 小时）是唯一硬墙。`onprogress: () => {}` 空回调——注释（client.ts:377）："Requesting progress keeps long calls alive under the SDK's timeout"，即仅声明进度兴趣以防 SDK 中层超时杀掉长调用。opencode 自身**不消费 MCP 进度通知、不转发 tool.progress 事件**（实测无此通路）。
- **会话过期重放**：`recovering`（index.ts:350-365）在请求撞上 `MCP.SessionExpiredError` 时先 `recover`（stop+start 重连）再用新连接**重放一次**；其他失败透传。错误按 `_tag` 字符串匹配而非类引用（client 模块懒加载，index.ts:358-359 注释）。
- **不可用语义**：无 client 时按 status 生成 `ToolCallError`，提示语含恢复路径（"Reconnect it from /mcps" / "Sign in from /mcps"，index.ts:65-76），模型可见并能把指引转达用户。

**结果解码** `toCallToolResult`（client.ts:427-446）：SDK content parts 归一为两类——text→`{type:"text"}`；image/audio→`{type:"media", data, mimeType}`；`resource_link`/嵌入 resource 退化为 uri 文本或 blob 媒体。`isError:true` 在桥接层转 `ToolFailure`，消息取 text parts 拼接（tool/mcp.ts:78-85）。

**输出形状**（tool/mcp.ts:96-106）：`structured`（structuredContent）优先；否则服务器未声明 outputSchema 且 text 形似 JSON（`{`/`[` 开头）时尝试 `JSON.parse`（注释：agents 假设 JSON 文本已是对象）；media 转内联 `data:` URI file content。

**sessionID 带外通道**（client.ts:375）：`_meta["ai.opencode/sessionID"]` 让有状态服务器（如 elicitation 关联、会话记忆）能定位调用来源会话。

## 6. CodeMode 默认值

`plugin/mcp-codemode-defaults.ts`（41 行，注意在 `src/plugin/` 而非 `tool/plugin/`）：一个**配置后处理插件**，`ctx.mcp.transform` 遍历所有服务器配置，只为**未显式设置 `codemode`** 的打默认补丁（:23 `if (server.codemode !== undefined) continue`）：

- `executor mcp` 命令（local）与 `https://executor.sh/*/mcp`（remote）：这些服务器**本身就是 Code Mode**，置 `codemode: false`，防止"OpenCode 的 Code Mode 套服务器的 Code Mode"嵌套（:6-9 注释）。
- PostHog（`mcp*.posthog.com`）：服务器默认把全部工具包进一个服务端 CodeMode 工具，注入 `x-posthog-mcp-mode: tools` header 要裸工具；用 header 而非 `?mode=` 是为了不动 URL——**URL 是凭证键**（:12-15 注释）。

它与连接层的配合：client.ts:266-270 连 remote 时若 `codemode !== false` 自动在 URL 加 `?codemode=false`，让自带 Code Mode 的服务器（Cloudflare 等）吐裸工具（client.ts:267 注释："Servers that bundle their own Code Mode...expose raw tools when asked with ?codemode=false, which is what our Code Mode wants. The configured URL stays the OAuth identity"）；`rejectsUnknownQuery`（client.ts:411-418）对 googleapis/zoho/loopback 白名单跳过（它们拒未知参数且非 CodeMode 服务器）。CodeMode 本体在 `packages/codemode` + `core/src/codemode/`（MCP 工具默认 `codemode: true` 进 execute 命名空间，tool/mcp.ts:45）。

codemode 标记的三处消费链：

```
config codemode?:boolean ──▶ mcp-codemode-defaults 插件补默认值（§6）
    │
    ├─▶ toTool 附到每个 Tool (index.ts:307)
    │     ├─▶ McpTool editor.add options.codemode (tool/mcp.ts:45)
    │     │     └─▶ CodeMode execute 工具的 tools[namespace] 命名空间编排
    │     └─▶ McpInstructions：codemode=false 的服务器提示走原生工具名 (instructions.ts:21-24)
    │
    └─▶ client.ts:269 连接 URL 注 ?codemode=false（关服务器自己的 CodeMode）
```

## 7. 权限接缝

MCP 工具的 action 命名：`McpTool.name(server, tool)` = `namespace(server) + "_" + 工具名`（tool/mcp.ts:16-17）。每工具统一 `resources: ["*"], save: ["*"]` + `metadata.input`（tool/mcp.ts:51-63）——即**按工具名单独授权、资源粒度全放行**；来源固定 `source: {type:"tool", messageID, id}`（tool/AGENTS.md 规定）。

资源工具刻意更细：`list_mcp_resources` 的 resources = 被列的服务器名数组（列全部时逐服务器检查，mcp-resource.ts:33-35 注释），action `opencode_list_mcp_resources`；`read_mcp_resource` 的 resources = `[server:uri]`、save = `[server:*]`（mcp-resource.ts:65-73）——读具体资源需授权，保存产物放宽到服务器通配。instructions 注入同样过权限闸（§4）。

## 8. 不变量与设计取舍

从代码注释直接提取的设计约束：

- **SDK 隔离**：SDK `Client` 不出 `mcp/client.ts`；其余 core 只见 `Connection` 接口（client.ts:113 注释 "Handle over a connected MCP server that keeps the SDK Client out of the rest of core"）。
- **每服务器一把锁**：生命周期操作串行化（`locks = KeyedMutex.makeUnsafe<ServerName>()`，index.ts:166-169 注释）；"从连接回调拿锁的代码必须 fork"——锁内关 scope 会触发 onClose，同步拿锁会死锁。
- **whenLive 守卫**：迟到的 SDK 回调不得提交已替换连接的状态（index.ts:326-327 注释："late SDK callbacks cannot commit obsolete state"）。同理 `recover`/`reconnect` 都先重验 `entry.client === connection` / integrationID 仍匹配（index.ts:342、585-586 注释）。
- **锁序防抖**：tools 变更"成簇广播、整目录重读"，所以 sliding(1) + 100ms debounce 只结算一次（tool/mcp.ts:131-132 注释）；bus 订阅保持 eager，只 debounce 已开的订阅。
- **凭证键 = name + URL**：同一 URL 两个名字是不同逻辑服务器，可持不同账号，不得共享 credential 行（index.ts:179-181 注释）；且凭证在 remove 后**故意保留**以便 re-add（index.ts:529 注释："Credentials are keyed by name + URL and intentionally survive removal for a later re-add"）。
- **注册即独立 scope**：每个 OAuth integration 注册 fork 子 scope，直接挂 root 会为每个被替换/移除的服务器累积死 finalizer（index.ts:191-193 注释）。
- **stdio 帧上限 16 MiB**（stdio.ts:20 `MAX_FRAME_BYTES`）、stderr 只留尾部 1,000 字节入错误报告（stdio.ts:22 `STDERR_TAIL`）；stderr 持续 drain 进 debug 日志——chatty 服务器不能靠塞满管道卡死连接（stdio.ts:157 注释）。
- **环境不越界**：stdio 服务器跑在 Location 的执行面上（workspace 沙箱内），不在宿主机（client.ts:140-144 注释 "A stdio server is spawned through the location's Environment...runs wherever the location's shell commands run"）；宿主 env 在 spawner 侧合并，沙箱驱动用沙箱自身 env（stdio.ts:26-33 注释）。
- **失败语义**：连不上 tools/list 就整连接标 failed，不留"空工具的假连接"（index.ts:437-438 注释）；list 是只读所以可安全重试（client.ts:324），call 不盲目重试、只在会话过期时重放一次（index.ts:348-350 注释）。
- **读取反映当下**：`tools()` 等读取返回"当前已连上"的内容，还在启动的服务器等它发布变更事件后再出现（index.ts:663 注释 "Reads report what is connected now"）。
- **类型边界**：MCP 工具 schema 来自外部 JSON Schema，注册边界即类型安全终点（tool/AGENTS.md："Type safety ends at registration"）——运行时由注册表验证模型输入。

## 9. 失败与恢复

| 故障 | 检测 | 恢复 |
|---|---|---|
| 子进程崩溃（stdio） | stdout 流结束 → exit code + stderr 尾部拼进错误（stdio.ts:172-190） | `onclose` → watch 的 onClose 回调 → status=failed + stopServer + StatusChanged（index.ts:404-411）；用户手动重连 |
| HTTP 会话过期（legacy） | 404 或特征 400"Server not initialized"，仅在带 Mcp-Session-Id 时判定（client.ts:224-244） | 前台请求经 `recovering` 重连+重放一次；后台刷新经 `onSessionExpired` fork `recover`（index.ts:414-415 注释：后台不重试，在此代它重连）。`session.reported` 保证过期只报一次 |
| 瞬态网络/5xx | `isTransient`（client.ts:55-58） | 协议层 retrying：250ms/1s 两次，仅 connect 与 list；deadline 后放弃 |
| OAuth 过期 | UnauthorizedError → `NeedsAuthError`（client.ts:400） | status=needs_auth；用户在 `/mcps` 走 Integration 登录 → `Credential.Event.Switched` → 自动 reconnect（index.ts:593-598）。refresh token 轮换时的并发刷新共享同一请求防 invalid_grant（oauth.ts:26-27） |
| 能力清单漂移 | `notifications/tools/list_changed`（SDK 侧 autoRefresh:false + 手接，client.ts:154-156） | refreshTools → ToolsChanged → McpTool debounce reload → 下一请求快照可见 |
| 协议版本不支持 | UnsupportedProtocolVersionError（client.ts:401-405） | 不恢复；错误消息列出服务器支持的版本，指引用户配 `protocol` |

## 演进注记

- 旧笔记 4.20（2026-08-03）写的 V1 位于 `opencode/src/mcp`——**该目录在当前 dev 树已不存在**（实测 `ls` 报不存在）；V2 全部收进 `packages/core/src/mcp/` 并 Effect 化。相对 V1 的结构性变化：连接管理从扁平函数变为 `State.Transformable` 状态机（配置 diff 驱动 reconcile）；stdio 传输弃用 SDK 的 StdioClientTransport 自研（为走 Environment/沙箱执行面，stdio.ts:34-37 注释）；OAuth 凭证迁入 Integration/Credential 全局体系；新增 elicitation、SessionExpiredError 恢复、codemode 协作、instructions 权限过滤。
- 协议时间线内嵌于代码：`protocol` 配置的 `"legacy" | "auto" | "2026-07-28"`（schema/mcp.ts:33-37）；roots 能力与 URL-elicitation complete 通知均标注 "Legacy era only... 2026-07-28"（client.ts:189-190、index.ts:240-241）。
- `Connection.modern`（client.ts:116, 337）暴露协议代次，供上层区分 legacy 会话语义。

## 相关篇

- [9_tool.md](9_tool.md) §5——MCP→Tool 桥接概览（本篇 §4/§5 为其协议侧下钻）
- [../2_tech-stack](../2_tech-stack/)——`mcp` 配置键在整体配置面的位置
- [11_permission.md](11_permission.md)——permission.assert 与 Ruleset 求值
- [4_session.md](4_session.md)——工具执行结果进入 Session Step 的结算路径
