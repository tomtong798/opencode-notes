# 凭证体系详解（Credential 存储 → OAuth 尝试机 → provider 凭证解析 → Copilot/Console 专层）

> 一句话：**v2 的认证是一个通用的 "integration（认证方法注册表）+ credential（SQLite 凭证库）+ connection（激活选择）" 三件套——每个 provider 插件向 Integration 注册认证方法（OAuth device/authorization-code、API key、command、env），凭证以 JSON 值落库、per-integration 单激活，模型解析时 `connection.resolve()` 惰性取凭证（OAuth 临期自动刷新），另有独立于 LLM 体系的 server 访问密码（HMAC 会话）。**
> 代码基线：v2.0.30，dev 分支（e25b7abf89）；行号实测（2026-10-19）。
> 权威出处：`packages/schema/src/credential.ts` / `integration.ts`、`packages/core/src/credential.ts` / `integration.ts`
> 拆分说明：本篇原含"服务访问凭证"一节——2026-10 拆出（server/auth.ts 与本篇体系零 import 实证），见 [18.1_server-access.md](18.1_server-access.md)。本篇聚焦 LLM 侧两把钥匙：秘密怎么来（尝试机）→ 存哪（credential 表）→ 怎么选（优先级）→ Copilot/Console 两大消费者案例。
> [返回 README](README.md) | 前置 [3_model-protocol.md](3_model-protocol.md) §3（provider 凭证解析接缝：`fromCatalogModel(model, credential)`）、[1_data-layer.md](1_data-layer.md)（数据库层）| 姊妹篇 [18.1_server-access.md](18.1_server-access.md)（服务访问：零耦合的第三把钥匙）

---

## 0.5 通俗完整描述（先读这段再看细节）

"认证"在这个系统里其实是**三层不同的事**，本篇拆开讲：

**第一层：provider 凭证**（最常用）——各家 LLM 的 API key。存储优先级：数据库里的凭证（最近激活的在前）> 环境变量 > 无。注意是**明文落 SQLite**（没有 keychain 加密——本地优先的信任模型），文件权限 0600 兜底。**第二层：OAuth 流程**——opencode 账户/Copilot/ChatGPT 这类"登录获取 token"：三种流程并存（设备码轮询、本机回调页 PKCE、手输授权码），统一的状态机管并发/超时/刷新。Copilot 有专层因为它的 token 不过期、要查订阅额度、请求头还要改造。**第三层：服务访问凭证**（已拆至 [18.1](18.1_server-access.md)）——连本机 server 的那把密码，与本篇前两层零代码耦合。

容易误会的两点：①**没有独立的 auth 模块**——认证逻辑散在 integration（尝试机）/credential（存储）/各 provider 插件三处，"加一种登录方式=写一个 provider 插件"；②**packages/identity 不是身份认证**——它只是 logo 图片资产（常见误认）。

一句话：**provider key 的存取优先级、三种 OAuth 流程的状态机、服务访问的密码注入——三层各自独立又都服务于"把钥匙管好"**。
---

## 0. 模块地图

先说实测结论：**现码没有独立的 `auth/` 目录**。旧笔记 4.12 里的 `opencode/src/auth` 已不存在（`packages/opencode` 只剩构建产物，无 src），认证被拆成 core 的 Integration/Credential 服务 + 各 provider 插件注册的认证方法 + server 侧独立的访问控制，分散在五个位置：

