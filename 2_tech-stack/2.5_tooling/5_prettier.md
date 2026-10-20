# Prettier

## 是什么

**无配置之争的代码格式化标准件**：按固定规则重排格式（缩进、换行、引号），消灭格式讨论。

## 业界类比

- gofmt / rustfmt 的 JS/TS 对应物
- "格式问题机器定，风格问题人才谈"

## 在 OpenCode 中的角色

- 全仓格式化（`.editorconfig` + `.prettierignore` 配套）
- **代码生成链的一环**：httpapi-codegen 生成的客户端代码经 Prettier 格式化后再落盘（生成物也符合仓内格式）
- 提交链：Husky 钩子内执行

## 与其他技术的关系

- **Husky**：提交前触发（见 [husky.md](6_husky.md)）
- **oxlint / ast-grep**：格式化（Prettier）与检查（oxlint/ast-grep）分工，不重叠
- **httpapi-codegen**：生成物消费者

## 相关源码

- `.prettierignore`、`.editorconfig`、`.husky/`
