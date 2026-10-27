# 包版图与依赖结构（Package Map & Dependency Structure）

> 一句话：**~40 个包按 `Schema → Core → Protocol → Server` 单向依赖组织成漏斗——契约层小而稳，表现层大而多变。**
> 数据来源：各包 `package.json` 实读 + 目录测量（2026-10-16，`dev` 分支）。
> [返回索引](README.md) | [上一篇：全局导览](1_overview.md) | [下一篇：规模数据 →](3_metrics.md)

---

## 0. 读图须知：包协作的最终结论

**启动链**：desktop（壳）用 client **拉起** cli（二进制）〔内含 server→core〕；另一侧 app（界面）用 client **直连** server。

**请求链**：app 收集输入并乐观渲染 → client 发请求、收 SSE 事件流 → server 翻译协议（HttpApi → core 服务）→ core 干全部的活（inbox 准入 → wake/drain → Step 执行），临场调 ai（LLM）、tool（工具，先过 permission 断言）、plugin（干预点钩子）。

三条配套事实：

1. **client 是"两处都在用"的连接件**——主进程用其 `service` 形态（发现/拉起服务），renderer 用其 `solid` 形态（前端数据层本体：SSE 事件 → Solid store 投影、断线重连、outbox 乐观跟踪）。client/solid 是数据层本体而非"一个连接"。
2. **session 创建与请求准入是两个时机**——新会话先独立调一次 `session.create`；每次请求只做 inbox 准入（admit 落 durable 行），`Session.prompt` 是准入不是创建。
3. **cli/desktop 在请求期不参与**——cli 只是服务进程的出生方式（spawn 后即服务本身）；desktop 仅剩隐形职责（Basic 头注入）。

> 展开图景见 §4.5（启动链）/ §4.6（请求链）。

---

## 1. 依赖铁律（实测成立）

```
schema（契约：领域形状 + durable 事件）
   ↓
core（引擎）──依赖──→ ai, codemode, plugin, plugin-browser, util
   ↓
protocol（HttpApi 定义）──依赖──→ schema
   ↓
server（handlers）──依赖──→ core, protocol, schema, simulation, util

client ──依赖──→ 仅 protocol + schema（铁律成立：实测无 core/server）
sdk   ──依赖──→ client + core + plugin + schema + server（组合层）
```

### 两个"越界"依赖（有意为之，值得记住）

| 边 | 原因 |
|---|---|
| **tui → core**（不止 client） | TUI 支持 standalone 形态：在进程内起 server 时需要 core。这使 TUI 与 app 本质不同——app 是纯远程客户端，TUI 双形态皆可 |
| **core → plugin-browser** | "浏览器插件"的 RPC 基础设施被引擎复用，名不副实的依赖方向 |

### 完整依赖声明表（@opencode/* 部分，package.json 实读）

