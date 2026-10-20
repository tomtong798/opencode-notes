# PlanetScale

## 是什么

**serverless MySQL 平台**（Vitess 驱动——YouTube 起源的 MySQL 分片中间件）：提供分支（branching）、非阻塞 schema 变更、连接池按用量计费。

## 业界类比

- "MySQL 界的 Netlify/Vercel"——托管的、按用量的数据库即服务
- 对比 Supabase/Neon（Postgres 系）：PlanetScale 是 MySQL 阵营

## 在 OpenCode 中的角色

- **仅内部运营使用**：stats 分析数据（`packages/stats/core`——用户匿名使用统计的上报与聚合）
  - Drizzle ORM 访问（MySQL 协议）
  - 有迁移脚本、唯一用户约束、Honeycomb 回填任务
- **与产品本体无关**：本地 opencode 运行不依赖任何 PlanetScale

## 与其他技术的关系

- **SST**：作为 provider 集成进基础设施定义（`sst.config.ts`）
- **Drizzle ORM**：访问层
- **stats 三件套**：`stats/app`（SolidStart 前端）+ `stats/core`（域服务）+ `stats/server`（同步进程）

## 相关源码

- `packages/stats/core/`（`database/`、`domain/`、迁移）
- `infra/stats.ts`（SST 资源定义）
