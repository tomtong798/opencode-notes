# Husky

## 是什么

**git hooks 管理工具**：把 `pre-commit` 等钩子脚本装进仓库（`.husky/` 目录），团队成员 clone 后钩子自动生效。

## 业界类比

- "让 git hooks 跟着仓库走"的原理解法（原生 hooks 在 `.git/` 里不进版本控制）
- simple-git-hooks / lefthook 的同类

## 在 OpenCode 中的角色

- 根 `package.json` 的 `prepare: "husky"` ——`bun install` 时自动安装钩子
- 提交前触发格式化/检查链（Prettier 等，`.husky/` 内脚本）
- 与 CI 的 `bun run check` 形成"本地拦截 + 云端兜底"两级

## 与其他技术的关系

- **Prettier / oxlint / ast-grep**：钩子内执行的工具
- **conventional commits**：提交信息规范（钩子可校验，见 [conventional-commits.md](10_conventional-commits.md)）

## 相关源码

- `.husky/`、根 `package.json`（`prepare` 脚本）
