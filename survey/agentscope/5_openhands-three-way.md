# 三方对照：AgentScope × OpenCode × OpenHands SDK

> 新增对比对象：
> - **OpenHands Software Agent SDK**：`github.com/OpenHands/software-agent-sdk`（分支 `main`，commit `3e8e17b`，2026-10-19，SDK 版本 `1.55.1`，MIT）—— OpenHands V1 的 Python SDK + Agent Server + Tools + Workspace 分包仓库。前身为 OpenDevin（2024-03 立项 → 更名 OpenHands → 2025 V1 SDK 化重构，见 arxiv 2511.03690）。
> - AgentScope / OpenCode 版本信息见 [第 1 篇](1_agentscope-vs-opencode.md) 文首。
>
> 本篇的论证目标：给"内核同构"补上**第三个数据点**，使第 1 篇的双点对比升级为三角验证。
> [返回系列索引](README.md) | [返回笔记索引](../../README.md)

---

## 目录

- [1. 结论先行](#1-结论先行)
- [2. 模块级对照表](#2-模块级对照表)
- [3. API 形态对照](#3-api-形态对照)
- [4. 为什么像：四个力叠加](#4-为什么像四个力叠加)
- [5. 差异仍在"魂"的层面](#5-差异仍在魂的层面)
- [6. 对调研链的意义：证据三角](#6-对调研链的意义证据三角)

---

## 1. 结论先行

> **AgentScope 和 OpenHands SDK 不是"像"，是同一个物种。**
> 同语言（Python 3.11+）+ 同代最佳实践（Pydantic v2 + asyncio）+ 同领域（coding-agent 中心），使两家的收敛**从概念层深入到代码形态层**——抽象基类的方法清单几乎可以逐行对齐。

一句话版本对照：

| 仓库 | 一句话定位 |
|---|---|
| AgentScope | 通用 agent 框架（阿里，平台化广度优先） |
| OpenCode | AI 编程 agent 产品（TS/Bun，垂直体验优先） |
| OpenHands SDK | coding agent 的 Python SDK + Server（学术出身，可组合性优先） |

## 2. 模块级对照表

AgentScope `src/agentscope/` vs OpenHands SDK `openhands-sdk/openhands/sdk/`（+ 同仓库姊妹包），全部实测核对：

| 维度 | AgentScope | OpenHands SDK |
|---|---|---|
| 核心 | `agent/_agent.py`（`Agent` 类） | `agent/base.py`（`AgentBase` 类，DiscriminatedUnionMixin） |
| 会话 | Agent 内部 state + `state/` | **`conversation/`（一等公民）**：event_store、fifo_lock、stuck_detector、resume、resource_lock_manager |
| 消息块 | `TextBlock/DataBlock/ToolCallBlock`，role: user/assistant/system | `TextContent/ImageContent/MessageToolCall`，role: user/assistant/system/**tool** |
| 事件 | `event/`（30 个 AgentEvent + 流式 delta） | `event/`（ActionEvent/ObservationEvent/StreamingDeltaEvent/condenser） |
| LLM | `model/`（10 家手写适配） | `llm/`（**litellm 一层代理**） |
| 沙箱 | `workspace/`（Local/Docker/K8s/E2B/Daytona/OpenSandbox/bubblewrap） | `openhands-workspace/`（Docker/AgentSandbox-K8s/Cloud/Apptainer/remote_api） |
| 扩展 | `middleware/`（洋葱） | `hooks/` + `plugin/` |
| 安全 | `permission/`（PermissionEngine + Rule） | `security/`（SecurityAnalyzerBase + ConfirmationPolicy） |
| MCP | `mcp/` | `mcp/` |
| 技能 | `skill/`（workspace 加载） | `skills/` + `marketplace/` |
| 子代理 | `pipeline/`（Goal/Team） | `subagent/` + tools 里的 `delegate`/`child_conversation` |
| 服务化 | `app/`（FastAPI Agent Service + Channel） | `openhands-agent-server/`（REST/WebSocket） |
| 工具集 | workspace `list_tools()`：filesystem/bash/grep/... | `openhands-tools/`：terminal/file_editor/glob/grep/task_tracker/ask_oracle/... |
| 观测 | OpenTelemetry 中间件 | `observability/`（Laminar span 体系） |

抽象基类的形态对照（同语言时收敛到代码级的证据）：

```
AgentScope WorkspaceBase            OpenHands BaseWorkspace/RemoteWorkspace
  async def initialize()              （AgentSandboxWorkspace：connect/health/logs）
  async def close()                   def cleanup()
  async def reset()                   def pause() / def resume()
  __aenter__ / __aexit__              __enter__ / __exit__ / __del__
```

## 3. API 形态对照

OpenHands README 首页示例：

```python
agent = Agent(llm=llm, tools=[Tool(name=TerminalTool.name), ...])
conversation = Conversation(agent=agent, workspace=cwd)
conversation.send_message("..."); conversation.run()
```

AgentScope console 示例：

```python
agent = Agent(name="Friday", system_prompt=..., model=..., toolkit=Toolkit(...), ...)
async for event in agent.reply_stream(msg): ...
```

差异一眼可见（这预演了 §5）：OpenHands 以 **Conversation** 为中心（先建会话再跑），AgentScope 以 **Agent** 为中心（agent 自己流式吐事件）。

## 4. 为什么像：四个力叠加

1. **类型收敛**（第 1 篇 §4.4 已论证）：Anthropic blocks / MCP / 流式三段事件塑造所有 runtime；
2. **同语言 + 同代最佳实践**：Python 3.11+、Pydantic v2 BaseModel、asyncio、ABC+abstractmethod、async context manager——语言相同时，收敛不只是概念层，**连代码形态都收敛**；
3. **同领域聚焦**：OpenHands README 第一句 "building agents that work with code"；AgentScope 虽是通用框架，但 workspace 工具链（filesystem/bash/ripgrep/tree_sitter 依赖）同样是 coding-agent 中心设计；
4. **平行演化的时间线**：AgentScope 论文 2024-02（v0.0.1 于 2024-02-26），2.0 重构发布于 2026-05-25；OpenDevin 2024-03 立项 → 2025 V1 SDK 化（software-agent-sdk 建库 2025-08-23，仅比 AgentScope v1.0.0 晚 8 天）。**两家在互不知情的情况下，各花约两年从单体应用重构成"SDK + Server + Tools 分包"的同构形态**——最有力地证明是环境在选择形状，不是抄袭。（版本日期来自 GitHub releases API，经 WorkBuddy 交叉验证，见[第 6 篇](6_workbuddy-cross-validation.md)。）

## 5. 差异仍在"魂"的层面

| | AgentScope | OpenHands SDK |
|---|---|---|
| **中心抽象** | Agent（`reply_stream` 驱动） | **Conversation**（事件存储、持久化、恢复、stuck 检测都在会话级） |
| **编排** | Pipeline 原语（Goal/Team） | 无显式编排——subagent/delegate 工具 + `critic_mixin` 组合 |
| **模型接入** | 手写 10 家 provider | litellm 一层代理（控制力 vs 覆盖面的取舍） |
| **广度** | Channel/Realtime/SOP/RAG 中间件 | 砍广度换深度（GitHub workflows、resolution、marketplace） |
| **异步观** | 全异步（async 优先） | 同步 `conversation.run()` 为主、内部异步（SDK 易用性取舍） |
| **持久化** | 可选 | Conversation 事件存储是前提 |

注意一个关键观察：**OpenHands 的 Conversation 与 OpenCode 的 Session 是同一种魂**（事件溯源、可恢复），而 AgentScope 与它们的差异依旧是第 1 篇指出的分叉——内存态 vs 落盘态。三个仓库里，"会话可恢复性"这个维度上 2:1，行业重心可见一斑。

## 6. 对调研链的意义：证据三角

本篇把"内核同构"的论证从双点升级为三角：

| 对照 | 语言 | 收敛深度 | 证明了什么 |
|---|---|---|---|
| AgentScope vs OpenCode（第 1 篇） | 跨语言（Py/TS） | 概念层同构 | 排除"同语言巧合"，协议塑造力 |
| AgentScope vs OpenHands（本篇） | 同语言（Py） | **代码形态层同构** | 连抽象基类方法清单都对齐 |
| OpenCode vs OpenHands（交叉验证） | 跨语言 | Session ↔ Conversation 同魂 | 可恢复性是行业收敛方向 |

对第 3 篇（引擎收敛预测）的修正：agent SDK 作为"类型"（genre）已经定型，新入场者没有塑造形状的机会——**引擎化窗口正在打开，且置信度应上调**。三家都已自发长出 proto-控制平面的部件（AgentScope 的面、OpenCode 的魂、OpenHands 的 Conversation + Agent Server + workspace 分包），只缺统一接口层把它们接起来。

---

> 一句话收尾：**三个仓库、两种语言、三条演化路径，收敛到同一种形状——这不是巧合，是自然选择。**
