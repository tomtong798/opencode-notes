# Vite（8.x）

## 是什么

主流前端构建工具，两种工作模式：

- **开发态**：dev server 直接跑浏览器原生 ES Module，按需转译（esbuild 内核），秒级冷启动 + HMR 热更新
- **生产构建**：Rollup 打包（当前也支持 Rolldown）

## 业界类比

- Webpack 的继任者（Rails 8、Nuxt、SvelteKit、SolidStart 的默认选择）
- "开发快是因为不打包，生产稳是因为仍打包"

## 在 OpenCode 中的角色

- **只服务 Web/桌面端**：`packages/app`（Web 应用，也是桌面端渲染器）用 Vite 8 构建
- TUI 与后端由 Bun 直接运行 TS，**不需要 Vite**
- PWA 支持经 Vite 插件体系接入

## 与其他技术的关系

- **SolidJS**：`vite-plugin-solid` 做 JSX/TSX 转译
- **esbuild**：Vite 的依赖预构建内核（见 [esbuild.md](9_esbuild.md)）
- **Electron**：桌面端经 electron-vite 集成 Vite
- **Bun**： Bun 时代 Vite 仍是"浏览器侧"构建的标准件——Bun 管运行，Vite 管浏览器产物

## 特殊用法：Vite 通道（dev 模式）

`packages/cli/dev/vite.ts`（`bun run dev` / `dev:vite`）：开发 TUI 时提前加载 Vite 做 Solid JSX 转译，把 TUI 的 `run` 函数注册为 Bun 插件模块（`build.module("@opencode/tui", ...)`），实现 TUI 热重载开发。

## 相关源码

- Web 应用：`packages/app/`（vite 配置在此包）
- 开发通道：`packages/cli/dev/vite.ts`、根 `package.json` 的 `dev:vite` 脚本
