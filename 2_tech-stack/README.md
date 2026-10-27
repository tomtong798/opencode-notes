# OpenCode 技术栈笔记（Tech Stack Notes）

> 对象：`/home/donghe/github/opencode`（v2.0.30，分支 `dev`）。
> 组织方式：**一项技术一个 md 文件**，按 6 大维度分目录（`2.1`~`2.6`）；文件序号 = 目录内推荐阅读顺序，`x.y` = 从属关系（如 `3.1_effect-schema.md` 隶属 `3_effect.md`）。
> 每篇统一结构：**是什么 / 业界类比 / 在 OpenCode 中的角色 / 与其他技术的关系 / 相关源码**。

OpenCode 不是典型 Web 应用，而是**本地优先（local-first）的 AI 编程 agent 工具**——"前端"是四个客户端（终端/Web/桌面/VS Code），"后端"是本机常驻服务进程。

---

## 目录索引

### 1. 客户端层（`2.1_client/`）

| 文件 | 技术 | 一句话 |
|---|---|---|
| [1_typescript.md](2.1_client/1_typescript.md) | TypeScript | 全仓统一语言，strict 风格（禁 `any`） |
| [2.0_reactivity.md](2.1_client/2.0_reactivity.md) | 响应式编程 | 概念前置：拉/推模型、四大流派、Signals 收敛 |
| [2_solidjs.md](2.1_client/2_solidjs.md) | SolidJS | **家族主篇**：细粒度响应式 UI 框架，四端共享的"组件语言" |
| [2.1_solidjs-reactivity.md](2.1_client/2.1_solidjs-reactivity.md) | └ SolidJS 响应式机制 | 实战：signal/store 三件套/batch/keyed（opencode 现码样例） |
| [3_opentui.md](2.1_client/3_opentui.md) | OpenTUI | 自研终端渲染框架（含 kitty 协议、keymap） |
| [4_vite.md](2.1_client/4_vite.md) | Vite 8 | Web/桌面端构建工具 |
| [5_tailwind-css.md](2.1_client/5_tailwind-css.md) | Tailwind CSS 4 | 原子化样式 |
| [6_kobalte.md](2.1_client/6_kobalte.md) | Kobalte | SolidJS headless 组件库（行为） |
| [7_tanstack-solid-query.md](2.1_client/7_tanstack-solid-query.md) | TanStack Solid Query | Web 端服务端状态管理 |
| [8_electron.md](2.1_client/8_electron.md) | Electron 44 | **家族主篇**：opencode 的桌面壳 |
| [8.1_electron-fundamentals.md](2.1_client/8.1_electron-fundamentals.md) | └ Electron 基础 | 多进程模型/Preload 桥/安全基线/打包更新 |
| [9_esbuild.md](2.1_client/9_esbuild.md) | esbuild | VS Code 扩展构建内核 |
| [10_pwa.md](2.1_client/10_pwa.md) | PWA | Web 端可安装/离线增强 |

### 2. 服务端层（`2.2_server/`）

| 文件 | 技术 | 一句话 |
|---|---|---|
| [1_js-runtime.md](2.2_server/1_js-runtime.md) | JS 运行时 | 什么是运行时/与 OS 的关系/运行时解剖（入门篇） |
| [2_bun.md](2.2_server/2_bun.md) | Bun | 三位一体 JS 工具链：运行时+包管理+测试 |
| [3_effect.md](2.2_server/3_effect.md) | **Effect-TS（家族主篇）** | 函数式效应系统：DI/Fiber/类型化错误——含家族导航 |
| [3.1_effect-schema.md](2.2_server/3.1_effect-schema.md) | └ Effect Schema | 契约单一真相源：类型+校验一体 |
| [3.2_effect-httpapi.md](2.2_server/3.2_effect-httpapi.md) | └ Effect HttpApi | 声明式 API 定义（~30 Group） |
| [3.3_effect-cli.md](2.2_server/3.3_effect-cli.md) | └ effect/cli | `opencode` 命令树 |
| [3.4_effect-guide.md](2.2_server/3.4_effect-guide.md) | └ Effect 学习指南 | 教程级：为什么/怎么用/Go 类比（含 4.0.1 实测频率） |
| [3.5_effect-api.md](2.2_server/3.5_effect-api.md) | └ Effect API 速查 | 分模块 API 表 + Go 对照 + 自建 LayerNode/LayerMap 专节 |
| [7_sse.md](2.2_server/7_sse.md) | SSE | 默认实时事件通道（单向推送） |
| [8_websocket.md](2.2_server/8_websocket.md) | WebSocket | 双向场景专用：PTY 终端流/LLM 传输/会话分享 |
| [9_pty.md](2.2_server/9_pty.md) | PTY | 伪终端机制：终端面板/持久终端的底层（含与模拟器/shell 的辨析） |
| [10_httpapi-codegen.md](2.2_server/10_httpapi-codegen.md) | httpapi-codegen | 契约 → 生成类型化客户端 |
| [11_opencode-ai.md](2.2_server/11_opencode-ai.md) | @opencode/ai | ~50 家 LLM 供应商统一抽象 |
| [12_codemode.md](2.2_server/12_codemode.md) | CodeMode | Acorn 沙箱解释器执行 AI 代码 |

### 3. 数据层（`2.3_data/`）

| 文件 | 技术 | 一句话 |
|---|---|---|
| [1_sqlite.md](2.3_data/1_sqlite.md) | SQLite（WAL） | 本地状态唯一主存储 |
| [2_drizzle-orm.md](2.3_data/2_drizzle-orm.md) | Drizzle ORM | TS-first 轻量 ORM |
| [3_event-sourcing.md](2.3_data/3_event-sourcing.md) | 事件溯源 | 存事实不存状态；write-ahead claim |
| [3.1_cqrs-projection.md](2.3_data/3.1_cqrs-projection.md) | CQRS 投影 | 事件同事务折叠成读模型 |
| [5_planetscale.md](2.3_data/5_planetscale.md) | PlanetScale | stats 分析数据（内部） |