```
packages/
├─ schema/src/                          ← 公共形状（无逻辑）
│   ├─ credential.ts    ★ Credential.Value 三态联合（oauth/key/external）+ 事件（79 行）
│   └─ integration.ts   ★ Integration.Method 五态（oauth/command/key/external/env）
│   │                     + Attempt 状态机形状（127 行）
├─ core/src/
│   ├─ credential.ts    ★ Credential 服务：凭证表 CRUD + 激活切换 + 事件（328 行）
│   ├─ credential/
│   │   └─ sql.ts         drizzle 表定义 credential（14 行）
│   ├─ integration.ts   ★ Integration 服务：方法注册表 + OAuth/command 尝试机
│   │                     + connection 解析/刷新（886 行，本篇主角）
│   ├─ integration/
│   │   └─ connection.ts  connection 的 key/投影辅助（26 行）
│   ├─ oauth/
│   │   └─ page.ts      ★ 回调页 HTML 模板（success/error/bootstrap，276 行）
│   ├─ account/
│   │   └─ sql.ts       ★ 遗留 account 表（LEGACY control_account，36 行；无活代码引用）
│   ├─ mcp/
│   │   └─ oauth.ts     ★ MCP OAuth：modelcontextprotocol SDK provider 适配
│   │                     + loopback 回调 + 刷新去重（532 行）
│   ├─ model-resolver.ts  ★ 消费方：resolveModel 时注入凭证（484 行）
│   └─ plugin/provider/   ← 认证方法的实际注册者
│       ├─ opencode.ts        ★ OpenCode Console：device flow + refresh + 远端配置（667 行）
│       ├─ github-copilot.ts  ★ Copilot：device flow + entitlement 校验 + 头改造（570 行）
│       ├─ chatgpt.ts           ChatGPT token sharing：PKCE + loopback 回调（544 行）
│       └─ configured.ts、config/plugin/provider.ts  env 方法注册（config 层）
├─ server/src/
│   ├─ auth.ts            ★ 服务访问凭证：basic auth + HMAC 签名会话（66 行）
│   └─ handlers/
│       ├─ credential.ts     HTTP API 暴露（66 行）
│       └─ integration.ts    （267 行）
├─ cli/src/
│   ├─ commands/handlers/auth/    opencode auth login/logout/switch 命令（8 文件 861 行）
│   └─ services/service-config.ts ★ service.json：本地服务注册 + password（357 行）
└─ identity/              ← 只有 6 个 logo SVG/PNG 资产，无任何代码（见 §6）
```

与 auth 无关而**不硬塞**的：

- `packages/core/src/github-copilot/` 目录本体（chat/ + responses/ 是 vendored AI SDK 适配器，README:5 明言 "It is not a generic OpenAI-compatible provider"，且 "Authentication … are owned by the provider plugin"）——归 §5 讲其与认证的接缝；
- `github-copilot/models.ts` 是模型目录同步，归 [3_model-protocol.md](3_model-protocol.md)；
- `packages/core/src/effect/`、`packages/enterprise/` 等不含认证逻辑。

## 1. 心智模型（三层认证图）

实测后我把"三层"修正为：**凭证获取层（怎么拿到秘密）→ 存储与选择层（秘密放哪、用哪个）→ 使用层（谁在什么时机消费秘密）**，再叠加一个完全独立的第四域：server 访问凭证。provider 调用时的凭证解析链：

```
provider 插件启动                      模型解析（每次 resolveModel）                 LLM 请求
───────────────────                  ──────────────────────────────             ──────────────
editor.method.update({           ┌─ ModelResolver.load (model-resolver.ts:374)
  oauth|key|env|command|...  ────┤   connection = Integration.connection
        │                         │                 .active(provider.integrationID)  :377-378
Integration 注册表                │   credential = connection.resolve(connection)   :379
  methods[]                       │      │
  implementations[methodID]       │      ├─ env      → process.env[name] 包成 Key
        │                         │      │            (integration.ts:724-727)
        ▼                         │      ├─ key/external → 原样返回        :730
OAuth/key/command 尝试机 ──落库──► │      └─ oauth → 未过期(>5min) 原样返回  :736-737
  (integration.ts:585-701)        │              临期 → impl.refresh()
        │     ▲ 订阅               │                     → credentials.update 回写 :738-740
        │     └─ Credential.Event.Switched ──► provider 插件 load() 重载
        ▼                         │      ▼
   Credential 表（SQLite）  ◄─────┘   fromCatalogModel(model, credential)
   per-integration 单 active            key→{apiKey}  oauth→{authToken/apiKey...}
   (credential.ts:115-145)              (model-resolver.ts:308-316) ──────► 出站请求
```

