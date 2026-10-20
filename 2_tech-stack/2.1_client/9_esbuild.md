# esbuild

## 是什么

Go 编写的超高速 JS/TS 编译打包器：转译速度比传统工具（webpack/tsc）快 10-100 倍。只做转译与打包，不做类型检查。

## 业界类比

- "转译内核界的跑车"——Vite 的依赖预构建也用它
- Bun 打包器出现前生态最快的通用打包器

## 在 OpenCode 中的角色

- **VS Code 扩展（`sdks/vscode`）** 的独立构建工具（`./dist/extension.js` 产物）
- 间接出现于 Web 构建（Vite 内核）与 Vite 通道（JSX 转译链路）

## 与其他技术的关系

- **Vite**：esbuild 是其依赖预构建与转译内核（见 [vite.md](4_vite.md)）
- **Bun**：Bun 自带打包器，仓内大部分场景由 Bun 覆盖；esbuild 只在 VS Code 扩展等特定构建里直接使用
- **TypeScript**：esbuild 只擦除类型不检查；类型检查由 `bun typecheck` 承担

## 相关源码

- `sdks/vscode/`（构建脚本）
