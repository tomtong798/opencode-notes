# 将 AgentScope 中台能力引入 OpenCode：可行性分析与移植方案

> 承接前三篇：
> [1_agentscope-vs-opencode.md](1_agentscope-vs-opencode.md) —— 内核同构、中台分叉；
> [2_agent-value-framework.md](2_agent-value-framework.md) —— 价值向运行时与产品层上移；
> [3_agent-engine-convergence.md](3_agent-engine-convergence.md) —— 引擎收敛预测。
> 本文回答一个工程落地问题：**能否把 AgentScope 有而 OpenCode 没有的中台服务搬进 OpenCode？以"内置插件"形态添加是否可行？**
> [返回笔记索引](../../README.md)

---

## 目录

- [1. 结论先行](#1-结论先行)
- [2. OpenCode 的扩展面（实证）](#2-opencode-的扩展面实证)
- [3. 三区可行性划分](#3-三区可行性划分)
- [4. 绿区详解：外部插件路径](#4-绿区详解外部插件路径)
- [5. 内置插件路径：语义移植方案](#5-内置插件路径语义移植方案)
- [6. 三个非技术的坑](#6-三个非技术的坑)
- [7. 落地建议](#7-落地建议)

---

## 1. 结论先行

| 问题 | 答案 |
|---|---|
| 把中台服务搬进 OpenCode，可行吗？ | **分模块**：绿区（RAG/记忆/路由等）完全可行；黄区（编排/沙箱）侵入；红区（语音/多租户）不匹配 |
| 以"内置插件"形态添加，可行吗？ | **机制上完全对路，但本质是"语义移植"而非"添加"**——需用 TS 重写，且必须做哲学翻译 |

两个关键实证（详见 §2）：

1. **OpenCode 的内置工具本身就是插件**（`websearch.ts` 导出 `Plugin = { id: "opencode.tool.websearch", ... }`，与外部插件共用同一 API）——"内置"与"插件"在本仓库不是对立概念；
2. 挂载点齐备：`session.hook("context" / "compaction" / "generate" / "retry")` + `tool.transform` + `model.transform`——恰好覆盖 AgentScope 中间件的能力面。

License 无障碍：OpenCode MIT、AgentScope Apache-2.0，移植只需保留 attribution。

## 2. OpenCode 的扩展面（实证）

### 插件 API 的两类钩子

来自 `packages/plugin/src/README.md`：

```ts
// Transform hooks —— 作用于有状态域（声明式数据）
ctx.agent.transform     ctx.command.transform   ctx.integration.transform
ctx.mcp.transform       ctx.model.transform     ctx.provider.transform
ctx.reference.transform ctx.skill.transform     ctx.tool.transform
ctx.vcs.transform       ctx.websearch.transform

// Runtime hooks —— 拦截实时操作
await ctx.session.hook("context", async (event) => {   // 每次 provider dispatch 前
  event.tools.read.description = "..."
  delete event.tools.write
})
await ctx.session.hook("retry", (event) => { ... })
await ctx.aisdk.hook("sdk" / "language", (event) => { ... })
```

### 内置工具即插件（关键发现）

`packages/core/src/tool/plugin/` 下 14 个内置工具（read/edit/write/shell/grep/glob/patch/question/skill/subagent/webfetch/websearch/mcp-resource/opencode）**全部**以 `export const Plugin = { id: "opencode.tool.xxx", ... }` 形式实现，例如 `websearch.ts`：

```ts
export const Plugin = {
  id: "opencode.tool.websearch",
  effect: Effect.fn("WebSearchTool.Plugin")(function* (ctx: Context) {
    yield* ctx.tool.transform((editor) => { /* 注册工具 */ })
  }),
}
```

`opencode.ts` 工具插件实测挂了三个 session hook：

```ts
yield* ctx.session.hook("context", hook)
yield* ctx.session.hook("compaction", hook)
yield* ctx.session.hook("generate", hook)
```

### 落位先例：figma 模式

`packages/plugins/figma`（`@opencode/plugin-figma`）——in-repo 独立包、随仓库发布、可独立启用、自带 `skills/` 目录。这就是移植目标的标准落位。

### 洋葱 → hook 的映射关系

AgentScope 中间件（`on_reply_start` / `on_tool_call` / post-hooks 洋葱模型）可无损映射到 hook 体系——**hook 就是展开的洋葱层**：

| AgentScope 洋葱层 | OpenCode hook |
|---|---|
| `on_reply_start` | `session.hook("context")` |
| `on_tool_call` | 工具自身的 execute + `session.hook` |
| post-hooks / 修改输出 | `session.hook("generate")` / SSE 消费 |
| 上下文压缩干预 | `session.hook("compaction")` |
| 失败重试干预 | `session.hook("retry")` |

## 3. 三区可行性划分

| 区域 | 模块 | 可行性 |
|---|---|---|
| 🟢 绿区（插件层即可） | RAG、长期记忆（Agentic/Mem0/ReMe）、模型路由、Budget、A2A、TTS | ✅ 完全可行 |
| 🟡 黄区（需改 core 或 sdk 组合） | Pipeline 编排、Channel、SOP、Workspace 沙箱 | ⚠️ 可行但侵入 |
| 🔴 红区（架构性不匹配） | Realtime 语音、多租户 agent 舰队 | ❌ 不建议 |

### 黄区的两条路线

- **路线 A（fork/改 core）**：OpenCode 已有 subagent 机制（`agent.mode: "subagent"`、`session/subagent-job.ts`），但没有编排原语。TeamPipeline 可用 subagent + leader agent 配置模拟；GoalPipeline 的 executor↔verifier 循环与"单步执行、无内存工具循环"不变量**直接冲突**。Workspace 沙箱需动 `tool/plugin/shell.ts` 与 filesystem 工具层——没有 `WorkspaceBase` 那样的干净接口（印证第 3 篇：CRI 等价物不存在）。
- **路线 B（sdk 组合，非侵入）**：`packages/sdk` = client + core + server 的嵌入式 host。Channel 集成的正确路径是**用 sdk 组一个自己的服务**，在它上面盖飞书/钉钉 gateway（借鉴 AgentScope `channel/_gateway.py` 模式），完全不改 OpenCode 代码。

### 红区的判定理由

- **Realtime 语音**：session 是"文本、步进、可回放"模型；realtime 是"音频流、全双工、低延迟"模型。状态机、事件流、错误恢复语义全不相同。AgentScope 为此做了独立的 `RealtimeAgent` + `realtime/` 树——这是另一套 runtime，不是中台模块。
- **多租户舰队**：OpenCode 的 Location 是单工作区语义、本地 service 进程模型。它的魂是单会话 controller，不是多租户 scheduler。

## 4. 绿区详解：外部插件路径

如果只是"让现有 agent 更聪明/更懂业务"，外部插件即可，不动 core：

| AgentScope 模块 | OpenCode 实现路径 |
|---|---|
| `RAGMiddleware` | `session.hook("context")` 注入检索结果，或 `tool.transform` 注册 `search_knowledge_base` |
| `AgenticMemoryMiddleware` / Mem0 / ReMe | 双路：上下文注入 + `memory_save`/`memory_search` 工具 |
| `ModelRouterMiddleware` | `model.transform` + `session.hook` 按任务选模型 |
| `BudgetMiddleware` | `session.hook` + `session/stats.ts`（已有 token 统计） |
| `A2AAgent` | `tool.transform` 把远程 A2A agent 包装成工具调用（本就是 A2A 语义） |
| `TTSMiddleware` | 挂 SSE 事件流旁路处理，不碰 core |

## 5. 内置插件路径：语义移植方案

### 先破题：三种"添加"形态

| 形态 | 做法 | 判定 |
|---|---|---|
| **A. 语义移植** | 用 TS 重写中间件逻辑，作为内置插件 | ✅ 正路 |
| B. 进程桥接 | TS 插件 spawn Python 子进程调 AgentScope | ❌ 伪内置 |
| C. 仅借设计 | 只参考架构不搬实现 | ✅（A 的简化版） |

**B 被分发形态否决**：OpenCode 的交付文化是 `curl install` 单命令、Bun 单运行时（连原生依赖都要 `fix-node-pty` 特殊处理）。要求用户预装 Python 的"内置插件"在产品上不成立。

### 逐个中间件的移植蓝图

| AgentScope 中间件 | 内置插件形态 | 挂载点 | 难度 | 备注 |
|---|---|---|---|---|
| `AgenticMemoryMiddleware` | `memory_save`/`memory_search` 工具组 | `tool.transform` + `session.hook("context")` | 中 | 依赖换 TS 向量库/SQLite |
| `RAGMiddleware` | `search_knowledge_base` 工具 + 检索提示 | 同上 | 中 | qdrant/milvus 均有 JS client |
| Mem0 / ReMe | 薄 HTTP 客户端工具 | `tool.transform` | 低 | 外部服务，纯 API 调用 |
| `BudgetMiddleware` | 用量护栏 | `session.hook("retry")` + usage stats | 低 | |
| `ModelRouterMiddleware` | 模型策略过滤器 | `model.transform` | 中 | README 有现成示例 |
| `TTSMiddleware` | 语音输出侧车 | SSE 事件流消费 | 低 | 不碰 core |
| `TracingMiddleware` | ~~移植~~ | — | — | **不用做**：仓库已依赖 `@effect/opentelemetry` |

### 推荐目录结构（figma 模式）

```
packages/plugins/agentscope-memory/    ← 第一个：长期记忆（价值最高）
packages/plugins/agentscope-rag/       ← 第二个：知识库检索
packages/plugins/agentscope-budget/    ← 第三个：用量护栏（最便宜）
```

每个包自带 `skills/` 目录（figma 先例：`skills/` + `skills-source.json`）——这正是"哲学翻译"的载体（见下节）。

## 6. 三个非技术的坑

### 坑一：哲学翻译（最重要）

- AgentScope 中间件是 **developer-oriented**：开发者强制注入上下文，模型无感知；
- OpenCode 惯例是 **agent-oriented**：暴露工具，让模型自己决定何时用（内置工具描述到 AGENTS.md 一以贯之）。

同一个"记忆"能力的两种写法：

- AgentScope：每轮 reply 前自动检索记忆塞进 context；
- OpenCode 正统：给模型 `memory_search` 工具 + 在 skill/instructions 里教它何时用。

直译前者违背 OpenCode 设计哲学，且效果更差（无差别注入浪费上下文）。**移植必须做语义翻译，不能机械翻译。**

### 坑二：上游不变量的硬约束

AGENTS.md 明文：*"Each step is one explicit `llm.stream` call... Do not add an in-memory tool loop"*。这直接判了编排类中间件的死刑——GoalPipeline 式循环无法以"内置插件"形态进入。**不是技术不行，是架构法不许。**

### 坑三：上游节奏

dev 分支高频演进（架构不变量仍在演化期）。in-repo 插件跟着 monorepo 走，rebase 成本可控但持续存在——比 fork core 便宜一个量级，这正是此路线的价值。**插件能解决的，绝不碰 core。**

## 7. 落地建议

```
你想加的能力是什么性质？
│
├─ "让现有 agent 更聪明/更懂业务"
│   → 🟢 写外部插件（RAG/记忆/路由），2-4 周量级，随上游升级
│
├─ "改变 agent 的协作/编排方式"
│   → 🟡 先试 agent 配置 + subagent + 工具组合（不动 core）
│      不够再考虑 fork runner——做好长期维护 fork 的准备
│
└─ "把 opencode 当底座做自己的产品"
    → 🟡/🔴 用 sdk 组合，把它当 library 而非 fork 目标
       （此时你其实在造 AgentScope 想造的东西——见第 3 篇）
```

想回馈上游：记忆/RAG 这类通用能力有被接受的可能；编排类别投（哲学冲突）。

---

> 一句话收尾：**机制上这是"把 AgentScope 的中台语义翻译进 OpenCode 的插件原语"，工程上是 3-5 个 TS 插件包的移植工作，真正的难点不在代码，而在两套哲学的语义转换。**
