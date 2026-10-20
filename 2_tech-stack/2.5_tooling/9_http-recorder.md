# http-recorder（VCR 模式测试）

## 是什么

opencode 自研的 **HTTP/WS 录制回放库**（`@opencode/http-recorder`）：VCR 模式 = 首次真实请求录成"磁带"（JSON fixture），此后测试回放磁带——确定性、离线、免费、快速。

## 业界类比

- Ruby VCR / Polly.js / nock.record 思想在 Bun/Effect 栈的实现
- "外部依赖的时光机"——供应商响应被固化成 fixture

## 在 OpenCode 中的角色

- **LLM provider 测试的关键设施**：`@opencode/ai` 与 core 的 provider 集成测试跑磁带，不真调 OpenAI/Anthropic API
- 磁带存放：各包 `test/fixtures/recordings/`
- 支持请求匹配、响应回放，覆盖 WebSocket 流量
- 使 provider 测试可以进 CI（零网络依赖、零费用、可重现）

## 与其他技术的关系

- **@opencode/ai**：主要被测对象（见 [../2.2_server/opencode-ai.md](../2.2_server/11_opencode-ai.md)）
- **Effect**：基于 Effect 实现（`@effect/platform-node-shared`）
- **bun test**：宿主测试运行器

## 相关源码

- `packages/http-recorder/src/`、各包 `test/fixtures/recordings/`
