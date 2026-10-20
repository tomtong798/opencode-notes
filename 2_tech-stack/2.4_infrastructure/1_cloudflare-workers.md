# Cloudflare Workers

## 是什么

跑在**边缘 V8 isolate** 上的 serverless 计算平台：代码部署到全球节点、请求就近执行，isolate（非容器）启动即响应（无冷启动）。

## 业界类比

- "更快的 AWS Lambda，但跑在全球边缘"
- V8 isolate 共享进程，比容器/微 VM 启动快几个数量级

## 在 OpenCode 中的角色

云侧函数全部承载于 Workers（均为辅助服务，产品核心在本地）：

| 服务 | 用途 |
|---|---|
| `packages/function`（Hono） | 会话分享 API（配 Durable Objects）、GitHub App token 交换、飞书/Discord 桥 |
| `services/update` | 桌面端 electron-updater 的更新元数据（配 D1） |
| `services/changelog` | `/changelog` 发布说明 |
| `services/files` | 公共文件分发 |
| console/stats 站点 | SolidStart 亦以 Workers 为部署目标 |

注意：opencode 本地代码里有 `sqlite.workerd.ts`——SDK/云场景嵌入时同一核心可跑在 Workers 运行时。

## 与其他技术的关系

- **Hono**：`packages/function` 用的超轻 Web 框架（Workers 原生生态标配，类比 Express 之于 Node）
- **Durable Objects / D1**：Workers 的配套状态原语（各有专篇）
- **SST / Wrangler**：部署编排（SST 为主，Wrangler 辅助）
- **GitHub Actions**：CI 内部署

## 相关源码

- `packages/function/src/api.ts`、`services/*/`
- `sst.config.ts`、`infra/`
