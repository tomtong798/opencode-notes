# 自研特性卷（Features）

> 对象：opencode fork（分支 `user/donghe/dev`）——本目录文档描述的是**自己的开发特性**，非上游 opencode。
> ⚠️ **基线差异**：本卷的 `packages/opencode/...` 等路径与行号基于 fork 代码树，与全库其他卷的上游 `dev`（v2.0.30）基线**不同**，行号不可跨卷互查；上游架构参考见 `1_architecture/`、`6_modules/`。
> 来源：从 gitee 旧库 `opencode-notes/features`（2026-08 建立）整体迁入，内容未改动。

## 特性索引

| # | 特性 | 一句话 |
|---|---|---|
| 01 | [desktop-config](01-desktop-config/desktop-skill-agent-config-feature.md) | Desktop 设置里的 Skill/Agent 可视化管理 |
| 02 | [ganet-group-chat](02-ganet-group-chat/agent-group-chat-feature.md) | Agent 群聊（多 agent 会话面板） |
| 03 | [session-dashboard](03-session-dashboard/session-dashboard-feature.md) | 会话仪表盘 |
| 04 | [pet](04-pet/pet-system-feature.md) | 桌面宠物系统 |
| 05 | [recap](05-recap/session-recap-feature.md) | 会话回顾 |
| 06 | [goal](06-goal/goal-feature.md) | 目标追踪 |
| 07 | [dynamic-workflow](07-dynamic-workflow/dynamic-workflow-feature.md) | 动态工作流 |
| 08 | [wechat](08-wechat/wechat-channel-feature.md) | 微信通道（[架构全解](08-wechat/WECHAT_BOT_ARCHITECTURE.md)） |
| 09 | [memory](09-memory/memory-feature.md) | 持久化记忆系统（[深度分析](09-memory/持久化记忆系统深度分析.md)、[蒸馏扫描流程](09-memory/distill-scan-process.md)） |
| 10 | [ppt-agent](10-ppt-agent/ppt-agent-plugin-design.md) | PPT 模板生成插件（方案设计，未实施；对标 WorkBuddy） |

另：[opencode-expert-design-vs-workbuddy.md](opencode-expert-design-vs-workbuddy.md)——opencode 专家设计 vs WorkBuddy 的对比分析。

## 与全库的关系

- 本卷是**产出物文档**（描述自己写的特性），其余卷是**学习笔记**（理解上游代码）——性质不同，故独立成卷不编号入主线。
- 写特性时用到的上游机制参考：插件系统 → [../6_modules/13_plugin.md](../6_modules/13_plugin.md)；配置 → [../6_modules/2_config.md](../6_modules/2_config.md)；技能 → [../6_modules/10_skill.md](../6_modules/10_skill.md)；Agent → [../6_modules/8_agent.md](../6_modules/8_agent.md)。
