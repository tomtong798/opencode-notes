# SQLite（WAL 模式）

## 是什么

**嵌入式单文件关系数据库**：无需独立服务进程，整个库就是一个 `.db` 文件，进程内直连，零配置。世界上部署量最大的数据库（每台手机、每个浏览器里都有）。

**WAL（Write-Ahead Logging）模式**：写操作先追加到 `-wal` 日志文件、再择机合并进主库，**读写不再互斥**（对比默认 journal 模式写时阻塞读）。

## 业界类比

- "数据库界的文件格式"——像打开文件一样用 SQL
- WAL ≈ 数据库版的"先写日志再落盘"（与 redo log 思想同源）

## 在 OpenCode 中的角色

- **本地 agent 状态的唯一主存储**：安装即用、零运维，契合 local-first 形态（不需要用户装 PostgreSQL）
- WAL 默认开启：TUI 读历史与 runner 写事件可并发
- **三平台驱动实现**：bun（`bun:sqlite`）/ node（better-sqlite3 系）/ workerd（Cloudflare）各一份（`packages/core/src/database/`）
- 迁移自动应用（`DatabaseMigration.apply`）；按数据库文件路径加信号量防并发 bootstrap
- SDK 嵌入式场景用 `:memory:`
- 云侧同源技术：Cloudflare D1 也是 SQLite（见 [cloudflare-d1](../2.4_infrastructure/2_cloudflare-d1.md)）

## 与其他技术的关系

- **Drizzle ORM**：类型化查询出口（见 [drizzle-orm.md](2_drizzle-orm.md)）
- **事件溯源**：`event`/`event_sequence` 表 + 投影读模型都存在同一库（见 [event-sourcing.md](3_event-sourcing.md)）
- **Effect**：经 `@effect/sql-sqlite-bun` 等包融入 Effect 体系

## 相关源码

- `packages/core/src/database/`（`database.ts`、`sqlite.bun.ts`、`sqlite.node.ts`、`sqlite.workerd.ts`、`migration/`）
- 表定义：`packages/core/src/session/sql.ts`、`event/sql.ts` 等
