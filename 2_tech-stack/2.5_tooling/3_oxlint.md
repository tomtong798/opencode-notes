# oxlint

## 是什么

**oxc 项目（Rust 写的 JS 工具链）出品的 linter**：规则覆盖 ESLint 常用子集（正确性类为主），速度快 50-100 倍。

## 业界类比

- "Rust 重写的 ESLint（常用规则集）"——同项目还有 oxlint-tsgolint（类型感知规则）路线
- ESLint 生态位替代：本项目**不用 ESLint**

## 在 OpenCode 中的角色

- 全仓 lint 主力：`bun run lint` = `oxlint && ast-grep scan … && bun script/sdk-docs.ts`
- 配置在根 `.oxlintrc.json`
- 配合 `lint:changed`（`script/lint-changed.ts`）做增量检查

## 与其他技术的关系

- **ast-grep**：lint 的第二引擎，管"结构化模式"规则（见 [ast-grep.md](4_ast-grep.md)）——oxlint 管通用规则，ast-grep 管仓库特有约定
- **Prettier**：格式化（lint 不格式化）
- **Husky**：本地提交前触发
- **GitHub Actions**：CI 侧跑 `bun run check`

## 相关源码

- `.oxlintrc.json`、根 `package.json` 的 `lint` 脚本