三个关键解耦（实测自源码注释）：

1. **注册与存储解耦**：`Integration.Interface.revision` 注释 "credential values remain owned by Credential"（integration.ts:171）——方法定义是插件态可热插拔，秘密只有 Credential 服务管。
2. **获取与使用解耦**：provider 插件在自己的 `load()` 里也走 `connection.active + resolve`（github-copilot.ts:173-176、opencode.ts:216-219），与模型解析共用同一条刷新路径，不存在第二套取凭证代码。
3. **连接身份 ≠ token 值**：`IntegrationConnection.key` 的 doc 注释 "labels and refreshed token values do not identify a new connection"（connection.ts:17）——刷新 token 不算换连接，UI/缓存以 `credential:{id}` 或 `env:{name}` 为准。

## 2. 凭证存储与解析

### 2.1 存储：SQLite 明文 JSON，无 keychain、无加密

实测 `credential/sql.ts:6-14`：

```ts
export const CredentialTable = sqliteTable("credential", {
  id: text().$type<Credential.ID>().primaryKey(),
  integration_id: text().$type<Credential.Info["integrationID"]>(),
  label: text().notNull(),
  value: text({ mode: "json" }).$type<Credential.Value>().notNull(),   // ← JSON 直存
  connector_id: text(),
  method_id: text(),
  active: integer({ mode: "boolean" }),
  ...Timestamps,
})
```

全仓库 grep `keychain|libsecret|encrypt` 零命中（仅测试与迁移文件提到 legacy auth.json）；唯一带保护的是 CLI 的 `service.json` 落盘 `mode: 0o600`（service-config.ts:97-100，文件权限而非加密）。**"凭证不落明文"不是本仓库的不变量**——明文落库、靠本地数据库文件属主保护，是明确的取舍（见 §8）。

### 2.2 Value 三态（schema/credential.ts:32-60）

```
Credential.Value（tagged union，按 type 判别）
├─ OAuth     { type:"oauth", methodID, refresh, access, expires, metadata? }
│            └─ expires: NonNegativeInt 毫秒时间戳；0 = 永不过期（Copilot 用）
├─ Key       { type:"key", key, metadata?, configuration?: Form.Answer }
│            └─ configuration 存登录表单答案；解析时叠加为运行时设置
│               （model-resolver.ts:202-203）
└─ External  { type:"external", methodID, metadata? }
             └─ 语义见原注释："References a credential source whose secrets
                and renewal are managed outside the credential store"
                （schema/credential.ts:50）——只存引用，秘密在别处
```

### 2.3 读写 API 与单激活不变量

`Credential.Interface`（credential.ts:37-61）：`all / list(integrationID) / get(id) / create / activate(id) / update(label|value) / remove(id)`。

核心不变量是 **per-integration 单激活**，全部在数据库事务里维护：

- `create`（:115-145）：查当前激活行 → 默认把新行设为激活（除非 `activate:false` 且已有凭证），先 `UPDATE ... SET active=false` 全清再插入；:131-133 处理 legacy 行——"Legacy rows have no active flag and the newest one is selected, so pin it before inserting a newer row"。
- `activate`（:172-207）：同 integration 全部置 false，目标行置 true。
- `remove`（:258-323）：删掉的若是激活行，则把剩余最新一行补位为激活（:275-292），并在事件里携带 replacement。

事件面：`create/activate/remove` 广播 `credential.updated`（global）+ `credential.switched`（携带 integrationID/credentialID，:163-169、:205、:316-322）。provider 插件正是靠订阅 `Switched` 触发重载（§5）。

### 2.4 解析优先级（实测）

不是简单的 config > env > 存储。连接列表的构造在 `resolveConnections`（integration.ts:396-410）：

```
resolveConnections(entry, savedCredentials)
├─ 存储凭证：saved.map(→{type:"credential",...}).toReversed()   ← 时间新者在前
└─ env 连接：entry.methods 里 type==="env" 的方法
             .flatMap(names.filter(name => process.env[name]))  ← 变量为空不出现(:407)
`connection.active` 取列表第一项（:719-722）
```

