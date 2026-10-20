# SST（Serverless Stack）

## 是什么

用 **TypeScript 定义云基础设施并部署**的开源框架：以组件树描述资源（函数、站点、数据库、队列），一键部署到 AWS / Cloudflare 等平台。

## 业界类比

- "terraform，但写 TS、带热开发和类型安全"
- Cloudflare 的资源是"一等公民"而非插件（sst v3/Ion 路线）

## 在 OpenCode 中的角色

- 全部云侧资源的**编排层**：根 `sst.config.ts` + `infra/` 模块（`app.ts`、`console.ts`、`monitoring.ts`、`secret.ts`、`share.ts`、`stage.ts`、`stats.ts`、`webapp.ts`）
- 平台组合：**Cloudflare 为主**（home），AWS us-east-1 为辅；集成 Stripe、PlanetScale、Honeycomb
- 环境分 stage：`production`（保留资源）、`beta`（最小，仅 WebApp）、`vimtor`（监控）
- 本地开发：`bun sst shell --stage=production -- ...` 注入云环境（如 `dev:stats`）

## 与其他技术的关系

- **Cloudflare Workers / D1 / Durable Objects**：被编排的目标资源
- **PlanetScale / Stripe / Honeycomb**：作为 provider 集成
- **GitHub Actions**：CI 中执行 SST 部署
- **Astro / SolidStart 站点**：作为 Web 组件部署

## 相关源码

- `sst.config.ts`、`infra/*.ts`、`sst-env.d.ts`
