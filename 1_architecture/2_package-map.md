# 包版图与依赖结构（Package Map & Dependency Structure）

> 一句话：**~40 个包按 `Schema → Core → Protocol → Server` 单向依赖组织成漏斗——契约层小而稳，表现层大而多变。**
> 数据来源：各包 `package.json` 实读 + 目录测量（2026-10-16，`dev` 分支）。
> [返回索引](README.md) | [上一篇：全局导览](1_overview.md) | [下一篇：规模数据 →](3_metrics.md)

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
