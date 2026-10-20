# V1 → V2 架构演进（V1 to V2 Evolution）

> 一句话：**V2 于 2025-06 落地（事件溯源会话引擎），V1 遗产收缩为三块兼容面——配置双形状、启动期数据迁移、事件流兼容——"临时共存、待旧库消亡后删除"。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16）；时间线来自 git log。
> [返回索引](README.md)

---

## 1. 两代架构对比

| 维度 | V1 | V2（现行） |
|---|---|---|
| 会话表 | `session` / `message` / `part`（消息拆三表） | `session_v2` / `session_message`（统一） |
| 执行模型 | 内存 tool loop（`SessionPrompt.loop`） | 一步一 `llm.stream`，durable 事件溯源，**禁止内存 tool loop** |
| 状态产生 | 直接写库（状态优先） | 事件 → Bus → projector 同事务投影（事实优先） |
| 作用域 | `Instance` / `InstanceBootstrap` | `Location`（目录/workspace 作用域服务） |
| 恢复 | 无系统化恢复 | write-ahead claim + 有界启动恢复 |
| 事件 | legacy 事件（如 `command.executed`） | durable/ephemeral 事件清单（`durable-event-manifest.ts`，无 V1） |

驱动重构的根本原因：**可恢复性**（进程死后从事件重放状态）、**可观测性**（事实日志天然可审计）、**多客户端一致性**（SSE 事件流成为唯一真相广播）。

## 2. 时间线（git 考证）

| 时间 | 事件 |
|---|---|
| 2025-06 | `76ecf2e58c` "make v2 session inputs event sourced" —— V2 事件溯源落地 |
| 2025-06 | `5caf1f9f6f` "promote v2 to dev" —— V2 成为主干 |
| 2025-07 | `0fbe489e35` "migrate v1 data to v2" —— 数据迁移引擎加入 |
| 2025-07 | `a267222080` "isolate v1 contracts" —— V1 契约隔离进 `schema/src/v1/` |
| 2025-09 | `2c369a21c9` 迁移时对改名工具留提示消息（bash→shell、task→subagent…） |
| 2026-10 | `37a426e7f4`、`ccd3aa787a` —— 持续清理/发布双形状配置 schema |

**注意**：网上流传的部分早期分析引用 `specs/v2/instructions.md` 与根 `CONTEXT.md`，二者**已不存在**（specs/v2 现为 README+session+tools+provider-policy+event-stream+catalog 六篇；`.opencode/glossary/` 是 17 语言 i18n 术语表，不是架构词汇表）。现行权威 = 根 `AGENTS.md` + `specs/v2/`。

## 3. V1 遗产的三个兼容面（现状盘点）

### 面 1：配置双形状（**活跃使用**）

- `core/src/v1/config/`（9 文件：agent/command/mcp/permission/plugin/provider…）
- 消费者：`core/src/config/normalize.ts` **import 全部 V1 配置 schema**——**至今同时接受 V1/V2 两种配置形状**并归一化（`v1/config/migrate.ts` 做 agent 归一、command 迁移、MCP 超时迁移等）
- CLI 侧另有 `cli/src/config/migrate.ts`（V1 TUI 的 tui.json+kv.json → V2）
- 2026-10 还在发布"accepts V1 and V2 shapes"的配置 schema——**这不是死代码**

### 面 2：启动期数据迁移（**兼容 shim，自动触发**）

```
2.2_server/src/routes.ts:190
  └─ V1Migration.layer 在服务启动时 fork（后台 Effect，非 API 触发）
      └─ core/src/database/v1-migration.bun.ts（1135 行迁移引擎）
          ① 清理 event 表旧 V1 事件
          ② 从 opencode-next.db（旧 V2 安装）导入 projects/sessions/messages
          ③ session/message/part（V1）→ session_v2/session_message（V2）
          ④ 处理 compaction part、工具调用、错误归一、附件
          ⑤ 被改名/移除的 V1 工具 → 插入"legacy tool notice"系统消息
          ⑥ 进度存 KV（migration.v1-v2 键）；幂等（完成或无遗留即跳过）
      └─ v1-migration.noop.ts：非 Bun 运行时（workerd）的空实现
      └─ 2.2_server/src/handlers/migration.ts：migration.v1.status 端点
          （required/running+progress/completed/error）
```

数据库侧：`database/migration.ts` 同时探测 `session`（V1）与 `session_v2` 判断迁移需求；存在针对 V1 残留的常规迁移（如 `20260910120000_clear_v1_session_permission.ts`）。

### 面 3：事件流兼容（**manifest 级保留**）

- `schema/src/v1/`（session.ts 667 行/permission/legacy-event/filesystem）
- `schema/src/event-manifest.ts` 的公共清单**仍包含** `LegacyEventV1.Definitions` 与 `FileSystemV1.Event.Definitions`——老客户端订阅事件流不会崩
- `schema/test/v1-isolation.test.ts`：**测试守卫**，强制现行 schema 不得直接 import v1/ 子树

## 4. 判定表：活跃 vs shim

| 组件 | 判定 |
|---|---|
| config/normalize.ts 的 V1 schema | **活跃**（双形状配置是发布特性） |
| cli config 迁移 | 活跃 |
| v1-migration.bun.ts + migration handler | **shim**（旧库消亡即可删） |
| schema v1/ 子树 + manifest 兼容 | shim（事件流向后兼容） |
| v1-isolation 测试守卫 | 长期保留的防腐机制 |
| durable-event-manifest | 纯 V2（无 V1 对应物） |

schema 包 AGENTS.md 原文："V1 coexistence is temporary…delete the V1 subtree when the legacy runtime is retired."——**未设具体删除日期**。

## 5. 给读者/贡献者的启示

1. **读代码**：见到 `session_v2` 表名不是"还有 session_v3 预留"，是 V1 迁移的历史命名
2. **写配置**：V1/V2 形状都能被接受，但新配置应写 V2 形状（以 `docs`/config schema 为准）
3. **改事件**：新事件只进现行 manifest；`LegacyEventV1` 只减不增
4. **数据迁移**：改 `session_message` 结构时必须同步考虑 `v1-migration.bun.ts` 的写入侧
5. **术语对照**：早期代码里的 `Instance`/`InstanceBootstrap`/`SessionPrompt.loop`/`packages/opencode`（现为空壳，旧 CLI 残留在 publish.yml 的 "Build legacy CLI"）对应今天的 `Location`/`session/runner`/`packages/cli`

## 相关篇

- [overview.md](1_overview.md)（V2 引擎全貌）
- [patterns.md](5_patterns.md)（Adapter 一节的 v1 shim）
- [../2_tech-stack/2.3_data/event-sourcing.md](../2_tech-stack/2.3_data/3_event-sourcing.md)
