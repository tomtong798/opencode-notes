# Cloudflare D1

## 是什么

Cloudflare 的 **serverless SQLite**：SQLite 数据库作为服务跑在边缘，单库就近复制（read replication），SQL 接口 + HTTP API。

## 业界类比

- "Turso 的 Cloudflare 版"——嵌入式 SQLite 的云托管形态
- 对比 PlanetScale/Neon：D1 是 SQLite 方言，不是 MySQL/Postgres

## 在 OpenCode 中的角色

- **更新元数据服务**（`services/update`）：桌面端 electron-updater 检查更新所需的数据（版本、产物地址、更新策略）
- 用 jose（JWT）+ semver + yaml 处理发布元数据；带 D1 迁移脚本

## 与其他技术的关系

- **SQLite**：同一数据库技术的云侧延续（本地 SQLite ↔ 边缘 D1，技术同源、场景互补，见 [sqlite.md](../2.3_data/1_sqlite.md)）
- **Cloudflare Workers**：D1 是 Workers 的配套存储
- **electron-updater**：消费方（桌面自动更新）

## 相关源码

- `services/update/`（含 D1 migrations）
