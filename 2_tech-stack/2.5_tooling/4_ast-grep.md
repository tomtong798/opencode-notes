# ast-grep

## 是什么

基于**语法树（AST）**的结构化搜索/改写工具：用"代码模式"（形似代码本身的模式，或 YAML 规则）匹配与重写代码，而非正则匹配文本。支持 25+ 语言，规则有测试框架（`ast-grep test`）。

## 业界类比

- "给代码用的正则，但懂语法"——不会匹配到字符串/注释里的同形文本
- semgrep 的轻量本地版

## 在 OpenCode 中的角色

**lint 的第二引擎**，管仓库特有约定（oxlint 管通用规则）。三个规则族：

| 规则族 | 命令 | 用途 |
|---|---|---|
| Effect 模式 | `lint:effect-patterns` | core/util/2.2_server/protocol/cli 的 Effect 写法约束 |
| Effect 简化 | `lint:effect-simplifications` | 可简化的 Effect 代码（带 `--off=unused-suppression`） |
| GUI 扩展 | `lint` 中的 `gui-extensions` 规则 | packages/gui 的约定 |

- 规则测试：`test:lint-rules`、`test:effect-simplification-rules`（`ast-grep test` 跑 YAML 规则的快照测试）

## 与其他技术的关系

- **oxlint**：并列 lint 引擎（见 [oxlint.md](3_oxlint.md)）
- **YAML**：规则以 sgconfig.yml + 规则文件表达
- **CI**：`bun run check` 的一部分

## 相关源码

- `script/ast-grep/`（`sgconfig.yml`、`effect-simplifications/`、`gui-extensions/`）