即优先级为：**存储凭证（新 > 旧，active 最优先，排序键 `asc(active), asc(time_created), asc(id)` 见 credential.ts:91、100）> 声明的 env 变量 > 无凭证**。env 方法的注册来源是 config 里 providers 的 `env: [...]` 声明（config/plugin/provider.ts:35-40）。特殊兜底：zen/opencode 在无任何凭证时回落 `apiKey: "public"` 并禁用付费模型（opencode.ts:339-354）。

## 3. OAuth 流程

### 3.1 统一抽象：OAuthAuthorization 两种模式

integration.ts:60-73：

```
OAuthAuthorization
├─ { mode:"auto",   url, instructions, callback: Effect<Credential.OAuth> }
│    └─ 发起后自动 fork 等待完成（device flow / loopback 回调）
└─ { mode:"code",   url, instructions, callback: (code) => Effect<Credential.OAuth> }
     └─ 用户手动粘贴授权码（无浏览器场景；complete 缺 code 抛 CodeRequiredError :122-124）
```

实测三种具体流程并存：

| 流程 | 使用方 | 机制与关键行 |
|---|---|---|
| Device flow（auto） | OpenCode Console（opencode.ts:118-155，`POST {server}/auth/device/code`，server 默认 `https://opencode.ai/console` :21-22）；GitHub Copilot（github-copilot.ts:82-155，`POST https://{domain}/login/device/code`，clientID `Ov23li8tweQw6odWQebz` :17） | 轮询 token 端点；`authorization_pending` 继续等、`slow_down` 按 server 给的 interval 退避（copilot:140-144，统一加 `pollingSafetyMargin = 3000` ms :20） |
| Authorization code + PKCE + loopback（auto） | ChatGPT token sharing（chatgpt.ts:111-186，issuer `auth.openai.com` :26-29）；MCP（mcp/oauth.ts:357-392） | `node:http` 临时 server `listen(0, "127.0.0.1")`（chatgpt:200-201 懒加载以兼容 workerd），redirect `http://127.0.0.1:{port}/auth/callback`；ChatGPT 校验 state + nonce + id_token 且要求 token sharing scope（:174-186）；MCP 支持 CIMD 免注册（client metadata URL `https://opencode.ai/oauth/opencode/client.json`，mcp/oauth.ts:27-28） |
| code（手输） | MCP 无浏览器环境 | complete 带 code 走 `callback(code)`（integration.ts:829-832） |

### 3.2 尝试机（attempt machine）

OAuth 的状态管理核心在 integration.ts:261-302、585-701：

```
attempt 状态机（内存态，SynchronizedRef 防并发）
  pending ──callback 成功──► persisting ──createCredential──► complete ──1min──► 删除
     │                          │                                （terminalRetention :262）
     ├─callback 失败──────────► failed ──1min──► 删除
     ├─10min 未完成（scrub 每 30s 扫）──► expired ──1min──► 删除
     └─cancel（persisting 中不可取消，:484 注释：
        "Persisting attempts cannot be cancelled, expired, or claimed again"）

时长常量（:261-263）：attemptLifetime=10min / terminalRetention=1min / scrubInterval=30s
并发位：completing（防 auto/code 双路重复完成，:818-820）
        persisting（落库中，免疫 cancel/expire）
```

`connect`（:585-635）：查 method 实现 → 校验表单（Form.validateFields/validateAnswer :596-599）→ fork attemptScope → 跑 `authorize(answer)` 得 authorization → 登记 pending；auto 模式立即 fork callback，出口进 `settle`。`settle`（:435-488）把成功 token 经 `createCredential` 落库（label 可由 implementation 的 `label(credential)` 派生，如 Console 用 orgName，opencode.ts:180-181），失败保留 message 供 `status` 查询。

`command` 方法走平行的一套（:637-701）：spawn 配置的命令，stdout 即凭证（trim 后包成 `Credential.Key`，:690-692、:515），stderr 聚为进度 message，非零退出报错（:686-688）。

