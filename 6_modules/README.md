# OpenCode 模块详解卷（Modules）

> 对象：`/home/donghe/github/opencode`（v2.0.30，`dev` 分支）。
> 定位：**模块参考深度**——回答"某个子系统内部如何工作"（查阅型）。与兄弟卷的分工：`1_architecture/` 回答全局形态，`5_timeline/` 回答一条链路的时序，本卷回答"进到某个模块内部"。
> 前身：gitee 旧库 `5_modules`（2026-08-03 基线，22 篇）。本卷按现码重建——旧库的 LSP/Todo/opencode-src 模块已随代码退役，V1/V2 并存叙事已过时（`core/src/v1/` 仅剩 config），新增了 ai 协议层、subagent、compaction 等旧库没有的模块。

---

## 一、心智模型（继承自旧库，按现码校准）

理解 OpenCode 运行时，最快的方式是抓住**四个领域核心 + 一个贯穿机制**：

| 模块 | 角色 | 类比 | 星级 |
|---|---|---|---|
| **Session** | 编排核心（聚合根） | 舞台 | ★★★★★ |
| **Tool** | 执行能力（能做什么） | 道具 | ★★★★★ |
| **Agent** | 行为控制（谁在演、什么设定、什么权限） | 演员 | ★★★★ |
| **Permission** | 安全边界（允许做什么） | 护栏 | ★★★★ |
| **Event/Bus** | 状态变更记录（贯穿上面所有） | 记录员 | ★★★★ |

```
┌─────────────────────────────────────────────┐
│  Session（聚合根，编排一切）                   │
│  prompt → inbox 准入 → drain → Step → 投影   │
└──────────┬──────────────┬──────────┬────────┘
           │              │          │
     ┌─────▼─────┐  ┌─────▼────┐  ┌──▼────────┐
     │ Agent     │  │ Tool     │  │ Permission│
     │ 谁在执行   │  │ 能做什么  │  │ 允许做什么 │
     └───────────┘  └─────┬────┘  └───────────┘
                          │
                   ┌──────▼──────┐
                   │ Event 溯源   │ ← 记录所有状态变更
                   └─────────────┘
```

底层支撑：Effect 基建（DI 组装）→ 数据层（事件真相源 + 投影读模型）→ 配置/模型目录。

## 二、篇目（17 篇，依赖序）

**排序逻辑**：先地基后依赖者（顺读无前向引用）；核心域内部按"一条 prompt 途经的模块"排。

### 第一批（★★★★★，理解运行时的最短路径）

| # | 篇 | 覆盖 | 状态 |
|---|---|---|---|
| 0 | [0_effect-infra.md](0_effect-infra.md) | Effect 基建：`core/src/effect/`（服务组装/State） | ✅ |
| 1 | [1_data-layer.md](1_data-layer.md) | 数据层：database/kv/bus/event（真相源与投影） | ✅ |
| 4 | [4_session.md](4_session.md) | Session 聚合：43 文件（准入/协调/读取/操作） | ✅ |
| 9 | [9_tool.md](9_tool.md) | 工具系统：定义/执行管线/15 内置/MCP 桥接 | ✅ |

### 第二批（行为与上下文域）

| # | 篇 | 覆盖 | 状态 |
|---|---|---|---|
| 3 | [3_model-protocol.md](3_model-protocol.md) | 模型目录 + `packages/ai` 协议层（anthropic/openai/gemini/responses） | ✅ |
| 5 | [5_inbox-coordination.md](5_inbox-coordination.md) | 准入与协调深入（steer/queue 词汇全语义） | ✅ |
| 6 | [6_context.md](6_context.md) | 上下文代数：system-prompt/instructions/context | ✅ |
| 7 | [7_compaction.md](7_compaction.md) | 压缩：摘要替换前缀、三路触发、预算/缩减 | ✅ |
| 7.1 | [7.1_session-operations.md](7.1_session-operations.md) | └ 会话四操作：revert/move/transfer/diff（零耦合，语义同位合卷） | ✅ |
| 8 | [8_agent.md](8_agent.md) | Agent 注册表/切换/权限内嵌 | ✅ |
| 11 | [11_permission.md](11_permission.md) | 权限评估/ruleset/saved | ✅ |
| 12 | [12_subagent.md](12_subagent.md) | 子会话派生与回收 | ✅ |

