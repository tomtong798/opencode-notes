# AgentScope 2.0 与 OpenCode 对比调研

> 对比对象：
> - **AgentScope**：`/home/donghe/github/agentscope`（分支 `main`，commit `976e2ffc`，2026-10-15，版本 `2.0.11dev`）—— 阿里通义实验室 SysML 团队开源的**多智能体开发框架**。
> - **OpenCode**：`/home/donghe/github/opencode`（分支 `dev`，commit `e25b7abf89`，2026-10-16，版本 `2.0.30`）—— 开源的 **AI 编程 agent 产品**。
>
> 本文只做定位、架构与能力面的横向对比，不评价两者优劣；结论取决于你要"造车"还是"开车"。
> [返回笔记索引](../../README.md)

---

## 目录

- [1. 一句话结论](#1-一句话结论)
- [2. 定位与目标用户](#2-定位与目标用户)
- [3. 技术栈对比](#3-技术栈对比)
- [4. 架构设计对比](#4-架构设计对比)
  - [4.1 AgentScope 分层](#41-agentscope-分层)
  - [4.2 OpenCode 分层](#42-opencode-分层)
  - [4.3 设计哲学差异](#43-设计哲学差异)
  - [4.4 基础设施同构性分析](#44-基础设施同构性分析)
- [5. 核心概念映射](#5-核心概念映射)
  - [5.1 Agent](#51-agent)
  - [5.2 Tool](#52-tool)
  - [5.3 Pipeline vs Session](#53-pipeline-vs-session)
  - [5.4 Model 层](#54-model-层)
- [6. 高级能力对比](#6-高级能力对比)
- [7. 工程实践对比](#7-工程实践对比)
- [8. 性能与扩展性](#8-性能与扩展性)
- [9. 选型建议](#9-选型建议)
- [附录 A：关键路径速查](#附录-a关键路径速查)

---

## 1. 一句话结论

> **AgentScope 是造车的工具箱，OpenCode 是一辆造好的车。**

- 你要**构建**自己的智能体产品、多 agent 系统、带记忆/RAG/语音的应用 → **AgentScope**。
- 你要**使用** AI 来写代码、读代码、改代码 → **OpenCode**。

两者不是竞品：AgentScope 是 library/framework（你写代码调用它），OpenCode 是 application/product（你直接运行它）。理论上可以用 AgentScope 构建一个"OpenCode 类"的产品，但 OpenCode 的 TUI 交互、会话持久化、Git 集成等产品化细节需要自行补齐。

---

## 2. 定位与目标用户

| 维度 | AgentScope 2.0 | OpenCode |
|------|----------------|----------|
| **产品定位** | 生产级多智能体开发框架 | 开源 AI 编程 agent |
| **形态** | Python 库 + 服务端框架 | CLI / TUI / Desktop / Web 应用 |
| **目标用户** | 构建 agent 应用的开发者/团队 | 日常编码的开发者 |
| **交付物** | 你自己的 agent 应用 | 开箱即用的编码助手 |
| **类比** | LangChain / AutoGen / 自研 agent 平台底座 | Cursor / Copilot CLI / Claude Code |
| **上手成本** | 高（需理解 Agent/Pipeline/Toolkit/Workspace 等概念） | 低（装完即用，Tab 切换 agent） |
| **可定制性** | 极高（源码级、中间件级） | 中（配置、插件、MCP、skills） |

**本质差异**：

- AgentScope 解决的是"**如何搭一个 agent 系统**"：抽象了模型、工具、消息、状态、权限、沙箱、编排。
- OpenCode 解决的是"**如何让 agent 帮我写代码**"：抽象了会话、编辑器、终端、Git、文件快照、审批流。

---

## 3. 技术栈对比

| 维度 | AgentScope | OpenCode |
|------|------------|----------|
| **语言** | Python 3.11+ | TypeScript（Bun 运行时） |
| **仓库形态** | 单包 `src/agentscope`（setuptools） | Bun workspaces + turbo monorepo（40+ 包） |
| **并发模型** | asyncio 协程 | Effect-TS（结构化并发 + 显式依赖注入） |
| **类型/校验** | Pydantic v2 + type hints | TypeScript strict + Effect Schema |
| **持久化** | 可选后端：Redis / SQLAlchemy / S3 | SQLite + Drizzle ORM（强制落盘） |
| **HTTP/API** | FastAPI + python-socketio | Hono + Effect HttpApi + SSE |
| **UI** | textual TUI / Web / Console | OpenTUI（Solid.js）TUI / Tauri Desktop / Solid Web |
| **测试** | pytest + pytest-forked | bun test + Playwright（E2E/组件） |
| **包管理** | pip / uv（可选依赖 extras 极多） | bun |
| **代码生成** | 无 | `@opencode/httpapi-codegen` 生成 client |

**有意思的对照点**：

- AgentScope 用 **Python 动态性**换生态与模型 SDK 覆盖；OpenCode 用 **Effect-TS 的编译期约束**换大型 monorepo 的可维护性。
- AgentScope 把"可选能力"做成 `pyproject.toml` 的 extras（`service`、`workspace-docker`、`rag`、`memory-mem0`…），按需安装；OpenCode 把能力拆成独立 workspace 包（`core`、`server`、`tui`、`plugin`…），靠依赖方向铁律约束。
- OpenCode 的 `dev` 分支有非常明确的架构不变量（见 `AGENTS.md` 的 Session Core 章节）：事件溯源、单步 LLM 调用、inbox 投递语义等；AgentScope 更偏"框架给原语，约束留给使用方"。

---

## 4. 架构设计对比

### 4.1 AgentScope 分层

```
┌──────────────────────────────────────────────┐
│ Application (app/)                           │  服务化 / 渠道
│  Agent Service (FastAPI + Socket.IO)         │
│  Channel: 飞书 / 钉钉 / Discord              │
│  Workspace Manager: Local/Docker/K8s/E2B/... │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Agent (agent/)                               │  智能体
│  Agent（统一 ReAct 循环）                    │
│  RealtimeAgent（WebRTC 语音）                │
│  A2AAgent（跨系统 agent 协议）               │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Pipeline (pipeline/)                         │  编排
│  GoalPipeline（executor ↔ verifier 循环）    │
│  TeamPipeline（leader 委派 member）          │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Middleware (middleware/)                     │  横切能力
│  RAG / 长期记忆(Mem0,ReMe) / 模型路由 /      │
│  TTS / Tracing(OpenTelemetry) / Budget       │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Infrastructure                               │  基础设施
│  Model（10+ provider 适配）                  │
│  Tool（Function/MCP/Skill + 工具中间件）     │
│  Message（Block 化消息）                     │
│  Workspace（沙箱）/ Permission / State       │
│  Event（细粒度流式事件）                     │
└──────────────────────────────────────────────┘
```

**关键特征**：

- **一切皆流**：`Agent.reply_stream()` / `Pipeline.reply_stream()` 返回 `AsyncGenerator[AgentEvent | Msg, None]`，UI、日志、审批都消费同一条事件流。
- **一切可插**：Agent middleware、Tool middleware、Model、Workspace、Storage 都是可替换实现。
- **编排是一等公民**：Pipeline 是显式概念，Goal/Team 两种模式内建。

### 4.2 OpenCode 分层

```
┌──────────────────────────────────────────────┐
│ Presentation                                 │  多端
│  tui（OpenTUI/Solid） / app（Web） /         │
│  desktop（Tauri） / cli                      │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Server (server/)                             │  HTTP/SSE
│  Hono routes / event-feed / PTY / auth       │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Core (core/)                                 │  运行时
│  Session（事件溯源） / Agent 执行 /          │
│  Tool registry / Permission / Model resolver │
│  Git / Filesystem / Snapshot / Worktree      │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ Infrastructure                               │
│  llm providers / MCP / SQLite(Drizzle) /     │
│  plugin / skill / PTY / ripgrep              │
└──────────────────────────────────────────────┘
```

**依赖方向铁律**（来自仓库 `AGENTS.md`）：

```
schema → protocol → server
             ↓
core ────────┘
client / sdk 只依赖 schema + protocol，永不依赖 core/server
```

**关键特征**：

- **事件溯源**：Session 的每一件事都是持久化事件（`session.inbox.enqueued`、`session.inbox.delivered`…），状态由有序事件折叠重建，投影（projection）提供读模型。
- **单步执行**：每个 step 就是一次显式 `llm.stream` 调用，从重新加载的历史继续；不允许内存里的工具循环。这是为了崩溃恢复与可重放。
- **位置感知（Location-scoped）**：`SessionRunner`、模型解析、工具注册、权限、文件系统都以 Location/Workspace 为作用域。
- **客户端即协议**：`HttpApi` 定义端点，client 由代码生成，多端（TUI/Desktop/Web/SDK）共享同一协议。

### 4.3 设计哲学差异

| 关注点 | AgentScope | OpenCode |
|--------|------------|----------|
| **核心抽象** | Agent + Message + Tool + Pipeline | Session + Step + Event + Tool |
| **状态观** | 内存态为主，可选持久化 | 事件溯源，持久化是前提 |
| **执行观** | ReAct 循环在 Agent 内 | 每步一次 LLM 调用，恢复友好 |
| **扩展方式** | 中间件（洋葱模型） | 插件 + 配置 + MCP + skills |
| **约束来源** | 使用方自觉 | 类型系统 + 架构不变量 + lint 规则 |
| **UI 观** | 框架附带 console/TUI/Web 示例 | UI 是产品核心，多端一等公民 |
| **部署观** | 服务化（FastAPI 服务 + 渠道） | 本地优先（后台 service + 客户端连接） |

### 4.4 基础设施同构性分析

> 直觉先行：初看两个仓库会觉得"基础设施几乎一样，只有中台模块不同"。这个直觉**基本正确**，且能精确到"哪些层同构、哪些层分叉"。本节用代码证据展开。

#### 消息内容块：一一对应

AgentScope `src/agentscope/message/_block.py` 与 OpenCode `packages/schema/src/session-message.ts` 的对照：

| 概念 | AgentScope | OpenCode |
|------|-----------|----------|
| 文本 | `TextBlock(type:"text")` | `AssistantText(type:"text")` |
| 思考 | `ThinkingBlock(type:"thinking")` | `AssistantReasoning(type:"reasoning")` |
| 工具调用+结果 | `ToolCallBlock` + `ToolResultBlock` 两个块 | `AssistantTool` 一个块（state 内嵌） |
| 多模态 | `DataBlock` / `Base64Source` / `URLSource` | `FilePart` / files 字段 |
| 消息角色 | `user/assistant/system` | `user/synthetic/system` + tool/shell/skill 派生类型 |

#### 工具生命周期状态机：同一问题，两套侧重

```
AgentScope ToolCallState（权限为中心，HITL 语义完整）:
  pending → asking → allowed → submitted → finished
  （含 DENY / 校验失败直接 finished 的旁路）
  ToolResultState: running / success / error / interrupted / denied

OpenCode ToolState（执行为中心）:
  streaming → running → completed / error
```

AgentScope 把"审批"做进了消息状态机（`asking/allowed/denied`），OpenCode 把审批放在独立的 permission 系统里、状态机只管执行——但骨架是同一个：**生成 → 审批 → 执行 → 完成**。

#### 其余基础设施对照

| 原语 | AgentScope | OpenCode |
|------|-----------|----------|
| 结构化流式事件 | `AgentEvent` 层级（`TextBlockStart/Delta/End` 等 30 个） | session events / SSE feed |
| Provider 归一化 | `ChatModelBase` + 10 家适配 | model resolver + models.dev |
| 权限引擎 | `PermissionEngine` + `PermissionRule` | `permission/` + saved rules |
| MCP | `mcp/` | `core/src/mcp/` |
| Skills | workspace 加载 skill | `core/src/skill/` |
| 上下文压缩 | CompressContext 工具 + offloader | `session/compaction.ts` |
| 消息持久化 | 消息内嵌状态 + 可选存储 | SQLite + projection（强制） |

#### 为什么同构：行业收敛，而非互相抄袭

两边被同一组上游约束塑造：

1. **Anthropic content blocks / OpenAI parts** 定义了消息块的形状；
2. **MCP** 统一了工具协议；
3. **流式 UI** 要求 Start/Delta/End 三段式事件；
4. **工具调用的客观生命周期**就只有那几种状态。

任何 2025 年后认真做的 agent runtime，基础设施层都会长成这个样子。**这层已经没有差异化空间——它是"操作系统内核"，不是产品。**

#### 真正的分叉：不在基础设施，在两处

**① 中台模块——按使命分叉**：

- AgentScope 中台 = **水平能力平台**：Pipeline 编排、Middleware（RAG/记忆/模型路由/TTS）、Workspace 沙箱、Channel、Realtime、A2A、SOP——服务于"造各种 agent 产品"。
- OpenCode 中台 = **垂直产品运行时**：事件溯源 Session、Git/worktree/snapshot、PTY、LSP、多端 UI——服务于"把编码这一个场景做到极致"。

**② 状态与执行模型（更深层，表面看不出）**：

| | AgentScope | OpenCode |
|---|---|---|
| 状态观 | 内存态优先，持久化可选 | 事件溯源，落盘是前提 |
| 执行观 | ReAct 循环在 Agent 内存里跑多轮 | 一步 = 一次 `llm.stream`，每步从重载的历史继续，禁止内存工具循环 |
| 动机 | 框架灵活，可靠性交由使用方 | 崩溃恢复 / 可重放是硬约束 |

同构的内核之上，跑着完全不同的可靠性哲学。AgentScope 的"内存 ReAct 循环"让编排逻辑任意组合；OpenCode 的"单步执行 + 事件溯源"让一次编码会话在任何时刻被杀掉都能恢复。

> 延伸阅读：由本节引出的"同质化之下 agent 的价值在哪里"的系统分析，见 [2_agent-value-framework.md](2_agent-value-framework.md)。

---

## 5. 核心概念映射

### 5.1 Agent

| 维度 | AgentScope | OpenCode |
|------|------------|----------|
| **定义方式** | `Agent(name, system_prompt, model, toolkit, ...)` 显式构造 | 内置 `build` / `plan` agent；配置 + 运行时 Tab 切换 |
| **推理循环** | ReAct：Reasoning → Acting → Observation | 单步 `llm.stream`，外层 runner 决定是否继续 |
| **子 agent** | Pipeline / 动态创建 | `@general` 子 agent、agent 配置 |
| **上下文管理** | 自动压缩（CompressContext 工具）+ offloader | 会话历史 + 文件快照 + 压缩策略 |
| **多模态** | 图片/音频/视频（DataBlock） | 图片为主（编码场景） |
| **实时语音** | ✅ RealtimeAgent（WebRTC） | ❌ |
| **中断/审批** | 事件 + PermissionEngine | 事件 + permission 策略 |

**AgentScope 构造示例**：

```python
agent = Agent(
    name="Friday",
    system_prompt="You are a helpful assistant.",
    model=DashScopeChatModel(
        credential=DashScopeCredential(api_key=...),
        model="qwen3.7-max",
    ),
    toolkit=Toolkit(tools=await workspace.list_tools()),
    middlewares=[AgenticMemoryMiddleware(...)],
    offloader=workspace,
)
async for event in agent.reply_stream(user_msg):
    ...
```

**OpenCode 使用方式**：

```bash
opencode            # 启动 TUI
# Tab 切换 build / plan
# @general 调用通用子 agent
```

### 5.2 Tool

| 维度 | AgentScope | OpenCode |
|------|------------|----------|
| **工具形态** | `FunctionTool` / `MCPTool` / `Skill` | 内置工具 + MCP + skill |
| **注册中心** | `Toolkit` | Tool registry |
| **工具中间件** | ✅ `ToolMiddlewareBase`（onion） | ❌ 无对应概念 |
| **权限** | `PermissionEngine` + `PermissionRule`（工具名/路径匹配） | permission 策略 + 审批 UI |
| **沙箱** | ✅ Workspace 抽象（Docker/K8s/E2B/Daytona/OpenSandbox） | ❌ 本地进程/PTY 执行 |
| **流式工具结果** | ✅ `ToolChunk` 异步生成器 | ✅ 流式输出 |
| **内置工具** | 文件系统、代码执行、搜索等（来自 Workspace） | read/edit/bash/grep/glob/git 等 |

**工具中间件（AgentScope 独有）**：

```python
class LoggingMiddleware(ToolMiddlewareBase):
    async def on_tool_call(self, tool, input_kwargs, next_handler):
        print(f"calling {tool.name}")
        async for chunk in next_handler(**input_kwargs):
            yield chunk
```

OpenCode 侧更接近"在工具实现内部处理" + "插件钩子"，没有统一的工具级洋葱中间件抽象。

### 5.3 Pipeline vs Session

| 维度 | AgentScope Pipeline | OpenCode Session |
|------|---------------------|------------------|
| **概念** | 多智能体编排模式 | 用户与 agent 的对话会话 |
| **内建模式** | GoalPipeline、TeamPipeline | 单一 Session，靠 agent/子 agent 分工 |
| **状态** | 内存态 | 事件溯源 + SQLite |
| **崩溃恢复** | ❌ 无内建 | ✅ 事件回放 + execution claim 恢复 |
| **并发** | 多 agent/pipeline 并行 | 多 Session 并发（同 Session 串行协调） |
| **A2A** | ✅ A2AAgent | ❌ |

结论：**Pipeline 是"编排原语"，Session 是"产品运行时"**，两者不在同一抽象层，无法直接等价比较。

### 5.4 Model 层

| 维度 | AgentScope | OpenCode |
|------|------------|----------|
| **抽象** | `ChatModelBase`（统一 `__call__`） | model resolver + provider 适配 |
| **Provider** | DashScope/OpenAI/Anthropic/Gemini/Ollama/xAI/MiniMax/Moonshot/DeepSeek/Volcengine | 通过 models.dev 覆盖主流 provider |
| **结构化输出** | ✅ JSON Schema（含 fallback 策略） | ✅ JSON Schema |
| **Thinking/推理** | ✅ 显式开关 + 各 provider 适配 | ✅ 支持 |
| **重试** | ✅ 可重试异常白名单 + 退避 | ✅ |
| **动态路由** | ✅ `ModelRouterMiddleware` | ❌ 无内建按回复路由 |
| **模型清单** | 每个 provider 自带 YAML model card | models.dev 元数据 |

---

## 6. 高级能力对比

| 能力 | AgentScope | OpenCode | 说明 |
|------|:---:|:---:|------|
| 多智能体编排 | ✅ | △ | AgentScope 有 Goal/Team Pipeline；OpenCode 有子 agent 但无显式编排原语 |
| 长期记忆 | ✅ | ❌ | AgentScope：AgenticMemory / Mem0 / ReMe |
| RAG / 向量库 | ✅ | ❌ | Qdrant / Milvus / MongoDB / Elasticsearch |
| 实时语音（WebRTC） | ✅ | ❌ | RealtimeAgent + 多 provider realtime API |
| A2A 协议 | ✅ | ❌ | 跨系统 agent 通信 |
| SOP（标准作业流程） | ✅ | ❌ | 多步骤流程化任务 |
| MCP | ✅ | ✅ | 双方都支持 |
| Skills | ✅ | ✅ | AgentScope 从 Workspace 加载；OpenCode 有 skills 体系 |
| 沙箱隔离 | ✅ | ❌ | Docker/K8s/E2B/Daytona/OpenSandbox vs 本地 PTY |
| 服务化部署 | ✅ | ✅ | FastAPI 服务 vs 后台 service |
| 渠道集成 | ✅ | ❌ | 飞书/钉钉/Discord vs 无 |
| 终端 TUI | ✅ | ✅ | textual vs OpenTUI（Solid） |
| 桌面应用 | ❌ | ✅ | Tauri Desktop |
| Web UI | ✅ | ✅ | 示例级 vs 产品级 |
| Git 深度集成 | ❌ | ✅ | worktree / snapshot / 修改追踪 |
| 会话持久化/恢复 | △ | ✅ | AgentScope 可选；OpenCode 事件溯源强制 |
| 插件体系 | △（中间件） | ✅（plugin） | 扩展粒度不同 |
| 遥测/Tracing | ✅ | ✅ | OpenTelemetry 双方都有 |
| 权限审批 | ✅ | ✅ | 规则引擎 vs 交互式审批 |

图例：✅ 一等支持；△ 有限/需自行实现；❌ 无内建。

---

## 7. 工程实践对比

| 维度 | AgentScope | OpenCode |
|------|------------|----------|
| **类型安全** | Pydantic + hints（运行时校验强） | TS strict + Effect Schema（编译期强） |
| **错误处理** | 自定义异常层级 + 结构化错误 | Effect 错误通道 + 类型化错误 |
| **测试** | pytest，含 fork 隔离 | bun test，明确反对 mock，E2E/组件测试 |
| **代码风格约束** | pre-commit（ruff/pylint 等） | oxlint + ast-grep 规则（含 Effect 模式检查） |
| **文档** | 中英双语文档站 + 大量 examples | 20+ 语言 README + docs 站 + specs/ |
| **架构文档** | 分散在 docs/ | `AGENTS.md` 记录强不变量 + `specs/` |
| **生成代码** | 无 | HttpApi → client 代码生成（禁止手改） |
| **发布** | PyPI 单包 + extras | npm 多包 + 桌面安装包 + brew/scoop/... |

**值得注意的差异**：

- OpenCode 把"架构不变量"写进 `AGENTS.md` 并配套 ast-grep 规则，属于**用工具链强制约束**；AgentScope 更依赖评审与文档。
- AgentScope 的可选依赖矩阵非常庞大（`full` extra 几乎覆盖所有后端），好处是轻量起步，代价是组合爆炸的测试面。
- OpenCode monorepo 的依赖方向由 lint/CI 守护，AgentScope 单包内部靠目录约定。

---

## 8. 性能与扩展性

| 维度 | AgentScope | OpenCode |
|------|------------|----------|
| **冷启动** | 中等（Python import + 依赖） | 快（Bun 单文件/编译产物） |
| **运行时开销** | 中（Python + provider SDK） | 低-中（Bun + Effect 运行时） |
| **并发模型** | asyncio，单进程高并发 I/O | Effect 结构化并发，多 Session 并行 |
| **水平扩展** | ✅ 服务化 + 多种存储后端 | △ 本地优先；多实例需自行解决 |
| **沙箱扩展** | ✅ 多种后端可插 | ❌ 依赖 OS 进程隔离 |
| **插件生态** | 中等（中间件/工具） | 中等（plugin/MCP/skill） |

---

## 9. 选型建议

### 选 AgentScope，如果你要：

- 构建**自有品牌的 agent 产品**（而非直接用现成工具）
- 需要**多 agent 协作**（leader/member、executor/verifier）
- 需要**长期记忆、RAG、知识库**
- 需要**语音实时交互**
- 需要接入**飞书/钉钉/Discord** 等企业渠道
- 需要**沙箱化执行**（多租户、不可信代码）
- 需要**SOP 流程化**的复杂任务
- 团队以 **Python** 为主

### 选 OpenCode，如果你要：

- 一个**开箱即用的 AI 编码助手**
- 终端优先的工作流（TUI），或桌面应用
- 深度 **Git / worktree / 文件快照** 集成
- 多端一致体验（TUI / Desktop / Web / SDK）
- 稳定的**会话持久化与恢复**
- 轻量、快速迭代的**本地工具**
- 不打算自己造 agent 框架

### 组合思路

两者并不互斥：

- 用 **AgentScope** 做后端 agent 服务（多 agent、记忆、RAG、渠道），把 **OpenCode** 当作开发者侧的编码入口。
- 或者反过来：借鉴 OpenCode 的**事件溯源 + 单步执行 + 审批流**设计，增强 AgentScope 服务的可恢复性与可观测性；借鉴 AgentScope 的**中间件 + Pipeline + Workspace** 抽象，扩展 OpenCode 的多 agent 与沙箱能力。

---

## 附录 A：关键路径速查

### AgentScope

| 关注点 | 路径 |
|--------|------|
| 包入口 | `src/agentscope/__init__.py` |
| Agent 核心 | `src/agentscope/agent/_agent.py`（约 3900 行） |
| Pipeline | `src/agentscope/pipeline/_goal_pipeline.py`、`_team_pipeline.py` |
| Pipeline 协议 | `src/agentscope/pipeline/_base.py` |
| Model 基类 | `src/agentscope/model/_base.py` |
| Tool 基类/中间件 | `src/agentscope/tool/_base.py` |
| Toolkit | `src/agentscope/tool/_toolkit.py` |
| 消息/内容块 | `src/agentscope/message/_base.py`、`_block.py` |
| 中间件 | `src/agentscope/middleware/` |
| 工作区 | `src/agentscope/workspace/` |
| 事件 | `src/agentscope/event/` |
| 权限 | `src/agentscope/permission/` |
| 服务化 | `src/agentscope/app/` |
| 示例 | `examples/console/main.py`、`examples/pipeline/goal/goal_pipeline.py` |
| 依赖矩阵 | `pyproject.toml` |

### OpenCode

| 关注点 | 路径 |
|--------|------|
| 架构总览 | `1_architecture/1_overview.md`（本笔记） |
| 包版图 | `1_architecture/2_package-map.md` |
| 架构不变量 | `AGENTS.md`（Session Core 等章节） |
| 核心运行时 | `packages/core/src/session.ts`、`packages/core/src/session/` |
| 事件总线 | `packages/core/src/bus.ts`、`event/` |
| 工具 | `packages/core/src/tool/`、`tool.ts` |
| 权限 | `packages/core/src/permission/`、`permission.ts` |
| 模型解析 | `packages/core/src/model.ts`、`model-resolver.ts` |
| Server | `packages/server/src/` |
| 协议/客户端 | `packages/protocol/`、`packages/client/` |
| TUI | `packages/tui/src/` |
| 持久化 | `packages/core/src/database/`、`drizzle.config.ts` |
| 插件 | `packages/plugin/`、`packages/plugins/` |
| MCP | `packages/core/src/mcp/` |
| 工作区/位置 | `packages/core/src/location*.ts`、`workspace*` |

---

> 版本与提交信息见文首。若后续代码有更新，建议以各自仓库 `main`/`dev` 的最新实现为准，本文的架构结论通常比具体行号更稳定。