### 3.3 回调页（oauth/page.ts）

品牌化 HTML 模板库，文件头注释（:1-10）："served by the loopback HTTP servers that finish an OAuth exchange (MCP, Codex/ChatGPT, xAI, Snowflake, ...)"，"fully self-contained HTML string with no external assets"。三个页面：

- `success` / `error`（:21-37）：结果卡片，success 2.5s 自动关窗；
- `bootstrap`（:60-66）：处理 implicit grant——凭证在 URL fragment 时，页面脚本读 fragment 后 POST 回同源 `tokenPath`，原地切换成功/失败态。

### 3.4 Token 刷新（统一惰性路径）

`connection.resolve`（integration.ts:723-741）：

```
resolve(connection)
├─ env            → process.env[name] 有则包 Key，无则 undefined（:724-727）
├─ 非 oauth       → 原样返回（:730）
└─ oauth
   ├─ implementation 无 refresh 能力 → 原样返回（:735）
   ├─ expires > now + 5min           → 原样返回（:736-737）
   └─ 临期 → authorize(impl.refresh(value)) → credentials.update 回写 → 返回新值（:738-740）
```

refresh 是 OAuthImplementation 的**可选**字段（integration.ts:79）：Console 实现了它（`POST /auth/device/token`，grant_type=refresh_token，opencode.ts:151-172）；Copilot 故意不实现（token 长效，见 §5）。

## 4. opencode 账户

"opencode 账户" = **OpenCode Console 的 OAuth 连接**，不是独立子系统。`opencode.ts` 注册 integration `opencode`（显示名 "OpenCode Console"，:257-259），device flow 到 `https://opencode.ai/console`（clientID `"opencode-cli"`，:21-22）。

凭证之外，Console 连接拉回的是**整份组织级配置**（`RemoteResponse`，:44-56）：

- **providers**：组织开通的 provider/模型目录，逐项 `mergeOverlay` 且全部过 `withoutCredentials`（:313-315 等多处）——组织配置永远带不进秘密，甚至显式 `delete model.settings.baseURL`（:316）防远程指路；
- **websearch**：注册一个走 Console 的搜索后端，执行时实时 `connection.resolve` 取 token 并带 `x-org-id`（:381-419）；
- **mcp.servers**：只接受 remote 类型——"the Console can never make the client run a command"（:47-48）；`auth: "console"` 让客户端把**自己的** Console 凭证附到该 server 请求头（:412-417）；
- **policies**：组织策略语句，喂给 ManagedPolicy（:258-261），随快照替换——"a credential switch, disconnect, or 404 replaces them too"（:263 注释）。

订阅计划的体现是**无凭证时的降级**：没有任何 key/连接时 zen provider 回落 `apiKey: "public"` 并禁用付费模型（:339-354）。

SSO 与登出是两个显式错误类型：

- `SsoRequiredError`（Console 403：组织强制 SSO 而 proof 过期，:77-96）→ connection 标 `needs_auth` 状态并给出 SSO 重连 URL（:534-544）；
- `SignedOutError`（401 或 refresh 拿到 400 `invalid_grant`，:152-156）——"only a new device login recovers"（:91 注释）。

推理时的 401/403 响应经 `http.response` hook 被 `explain` 拦截：先 `check()` 重拉 Console 配置让状态落地，再把响应体替换成带指引的 `authentication_error`（:467-484）。

## 5. GitHub Copilot 专层

Copilot 需要专层的原因，实测是三层叠加：

1. **认证本身特殊**（plugin/provider/github-copilot.ts）：
   - device flow（`{domain}/login/device/code`，scope 仅 `read:user`，:85）拿到的 GitHub OAuth token **不是** Copilot API token；
   - token 直接当长效凭证存：`refresh: access, access, expires: 0`（:125-128）——**expires=0 让统一刷新逻辑（§3.4 的 5 分钟阈值）永远跳过**，无刷新轮换；
   - 支持 GitHub Enterprise：登录表单选 deploymentType/enterpriseUrl（:54-73），domain 归一后 API 换 `https://copilot-api.{domain}`（:371-373）。
