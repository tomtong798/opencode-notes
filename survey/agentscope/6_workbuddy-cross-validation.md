# 交叉验证：WorkBuddy 的独立分析评价

> 背景：用户让另一个 AI 助手 WorkBuddy（腾讯 CodeBuddy 家族）独立分析了 AgentScope（归属考证 + 代码分析 + 与 OpenHands SDK 对比），会话存档于 workbuddy.link（2026-10-19）。本篇逐条核验其结论，评价参考价值，并吸收其增量信息。
> 与本系列的关系：第 1/5 篇是我们本地克隆的代码实证，本篇引入**独立第二意见**做交叉验证。
> [返回系列索引](README.md) | [返回笔记索引](../../README.md)

---

## 目录

- [1. 结论先行](#1-结论先行)
- [2. WorkBuddy 结论核验表](#2-workbuddy-结论核验表)
- [3. 增量信息：值得吸收的三件事](#3-增量信息值得吸收的三件事)
- [4. 短板与需过滤项](#4-短板与需过滤项)
- [5. 方法论意义](#5-方法论意义)

---

## 1. 结论先行

> **WorkBuddy 的分析有相当高的参考价值**：其代码级论断与本地仓库全部吻合（核验通过率 100%），且带来了三项我们没有的增量信息。定位是**交叉验证的独立第二意见**——两条独立实证路线（它走 GitHub API，我们走本地克隆）收敛到相同结论，双方置信度均应上调。

WorkBuddy 的分析方法：WebSearch 确认归属 → GitHub API 拉取全仓 tree + 核心源文件（base64）→ releases 端点获取版本历史。非纯训练数据回忆，是实际代码实证。

## 2. WorkBuddy 结论核验表

### 2.1 代码级论断（本地仓库逐条核验）

| WorkBuddy 论断 | 本地核验 | 结果 |
|---|---|---|
| 中间件 7 钩子（on_reply / on_reasoning / on_acting / on_check_permission / on_model_call / on_compress_context / on_system_prompt） | `middleware/_base.py` grep 实测，7 个 `def on_*` 一个不差 | ✅ |
| 权限五模式 DEFAULT / EXPLORE / ACCEPT_EDITS / BYPASS / DONT_ASK | `permission/_decision.py` + `_context.py` 实测 | ✅ |
| `_next_action()` 返回 Reasoning / Acting / Exit 三种 pydantic 对象，决策与执行分离 | `agent/_utils.py`：`class Acting/Reasoning/Exit` | ✅ |
| `reply()` 是 `_reply_stream()` 的包装；核心在 `_reply_impl` 的 `while True + match next_action` | 与我们读到的 3922 行 `_agent.py` 结构一致 | ✅ |
| 上下文压缩是模型主动调用的 `CompressContext` 工具 | `_COMPRESSION_TOOL_NAME = "CompressContext"`（`_agent.py:109`） | ✅ |
| 事件流二十多种（TextBlockStart/Delta/End 等） | `event/` 30 个事件类 | ✅ |
| `src/agentscope` 下 502 个 `.py` 文件 | `find | wc -l` = 502 | ✅ |
| `_agent.py` 3922 行、全仓只有一个 `Agent` 类 | 实测一致 | ✅ |
| 人机协同"挂起"：需确认时返回 `Exit(exit_events=None)`，外部传 `UserConfirmResultEvent` 继续 | 与 `ToolCallState.asking` 状态机吻合，且补全了触发路径 | ✅ |
| Bash 工具用 tree_sitter_bash 判断命令是否只读 | pyproject.toml `tree_sitter_bash` 依赖佐证 | ✅ |

### 2.2 归属考证

| WorkBuddy 结论 | 评价 |
|---|---|
| 阿里系：达摩院出身 → 通义实验室主导；`agentscope-ai` GitHub 组织；最初在 `modelscope/agentscope` | 合理，与 README/论文一致 |
| Apache-2.0 | ✅（本地 LICENSE 一致） |
| 易混淆项提示（ModelScope-Agent / 百炼） | 有价值的澄清 |

### 2.3 与 OpenHands 对比的核心论断

> "它们是**两套彼此独立的实现，共同对齐了同一套设计范式**。"

与第 5 篇结论一致（我们表述为"环境在选择形状，不是抄袭"）。它给出的四项差异（事件模型二元 vs 三元流、控制循环归属 Conversation vs Agent、四包 vs 单包、独有模块清单）方向正确，深度不及我们的三方对照。

## 3. 增量信息：值得吸收的三件事

### 3.1 版本时间线（GitHub releases API 实证，我们没有）

```
AgentScope v0.0.1     2024-02-26
AgentScope v1.0.0     2025-08-15
OpenHands SDK 建库     2025-08-23   ← 仅比 AgentScope 1.0 晚 8 天
AgentScope v2.0.0     2026-05-25
```

**强化第 5 篇的"平行演化"论断**：两家几乎同起点、同节奏重构，进一步排除互相模仿的可能。（注意：OpenHands SDK 建库晚于 AgentScope，但 OpenDevin 主仓 2024-03 已立项——SDK 是重构产物，演化起点仍是平行的。）

### 3.2 Claude Code 范本归因（比我们更具体的解释框架）

WorkBuddy："两边都受 Claude Code / Codex CLI 范式影响……**真正被模仿的是那个事实标准**，两边并没有互相模仿。"

硬证据：AgentScope 权限五模式与 Claude Code 的 `--permission-mode` 几乎同名（DEFAULT/ACCEPT_EDITS/BYPASS...）。这比第 1 篇"Anthropic blocks / MCP 塑造"的抽象归因更进一层——**塑造 agent runtime 的不只是模型 API，还有先行产品的交互范式**。上游约束链：模型 API → Claude Code 范式 → 各家 runtime。

### 3.3 工程债观察（我们对比文档的盲区）

- `_agent.py` 3922 行，需 `# pylint: disable=too-many-branches` 抑制；
- 28 个重依赖（anthropic/openai/dashscope/tree_sitter）全在主 `dependencies`，未拆 extras；
- `requires-python >= 3.11`，3.9/3.10 无法运行。

这些批评性观察可补进第 1 篇的工程实践对比（AgentScope 侧的减分项）。

## 4. 短板与需过滤项

| # | 问题 | 说明 |
|---|---|---|
| 1 | **事实前后矛盾** | v1.0 日期第一轮说"2025-09-02"，第三轮 releases 数据说"2025-08-15" |
| 2 | **无证据论断（疑似幻觉）** | "同时有 Java/TypeScript 版本，Go 版在推进"——tree 中无任何佐证 |
| 3 | **协议分歧未披露** | reasoning 中发现 MIT 说法（"one source said MIT"），最终回复只给 Apache-2.0，未说明分歧 |
| 4 | **对比缺第三点** | 只有 AgentScope vs OpenHands 双点，无 OpenCode 跨语言数据点，无"证据三角→引擎收敛"推理链 |
| 5 | **归属深度有限** | 未触及 pyproject.toml 作者字段（`SysML team of Alibaba Tongyi Lab`），本地可直接验证 |

## 5. 方法论意义

两条独立的实证路线收敛：

| | 本系列（第 1/5 篇） | WorkBuddy |
|---|---|---|
| 数据获取 | 本地 git clone，逐文件读取 | GitHub API（tree + contents base64 + releases） |
| 对比对象 | AgentScope / OpenCode / OpenHands 三方 | AgentScope / OpenHands 双方 |
| 独有产出 | 跨语言收敛证据、价值框架、引擎收敛预测 | 版本时间线、Claude Code 范本归因、工程债观察 |
| 错误 | — | v1.0 日期矛盾、Java/Go 版本疑似幻觉 |

结论：**代码级论断 100% 吻合 + 各有对方没有的增量** = 理想的交叉验证对象。对 AI 助手输出的引用规范：核心论断需本地验证（本篇 §2.1 即为示范），增量信息标注来源，可疑项明确标记。

---

> 一句话收尾：**两个独立实证、一条相同结论——"同构是环境选择的"这一判断，现在有了双盲验证级别的置信度。**
