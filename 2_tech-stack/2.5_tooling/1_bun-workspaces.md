# Bun Workspaces

## 是什么

包管理器的 **monorepo 协议**：根 `package.json` 声明 `workspaces` 目录通配，内部包互相以包名引用，共享单一 lockfile，内网包改动即时生效（符号链接 node_modules）。

## 业界类比

- npm/pnpm/yarn workspaces 的 Bun 版
- "把多包仓库当一个项目管依赖"

## 在 OpenCode 中的角色

- 管 **~40 个包**的依赖拓扑：workspace 目录为 `packages/*`、`packages/plugins/*`、`services/*`、`packages/console/*`、`packages/stats/*`
- **catalog 机制**：根 `package.json` 的 `workspaces.catalog` 集中钉版本（`effect`、`@effect/*`、`@opentui/*` 等），各包以 `"effect": "catalog:*"` 引用——避免 40 个包版本漂移
- 依赖方向铁律在这个拓扑上执行：Schema ← Core ← Protocol ← Server；Client 不依赖 Core/Server（见 [layered-monorepo](../2.6_architecture-styles/4_layered-monorepo.md)）

## 与其他技术的关系

- **Bun**：运行时与安装器（见 [../2.2_server/bun.md](../2.2_server/2_bun.md)）
- **Turborepo**：workspaces 管依赖，turbo 管任务与缓存（见 [turborepo.md](2_turborepo.md)）
- **turbo.json 的 transit 任务**：把 workspace 依赖折叠进测试哈希，"不实际构建也敏感于依赖变化"

## 相关源码

- 根 `package.json`（workspaces + catalog）、`bun.lock`
