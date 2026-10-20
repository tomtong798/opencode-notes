# @opencode/ai

## 是什么

opencode 自研的 **LLM provider 抽象层**：把 ~50 家模型供应商（OpenAI、Anthropic、AWS Bedrock、Google Vertex、GitHub Copilot、models.dev 目录里的开源网关…）统一成一套流式接口，抹平认证、重试、传输差异。

- 入口：`./llm`（流式）、`./providers`、`./promise`（Promise API）、`./testing`
- provider 以 npm 包形式可插拔（内置包 + npm 安装扩展），新供应商优先加到 models.dev 而非改 opencode（CONTRIBUTING 明文）

## 业界类比

- 自研版 LiteLLM / Vercel AI SDK
- "AI 界的数据库驱动层"——上层只面对统一的 `LanguageModel` 接口

## 在 OpenCode 中的角色

- **core 的模型底座**：
  - `Provider.Service`（provider 目录/包加载/可用性）与 `ModelResolver.Service`（目录模型 → 运行时 `LanguageModel`，含凭证注入、variant overlay）
  - Session Runner 每 Step 经 `llm.stream()` 发起一次调用（一步一流式调用，规范见 `specs/v2/session.md`）
  - 流式事件（文本/推理/工具输入 delta）经 publisher 发布到 Bus → SSE
- 测试：provider 集成测试跑 `@opencode/http-recorder` 磁带（不真调 API）
- 前向兼容是硬规则：未知字段必须透传（provider 测试断言）

## 与其他技术的关系

- **CodeMode**：模型"写代码调工具"时，工具 API 由 codemode 暴露（见 [codemode.md](12_codemode.md)）
- **Effect**：基于 Effect 实现流式与并发
- **models.dev**：外部 provider/模型目录仓库（同为 anomalyco 维护），内置快照随仓库刷新（`chore(core): refresh bundled models.dev snapshot`）

## 相关源码

- `packages/ai/src/`（`llm.ts`、`providers.ts`、`promise.ts`、`testing.ts`、`schema/`）
- core 侧：`packages/core/src/provider.ts`、`model.ts`、`model-resolver.ts`、`session/model-request.ts`、`session/runner/step.ts`
