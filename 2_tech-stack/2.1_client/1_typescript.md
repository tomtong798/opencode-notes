# TypeScript

## 是什么

带静态类型的 JavaScript 超集，编译（转译）到 JS 运行。类型在编译期检查，运行时擦除。

## 业界类比

- JavaScript 加上"编译期安全网"
- 大型 JS 项目的事实标准（Angular/VS Code/全仓 opencode 均为 TS-first）

## 在 OpenCode 中的角色

- 全部 ~40 个包的统一语言，从终端 UI 到后端服务到云函数
- 仓库以 strict 风格使用：**禁 `any`**、禁类型抑制（`@ts-ignore`）、依赖类型推断而非冗余注解（见根 `AGENTS.md` 风格指南）
- 类型即文档：Effect Schema 的契约、codegen 生成的客户端，都以 TS 类型为最终交付物

## 与其他技术的关系

- **Bun** 原生执行 TS（无需 `tsc` 预编译），这是全仓能"源码即运行"的基础
- **Effect / Effect Schema / HttpApi** 都是 TS 生态库，深度依赖 TS 类型系统（如条件类型、泛型推断）
- **Vite / esbuild** 负责 TS 转译（Web 端、VS Code 扩展）
- 类型检查跑 `bun typecheck`（各包内），**不直接跑 `tsc`**

## 相关源码

- 风格约束：根目录 `AGENTS.md` → Style Guide
- 根 `tsconfig.json`；`bun run check`（lint + typecheck 的权威校验）
