# CodeMode（@opencode/codemode）

## 是什么

opencode 自研的**沙箱 JS/TS 解释器**：基于 **Acorn**（轻量高性能 JS 解析器，产 AST）解析代码后，用自己的 Effect 原生解释器逐节点执行——**不是 eval、不是子进程**。AI 生成的代码只能调用显式暴露的工具 API（Wrapper/Opaque 对象模型），无法逃逸访问宿主环境。

## 业界类比

- "给 LLM 的 Web Worker + 受控 SDK"：隔离执行 + 白名单 API
- 类似 Pyodide 的"语言内嵌解释器"思路，但目标是**安全执行不可信代码**

## 在 OpenCode 中的角色

- **工具调用的另一种模式**：注册 `codemode !== false` 的工具进入 CodeMode 目录，模型可以"写一段代码批量调工具"（组合、循环、条件）而非逐个发起工具调用；`codemode: false` 的工具直接提供给 provider
- 注册表为 CodeMode 保留 `execute` 名字；工具的 `output`（schema 校验的机器值）就是给 CodeMode 消费的
- 自带标准库与 OpenAPI 工具定义生成（`tool*.ts`）
- 语言特性覆盖记录在 `interpreter-support.md`

## 与其他技术的关系

- **Acorn**：解析内核（见术语）
- **@opencode/ai / Tool 注册表**：CodeMode 是工具执行路径的"编排模式"，规范见 `specs/v2/tools.md`（"Code Mode receives"的 output 即机器值）
- **Effect**：解释器本身用 Effect 实现，执行可中断（遵守"中断不是工具结果"定律）

## 相关源码

- `packages/codemode/src/`（`codemode.ts`、`interpreter/`、`tool*.ts`）
- core 侧目录：`packages/core/src/codemode/`
- 规范：`specs/v2/tools.md`
