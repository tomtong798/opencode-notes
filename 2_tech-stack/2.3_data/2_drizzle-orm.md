# Drizzle ORM

## 是什么

轻量 **TypeScript-first ORM**：schema 用 TS 定义、API 形似 SQL（query builder 风格），无代码生成步骤、运行时开销接近零。同族还有 Drizzle Kit（迁移生成）。

## 业界类比

- Prisma 的"更 SQL 味、更轻"替代品：Prisma 用自有 DSL + 引擎，Drizzle 直接是 TS + SQL 心智
- "类型安全的 SQL 模板"

## 在 OpenCode 中的角色

- 全部表的**类型化定义与查询出口**：`session_v2`、`session_message`、`session_inbox`、`instruction_*`、`event`、`permission_saved`、`project`、`worktree`…
- **仓库风格硬规则**（根 AGENTS.md）：字段名用 **snake_case**，让列名无需字符串重定义（`project_id: text().notNull()` 而非 `projectID: text("project_id")`）
- 迁移由 Drizzle 工具链生成；stats 包（PlanetScale/MySQL）也用 Drizzle（`packages/stats/core`）
- 仓库维护两个 Effect 适配包（`effect-drizzle-sqlite`、`effect-sqlite-node`，目前空壳预留）

## 与其他技术的关系

- **SQLite**：驱动层（bun/node/workerd 三实现，见 [sqlite.md](1_sqlite.md)）
- **Effect**：查询包装为 Effect（`@effect/sql-sqlite-bun` catalog 依赖）
- **Effect Schema**：分工——Schema 管**契约**（API/事件形状），Drizzle 管**表结构**；两套系统各司其职，勿混
- **PlanetScale**：MySQL 协议同样用 Drizzle 访问

## 相关源码

- `packages/core/src/session/sql.ts`、`event/sql.ts`、`worktree/sql.ts`、`permission/sql.ts`、`project/sql.ts`
- `packages/core/src/database/drizzle/`
