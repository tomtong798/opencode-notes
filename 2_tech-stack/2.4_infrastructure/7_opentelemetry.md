# OpenTelemetry（OTel）

## 是什么

**厂商中立的遥测标准**：统一的 traces（追踪）/ metrics（指标）/ logs（日志）SDK 与协议（OTLP）。埋点一次，后端可换。

## 业界类比

- 监控界的"USB 接口"——应用侧统一插头，后端随便换（Honeycomb/Datadog/Jaeger…）
- CNCF 第三大活跃项目（仅次于 Kubernetes）

## 在 OpenCode 中的角色

- 客户端与服务进程**内置埋点**：CLI 入口注入 Observability 层（OTEL 导出配置）
- Effect 生态衔接：`@effect/opentelemetry`（catalog 依赖）——Effect 的 span/trace 自动导出
- 导出目标：Honeycomb（见 [honeycomb.md](8_honeycomb.md)）

## 与其他技术的关系

- **Honeycomb**：OTLP 数据的下游分析平台
- **Effect**：`@effect/opentelemetry` 把效应执行映射为 trace span
- **util 包**：OTel SDK 的组装处（`packages/util/src` 含 OpenTelemetry instrumentation）

## 相关源码

- `packages/cli/src/index.ts`（Observability 层）
- `packages/util/src/`（OTel 组装）、根 `package.json` catalog（`@effect/opentelemetry`）