| 包 | 依赖 |
|---|---|
| protocol | schema |
| core | ai, codemode, plugin, plugin-browser, schema, util |
| server | core, protocol, schema, simulation, util |
| client | protocol, schema |
| sdk | client, core, plugin, schema, server, util |
| cli | client, plugin, schema, server, tui, util（+ @opentui/*） |
| tui | client, **core**, latex, merman, plugin, schema, simulation, theme, ui, util（+ @opentui/*） |
| app | client, gui-extensions, schema, session-ui, ui, util |

---

## 2. 分层漏斗（规模与稳定性的分层管理）

```
┌─ 契约层 ────────────────────────────────────────────┐
│ schema（79 文件）protocol（36）server（52）           │ 小而稳定，变更成本高
│    ↓ 改动触发全仓重新 generate                        │
├─ 引擎层 ────────────────────────────────────────────┤
│ core（425 文件 / 68k LOC）ai（217 / 32k）             │ 中等，复杂度集中在 session/
│ codemode（45 / 11k）                                 │
├─ 组合层 ────────────────────────────────────────────┤
│ client（32）sdk（15）cli（116 / 13k）                 │ 薄
├─ 表现层 ────────────────────────────────────────────┤
│ app（534 / 131k）gui-extensions（845 / 61k）          │ 巨大，变化最快
│ tui（284 / 62k）ui（216 / 36k）session-ui（115 / 26k）│
├─ 边缘层 ────────────────────────────────────────────┤
│ 营销站（web/posts/services/*）内部运营（console/stats）│
│ IDE（sdks/vscode）桌面壳（desktop）                   │
├─ 空壳（忽略）───────────────────────────────────────┤
│ effect-drizzle-sqlite, effect-sqlite-node, enterprise,│
│ llm, opencode, sdk-next, slack, identity…            │
└─────────────────────────────────────────────────────┘
```

**漏斗的意义**：契约层几乎不动（动则全仓重生成），表现层天天动（动则互不影响）——依赖方向铁律 + 规模漏斗 = 变更频率与稳定性的分层管理。

---

## 3. 包清单（按层分组）

### 契约与 API 层

| 包 | 文件/LOC | 职责 |
|---|---|---|
| **schema** | 79 文件 | 领域契约单一真相源：Session/Agent/Model/Permission… + `durable-event-manifest`（对外 ServerDefinitions / 对内 Definitions） |
| **protocol** | 36 / 4.3k | HttpApi 定义：~30 Group、27+ 标签错误、SSE 端点、模拟协议 |
| **server** | 52 / 4.1k | Effect handlers（28 个模块，全部薄委托）+ EventFeed + 进程管理 |

### 引擎层

| 包 | 职责 |
|---|---|
| **core** | 事件溯源引擎：session/agent/tool/provider/config/permission/mcp/worktree…（37 个顶层服务目录） |
| **ai** | LLM provider 抽象：~50 家统一流式接口、Bedrock/Vertex 认证 |
| **codemode** | Acorn 沙箱解释器：安全执行 AI 生成的代码 |
| **plugin** | 插件 SDK（Effect + Promise 双 API） |

### 组合层

| 包 | 职责 |
|---|---|
| **client** | 生成客户端（promise/effect/solid 三风味）+ 服务发现 |
| **sdk** | 嵌入式组合：EmbeddedHost（进程内 server + `:memory:` DB） |
| **cli** | `opencode` 命令树 + 服务连接解析 + Vite 通道 |

### 表现层

| 包 | 职责 |
|---|---|
| **tui** | 终端 UI（OpenTUI + SolidJS，含独立的 mini 第二 UI） |
| **app** | Web 应用（也是桌面渲染器） |
| **gui-extensions** | 桌面/web 特性集中营：browser/ssh/wsl/review/office… 14 个垂直模块 |
| **ui / session-ui / theme** | 共享组件库 / 会话时间线 / TUI 主题 |
| **desktop** | Electron 壳（复用 app 渲染器） |

### 边缘与空壳

详见 [overview.md §5](1_overview.md#5-其余版图按重要性)。空壳目录（`llm`、`sdk-next`、`slack` 等）只含 node_modules 或品牌资产，**读代码时直接跳过**。

---

## 4. 一个包的内部解剖入口

想进某个包深挖时的入口文件速查：

| 包 | 入口 |
|---|---|
| core | `src/session.ts`（门面）→ `src/session/session.ts`（工厂） |
| protocol | `src/api.ts`（组装）→ `src/groups/*.ts`（30 Group） |
| server | `src/routes.ts` → `src/handlers.ts` → `src/handlers/*.ts` |
| client | `script/build.ts`（生成）→ `src/promise/`、`src/effect/`、`src/solid/data.ts` |
| tui | `src/app.tsx`（25+ provider）→ `src/routes/session/` |
| app | `src/index.ts` → `src/runtime/`（161 文件的基础设施） |

---

## 4.5 桌面启动链的包协作（结论版）

"desktop → app → client → cli → server → core"式的单线串联是**错的**——它混淆了进程与包，且方向有反。正确图景是**两个进程、三条依赖链、一次 spawn 边、一次网络边**：

```
进程 A：桌面（Electron）                进程 B：后端服务
─────────────────────────              ─────────────────
desktop 主进程
  │ import ①
  ▼
client/service ──── spawn ──────────►  cli（二进制宿主，detached）
（discover/ensure/注册文件协议）           └ 装配 server（HTTP 门面）
                                             └ 委托 core（引擎）
desktop 主进程 ◄──轮询注册文件拿 {url,密码}      ├ schema/protocol
  │（密码止步主进程，只传 url 给 renderer）       ├ ai
  ▼                                            └ plugin/util
app（renderer 的本体）
  │ import ②
  ▼
client/solid ════ HTTP + SSE 直连 ════════════► server
（store 投影/断线重连；请求经主进程注入 Basic 头）
```

**最终结论**：

1. **启动不是一条链**——desktop（壳）用 client **拉起** cli（二进制）；cli 进程内装配 server 承载 core；另一侧 app（界面）用 client **直连** server。两条线各管一段。
2. **client 是"两处都在用"的连接件，不是链上一站**——主进程用其 `service` 形态（发现/拉起服务），renderer 用其 `solid` 形态（SSE → store 投影）。
3. **cli 只在启动那一刻有意义**——spawn 完成后它就是服务进程本身（server/core 的宿主），运行期没有任何流量"经过 cli"。
4. **运行时数据流 ≠ 依赖流**——app 的请求直接打到 server（HTTP+SSE），不经过 desktop 转发，也不经过 cli。
5. **sdk 不在此链上**——它是进程内嵌入方案（EmbeddedHost + `:memory:` DB），给外部应用嵌入用；桌面走的是真实进程 spawn。`gui-extensions`/`ui`/`theme` 是界面特性阶段的事，启动链不经过。

> 一句话记忆：**desktop 用 client 拉起 cli〔内含 server→core〕；app 用 client 直连 server。**
> 桌面启动的时序细节（T0-T11）见 [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1/§1.5；spawn 的真实位置（client/service-contender）见同篇 §1.5.8。

---

## 4.6 一次用户请求的包协作（结论版）

用户在桌面敲下回车到流式渲染完成，包视角的完整协作。与 4.5 同构：**前半段复用已建好的连接，后半段全部发生在服务进程内**：

```
桌面进程（已启动）                        服务进程（常驻）
─────────────────────────            ─────────────────────
app（composer/submit.ts）
  │ 乐观 admitLocal → 界面先显示
  ▼
client/solid（data.ts session.prompt）
  │  outbox 跟踪 + 同会话串行发送
  │  POST /session/prompt（铸 id + delivery）
  ▼ ══════════ HTTP ════════════════════► server
（请求经 desktop 主进程注入 Basic 头；         │ handlers/session.ts
 HTTP 秒回 admitted——准入与执行分离）          ▼ 委托
                                            core
                                              ├ Session.prompt（幂等 reconcile
                                              │   + inbox 准入落 durable 行）
                                              ├ wake → drain（runner/llm.ts）
                                              │   └ Step 组装请求
                                              ├ ai ◄── llm.stream（流式调用）
                                              │        └ provider 适配层（~50 家）
                                              ├ tool/plugin/*（工具执行 fiber 并行；
                                              │   permission 先 assert）
                                              └ plugin hooks（session.prompt 等干预点）
                                              │
  ▲ ◄══════════ SSE 事件流 ══════════════════ bus → event-feed
  │                                             （text/tool/step 事件 → 帧）
client/solid（connection.ts 收流 10ms 批）
  ▼
app（store.produce → Solid 细粒度渲染）
```

**最终结论**：

1. **参与包恰好 5 个**：`app → client ═HTTP/SSE═► server → core`（core 再临场调 `ai`/`plugin`）。`desktop` 只剩一个隐形职责——Basic 头注入；`cli` 完全不参与（它只是这个服务进程的出生方式）。
2. **两次交接定义了整个流程**：HTTP 请求（秒回，准入与执行分离）和 SSE 事件流（渲染驱动）。二者之间服务进程内部自由运转。
3. **core 是唯一的"干活者"**——准入、执行、工具、权限全在它的 Location 服务树内；server 只做协议翻译（HttpApi → Effect 服务），无业务逻辑。
4. **plugin 在此链上**（与启动链不同）——`session.prompt`/`tool.execute.before` 等干预点在请求路径上被逐个触发。
5. **渲染闭环在 app**——client/solid 把事件投影进 store，Solid 细粒度响应只重渲染受影响的组件。

> 一句话记忆：**app 收集输入并乐观渲染，client 发请求收事件流，server 翻译协议，core 干全部的活，ai/plugin 临场被调。**
> 全链路时序细节见 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)（含每跳的文件:行号）。

---

## 5. 设计取舍

- **为什么 client 禁止依赖 core/server**：client 要独立发布给外部应用嵌入（sdk 的组合层才能轻）；也是防止领域逻辑泄漏进传输层。
- **为什么 tui 例外直依赖 core**：standalone 形态（`--standalone`）在 TUI 进程内起 server，省去独立服务进程——终端用户"单命令即用"的体验优先于分层纯洁性。
- **为什么空壳目录还留着**：包名占位（未来用途）+ workspace 拓扑稳定；但它们不参与构建。
- 红线出处：根 `AGENTS.md` 第 2 条（依赖方向）、`specs/v2/README.md` Authority 表。

## 相关篇

- [overview.md](1_overview.md) — 各包内部长什么样
- [metrics.md](3_metrics.md) — 规模数据的完整测量
- [../2_tech-stack/2.6_architecture-styles/layered-monorepo.md](../2_tech-stack/2.6_architecture-styles/4_layered-monorepo.md) — 分层铁律的技术栈视角
- TODO: extension-points.md — 加新包/新能力时的落点