### 第三批（支撑与集成，按需）

| # | 篇 | 覆盖 | 状态 |
|---|---|---|---|
| 2 | [2_config.md](2_config.md) | 配置发现/加载/合并（8 层优先级+插件源） | ✅ |
| 10 | [10_skill.md](10_skill.md) | 技能发现与注入（三路注入） | ✅ |
| 13 | [13_plugin.md](13_plugin.md) | 插件宿主：同进程 hooks/四来源通道 | ✅ |
| 14 | [14_mcp.md](14_mcp.md) | MCP 集成：连接生命周期/能力物化 | ✅ |
| 15 | [15_location.md](15_location.md) | Location 域：五实体关系/LayerMap 缓存 | ✅ |
| 16 | [16_pty.md](16_pty.md) | PTY：伪终端两档（进程内/外部 daemon 找回） | ✅ |
| 17 | [17_job.md](17_job.md) | Job 注册表：后台任务账本（身份/通知路由/恢复描述） | ✅ |
| 18 | [18_credentials.md](18_credentials.md) | 凭证体系：存储三态/优先级/OAuth 尝试机/Copilot·Console 专层 | ✅ |
| 18.1 | [18.1_server-access.md](18.1_server-access.md) | └ 服务访问：密码/HMAC 会话/配对码（零耦合的第三把钥匙） | ✅ |
| 19 | [19_context-engineering.md](19_context-engineering.md) | 上下文工程：把指令/压缩/输出有界化/投递四篇串成统一视角 | ✅ |

编号即推荐阅读序内的相对位置；4/5/9 等编号为固定锚点（与目录规划对齐），未写的编号不重排。

## 三、防重复规则（本卷的生存纪律）

| 内容 | 归属 | 本卷做法 |
|---|---|---|
| 执行链时序（runner/step/publisher/fiber） | `5_timeline/5` | 只写结构，一句互链 |
| Location 惰性编译时序 | `5_timeline/3` | 只写域模型 |
| server 装配/启动时序 | `5_timeline/2`、`1_architecture/1` | 不单篇 |
| Effect/SolidJS/Bun 等技术本体 | `2_tech-stack/` | 只写 opencode 的用法基建 |
| 前端消费（store 投影/TUI reducer） | `5_timeline/4-6` | 不涉及 |
| 业务表逐列（session 等） | 本卷 4_session | 其他篇只做全景表 |

## 四、篇目规范

- 每篇统一骨架：引言块（一句话 + 基线 + 权威出处 + 互链）→ §0 模块地图 → §1 心智模型 → 分层走读 → 不变量与设计取舍 → 失败与恢复 → 演进注记 → 相关篇
- 执行流/分支一律 ASCII 纵向缩进树（├─ └─ ★）；清单/表结构可用表格
- 行号当日实测；代码演进后以符号名检索为准
- 权威优先级：`specs/v2/` > 各包 AGENTS.md > 源码 > 本笔记

## 五、与旧库（gitee `5_modules`）的模块生死对照

| 旧库模块 | 现状 | 处置 |
|---|---|---|
| LSP（4.21） | `core/src` 已无 lsp 目录 | 不迁移 |
| Todo（4.14） | 独立模块已消失 | 不迁移 |
| opencode-src 目录介绍 | `packages/opencode/src` 已不存在（拆成 cli/tui 等） | 由 `1_architecture/2_package-map.md` 接替 |
| V1/V2 并存速查 | V1 基本退役（`core/src/v1/` 仅剩 config） | 归 `1_architecture/6_v1-to-v2-evolution.md` |
| 其余 19 个 | 路径漂移但存活 | 本卷按现码重写 |
| （无） | `packages/ai` 协议层、subagent、compaction 为新增域 | 本卷新篇（3/7/12） |