2. **entitlement 门槛**：GitHub OAuth 对任何账号都成功，:375-376 注释直言 "OAuth itself succeeds for any GitHub account, so this is the only signal that the account can chat"——必须额外打 `api.github.com/copilot_internal/user` 查 `chat_enabled`；且查询失败**不**拦截（:115-117 注释："a failed or malformed lookup must not turn a GitHub hiccup into a denial"）。user 响应里的 `endpoints.api` 存进 metadata 作 API 入口。
3. **请求协议改造**：所有出站请求要套 Copilot 专属头——`Openai-Intent / X-GitHub-Api-Version(2026-08-01) / x-initiator / X-Interaction-Type / Copilot-Vision-Request`（applyHeaders :487-508；由 requestMetadata 解析请求体判定 agent/vision :521-566；x-initiator "can only ever escalate to agent, never back to user" :502-505）。实现方式是 `aisdk "sdk"` hook 换装：`copilotFetch` 包装 fetch + 用 `github-copilot/copilot-provider.ts` 的 `createOpenaiCompatible` 替换默认 SDK 实例（:264-277）。**`github-copilot/` 目录就是为这个被换装的 SDK 实例存在的**（vendored AI SDK chat/responses 适配器，README:5-7 明确其 provenance 与 ownership 边界）。

模型路由随连接动态化：登录后从 Copilot API 拉实时模型列表 `CopilotModels.load`（:187-200，Bearer 用 `credential.refresh`），并订阅 `Credential.Event.Switched` 触发 `refresh → provider.reload`（:258-263）；会话标题走免费 utility 模型（gpt-4o-mini）的独立 `/chat/completions` 调用（:429-474，注释 :421-428 详述了为何绕开 core 的 title hook 而不自造 seam）。

## 6. Identity 包的边界

实测 `packages/identity/` **只有 6 个品牌 logo 文件**（mark.svg / mark-light.svg / mark-96/192/512×2.png；`ls` 确认无 package.json、无 src/），全仓库 grep `@opencode/identity` 零引用。它既不是设备身份也不是遥测匿名 ID——就是 opencode 品牌 mark 的资产存放目录。**本篇将其划出认证边界**；若其他卷引用 "identity 包"，应指此资产目录。core 里最接近"设备身份"的是 `chatgpt-agent-host-id`（chatgpt.ts:102-105：`urn:uuid:{randomUUID}` 持久化在 plugin storage），但那是 ChatGPT 动态客户端注册的 host 标识，非通用身份层。

## 7. 服务访问凭证（已拆至 18.1）

本节已拆至 [18.1_server-access.md](18.1_server-access.md)——与 LLM 凭证体系零代码耦合（server/auth.ts 对 credential/integration 零 import），2026-10 拆分归位。

## 8. 不变量与设计取舍

- **凭证明文落库**，靠本地 SQLite 文件属主保护——没有 keychain 抽象，是取舍而非遗漏（全仓库无 keychain/加密引用）。
- **单激活**：一个 integration 同时只有一个 `active` credential；create/activate/remove 全在事务里维护，remove 自动补位。
- **组织配置永不含秘密**：Console 的 providers/models/policies 配置全部过 `withoutCredentials`，且显式删远程 baseURL。
- **token 刷新不改连接身份**（connection.ts:17）；**credential 值归 Credential、方法定义归 Integration**（integration.ts:171）。
- **Console 远程 MCP 只接受 remote 类型**——远程配置不能注入本地命令执行（opencode.ts:47-48）。
- **entitlement 查询失败 ≠ 拒绝**（github-copilot.ts:115-117）：只有明确的 `chat_enabled:false` 拦登录。
- **失败加载保旧配置**：Console 加载失败且连接未变时保留上个快照——"dropping it would lift organization policy while personal credentials keep working"（opencode.ts:247-249）；只有 "well-formed Console errors" 才改连接状态，其他可能是瞬态（:240-242）。
- **Copilot token 不过期**（expires: 0）换取无刷新的简单性，代价是 revoke 只能靠重新登录。

