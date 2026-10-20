# Bun

## 是什么

三位一体的 JavaScript 工具链，Zig 编写：

- **运行时**：JavaScriptCore 引擎（Safari 的 JS 引擎），Node.js API 高度兼容的替代品（"什么是运行时"的入门讲解见 [js-runtime.md](1_js-runtime.md)）
- **包管理器**：`bun install`（对标 npm/pnpm/yarn，快数倍）
- **测试运行器**：`bun test`（Jest 兼容 API）
- 另含打包器与 `Bun.file()` 等原生 API

类比：*`node + npm + jest` 打包成一个更快替代品*。

## 业界类比

- Node.js 生态的"全套替换"尝试
- 仓库约定：能用 Bun API 就用（`Bun.file()` 优先于 `fs.readFile`，见根 AGENTS.md）

## 在 OpenCode 中的角色

- **从开发到生产全程使用**：TUI、后端服务、测试都直接 `bun` 起跑 TS 源码（无需预编译）
- **包管理**：`bun install` + `bun.lock`，workspaces 协议管 monorepo（见 [bun-workspaces](../2.5_tooling/1_bun-workspaces.md)）
- **版本要求**：≥1.3（`packageManager: bun@1.4.2`）
- 运行时差异处理：SQLite 驱动按 bun/node/workerd 三平台分实现（`packages/core/src/database/`）

## 与其他技术的关系

- **TypeScript**：Bun 原生执行 TS，是全仓"源码即运行"的基础
- **Vite**：Bun 管运行时与依赖，Vite 管浏览器侧构建（Web/桌面），TUI 开发态经 Vite 通道做 JSX 转译
- **Node.js**：桌面 Electron（主进程）与部分工具链仍是 Node；VS Code 扩展宿主天然是 Node
- **Turborepo**：任务编排跑在 Bun 上（`bun turbo typecheck`）

## 相关源码

- 根 `package.json`、`bunfig.toml`、`bun.lock`
- `packages/core` postinstall 脚本 `fix-node-pty`（跨运行时修补示例）
