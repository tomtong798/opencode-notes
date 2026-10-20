# Conventional Commits

## 是什么

**提交信息规范**：格式 `type(scope): summary`，类型取固定枚举（feat/fix/docs/chore/refactor/test…）。机器可解析（自动生成 changelog、判定版本 bump）。

## 业界类比

- "给 git log 加类型系统"
- angular/semantic-release 社区的通行惯例

## 在 OpenCode 中的角色

- **仓库明文约定**（根 AGENTS.md）：
  - 类型：`feat` `fix` `docs` `chore` `refactor` `test`；scope 可选（core/tui/app/desktop/sdk/plugin…）
  - PR 标题同格式；示例：`fix(tui): simplify thinking toggle styling`
- 配套约定：分支名 ≤3 词连字符（`session-recovery`，**禁** `feat/` 前缀与斜杠）；默认分支 `dev`，PR 目标 `dev`
- **不用 Changesets**：发布走既有流程（根 AGENTS.md 明文禁止加 `.changeset` 文件）
- 真实历史示例：`fix(session-ui): slide the tool group count when it changes`、`chore(core): refresh bundled models.dev snapshot`

## 与其他技术的关系

- **Husky**：提交链的触发器（可校验格式）
- **CHANGELOG.md**：人工/半自动维护的发布记录

## 相关源码

- `AGENTS.md`（Branch Names / Commits and PR Titles 节）、`CHANGELOG.md`