## 9. 失败与恢复

- **刷新竞态（MCP，最完整）**：refresh token 会轮换，并发刷新同一 token 第二个请求会拿 `invalid_grant`——mcp/oauth.ts:30-35 用模块级 `refreshes` Map 把同 key（`url+client_id+refresh_token`，:33-35）的请求合并为一个 in-flight Promise。落库侧 `connectProvider` 的 invalidate 只在行里仍是自己 presented 的 token 时才删（:288-292："only drop it while it still holds ours"），防止误删已轮换后的新凭证。
- **刷新竞态（一般 OAuth）**：统一路径靠 5 分钟提前量缩小窗口（integration.ts:737），Integration 层**没有** MCP 那样的跨请求去重——依赖各插件 `Semaphore.makeUnsafe(1)` 串行 load（github-copilot.ts:164、opencode.ts:196）在插件层兜底。
- **Console 登出**：refresh 收 400 `invalid_grant` → `SignedOutError` → 连接标 `needs_auth`，推理响应被改写为 "Reconnect OpenCode Console to continue"（opencode.ts:89-90、152-156）。
- **凭证缺失的报错路径**：模型解析时无凭证且 provider 是 `usesAPIKeyAuth` 包 → route 降级 `Auth.none`（model-resolver.ts:390-396），让 SDK 自己抛"缺 API key"而不是解析器造错；env 方法变量为空 → 连接列表不出现该项（integration.ts:407），resolve 返回 undefined。
- **attempt 过期/清理**：10 分钟未完成的 OAuth/command 尝试由 scrub 标 `expired` 并关 scope（:536-557、559-580）；persisting 中的尝试免疫取消（:484）。
- **ChatGPT 回调竞态**：第二个回调 `Deferred.succeed` 失败得 409（chatgpt.ts:137-139）；回调 clientID 与存量不一致报错要求重连（:171-173）；验证失败/缺 scope 都给出针对性的下一步指引（:174-186）。

## 演进注记

旧笔记 4.12（2026-08-03）记载 "auth = opencode/src/auth + core/src/credential/oauth/account，V1 为主"。现码（v2.0.30 dev）实测四点：

1. **auth 已完全迁出 `opencode/src`**——`packages/opencode` 无 src，CLI 认证命令在 `packages/cli/src/commands/handlers/auth/`（login/logout/switch/list/export/import，8 文件 861 行）。
2. **V1 遗留只剩两处化石**：`account/sql.ts` 的 `ControlAccountTable` 标着 `// LEGACY`（:21），全仓库无活代码引用（"opencode 账户"已并入 Console integration 凭证，access_token/refresh_token 直接住在 credential 表）；迁移 `20260805200742_import_legacy_credentials.ts` 把 V1 的 `auth.json`（oauth/api/wellknown 三态）一次性导入新 credential 表——wellknown 类型在新 Value 里没有对应位，迁移时按来源特殊处理。
3. **`oauth/` 目录已退化**：只剩 `page.ts` 回调页模板——流程逻辑全部上收到 `integration.ts` 的通用尝试机和各 provider 插件，不再有共享的 OAuth 客户端模块。
4. **github-copilot 与 auth 的关系重画**：认证在 `plugin/provider/github-copilot.ts`，`github-copilot/` 目录只是被 aisdk hook 换装的 vendored SDK。

## 相关篇

- [3_model-protocol.md](3_model-protocol.md)——provider 目录与 `fromCatalogModel` 的凭证注入细节（本篇 §1 只画解析链）
- [1_data-layer.md](1_data-layer.md)——credential 表所在的 SQLite/Effect Layer 基座
- [13_plugin.md](13_plugin.md)——provider 插件的 transform/hook 机制（本篇 §4/§5 大量使用）
- [11_permission.md](11_permission.md)——ManagedPolicy（Console 组织策略的消费端）