### 4. 基础设施与部署（`2.4_infrastructure/`）

| 文件 | 技术 | 一句话 |
|---|---|---|
| [1_cloudflare-workers.md](2.4_infrastructure/1_cloudflare-workers.md) | Workers | 边缘 serverless 计算 |
| [2_cloudflare-d1.md](2.4_infrastructure/2_cloudflare-d1.md) | D1 | Cloudflare 的 serverless SQLite |
| [3_durable-objects.md](2.4_infrastructure/3_durable-objects.md) | Durable Objects | 强一致同步原语（会话分享） |
| [4_sst.md](2.4_infrastructure/4_sst.md) | SST | TS 定义云基建 |
| [5_github-actions.md](2.4_infrastructure/5_github-actions.md) | GitHub Actions | CI/CD |
| [6_docker.md](2.4_infrastructure/6_docker.md) | Docker | 仅 CI 预烘镜像 |
| [7_opentelemetry.md](2.4_infrastructure/7_opentelemetry.md) | OpenTelemetry | 厂商中立遥测标准 |
| [8_honeycomb.md](2.4_infrastructure/8_honeycomb.md) | Honeycomb | 观测后端 |
| [9_xdg-base-directory.md](2.4_infrastructure/9_xdg-base-directory.md) | XDG Base Directory | Linux 路径规范：opencode 全局路径的派生源 |

### 5. 开发与协作工具（`2.5_tooling/`）

| 文件 | 技术 | 一句话 |
|---|---|---|
| [0.1_ts-running-guide.md](2.5_tooling/0.1_ts-running-guide.md) | └ TS 运行指南 | 零基础前置：npm/ts-node/tsx/tsc 与排错（通用知识） |
| [1_bun-workspaces.md](2.5_tooling/1_bun-workspaces.md) | Bun workspaces | monorepo 包拓扑 |
| [2_turborepo.md](2.5_tooling/2_turborepo.md) | Turborepo | 任务编排+缓存 |
| [3_oxlint.md](2.5_tooling/3_oxlint.md) | oxlint | Rust 写的 linter（替代 ESLint） |
| [4_ast-grep.md](2.5_tooling/4_ast-grep.md) | ast-grep | AST 结构化规则检查 |
| [5_prettier.md](2.5_tooling/5_prettier.md) | Prettier | 格式化 |
| [6_husky.md](2.5_tooling/6_husky.md) | Husky | git hooks |
| [7_playwright.md](2.5_tooling/7_playwright.md) | Playwright | e2e + 组件测试 |
| [8_storybook.md](2.5_tooling/8_storybook.md) | Storybook 10 | 组件工作台 |
| [9_http-recorder.md](2.5_tooling/9_http-recorder.md) | http-recorder | VCR 式 HTTP 磁带（自研） |
| [10_conventional-commits.md](2.5_tooling/10_conventional-commits.md) | Conventional Commits | 提交规范 |

### 6. 架构风格（`architecture/`）

| 文件 | 风格 | 一句话 |
|---|---|---|
| [1_local-first.md](2.6_architecture-styles/1_local-first.md) | 本地优先 | 数据与计算默认在本机 |
| [2_daemon-client.md](2.6_architecture-styles/2_daemon-client.md) | 守护进程+瘦客户端 | 本地版 C/S 分离 |
| [3_contract-first.md](2.6_architecture-styles/3_contract-first.md) | 契约先行 | Schema 定义一次、处处生成 |
| [4_layered-monorepo.md](2.6_architecture-styles/4_layered-monorepo.md) | 严格分层 | 依赖方向是仓库铁律 |

---

## 全局技术关系图

```
┌─ 用户 ─────────────────────────────────────────────────────────────┐
│  TUI(OpenTUI+Solid)   Web(Solid+Vite+Tailwind+Kobalte)   Electron   │
│  VS Code 扩展(esbuild)                    │                          │
└────────────┬───────────────────────────────────────────┬───────────┘
             │ HTTP 请求 / SSE 事件流（类型化 client，codegen 生成）
             ▼
┌─ 本地服务进程（Bun 运行时）──────────────────────────────────────────┐
│  Effect HttpApi server（handlers / 中间件）                          │
│      │ Effect Layer 注入                                            │
│  core 引擎：Session / Runner / Tools / Permission / Config          │
│      │ Fiber 并行工具执行；write-ahead claim 恢复                   │
│  Bus（durable 事件）──→ SessionProjector ──→ 读模型表                │
│      │                                    │ Drizzle ORM             │
│      ▼                                    ▼                         │
│  SQLite(WAL)：event/event_sequence + session_v2/…（单文件）          │
│      │                                                              │
│  @opencode/ai ──流式──→ LLM providers（~50 家）                     │
│  CodeMode（Acorn 沙箱解释器）执行 AI 生成的代码                      │
└──────────────┬──────────────────────────────────────────────────────┘
               │ OTel 埋点          会话分享等云能力
               ▼                            ▼
        Honeycomb（观测）      Cloudflare Workers / Durable Objects / D1
                                    ▲ SST 编排部署（GitHub Actions CI）
```

## 选型主线

一套 **TypeScript** + 一套响应式模型（**SolidJS**）贯穿四端；一套效应系统（**Effect**）+ 一套契约（**Effect Schema**）贯穿前后端；一个嵌入式库（**SQLite**）承载事件溯源——每一层都追求"**单一技术心智模型覆盖多个表面**"。

电梯陈述：**"Effect-native 的事件溯源 AI agent"**。
