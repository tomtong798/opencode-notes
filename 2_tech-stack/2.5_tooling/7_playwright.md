# Playwright

## 是什么

微软出品的**跨浏览器自动化测试框架**：驱动真实浏览器（Chromium/Firefox/WebKit）做 e2e 与组件测试，自带 trace viewer、并行执行、自动等待。

## 业界类比

- Cypress 的"更底座中立（三引擎）、更强多标签页/多上下文"继任者
- 浏览器自动化的事实标准（借用其 API 的还有 MCP browser 工具）

## 在 OpenCode 中的角色

- **Web 端 e2e**：`packages/app/e2e`（写测试前必读 `.opencode/skills/app-e2e-tests`——记录了 CI 里慢/flaky 的教训）
- **组件测试**：Storybook 组件的 Playwright 交互测试（`.opencode/skills/component-tests`）
- 桌面/终端 UI 不用它——TUI 用 OpenTUI 渲染器测试（`tui-tests`）

## 与其他技术的关系

- **Storybook**：组件测试的宿主（Playwright 驱动 story，见 [storybook.md](8_storybook.md)）
- **bun test**：单测的执行器（Playwright 只管浏览器侧）
- **OpenTUI 测试**：TUI 侧对应物（见 [../2.1_client/opentui.md](../2.1_client/3_opentui.md)）

## 相关源码

- `packages/app/e2e/`、`packages/session-ui/`（组件测试）、`.opencode/skills/app-e2e-tests`、`.opencode/skills/component-tests`
