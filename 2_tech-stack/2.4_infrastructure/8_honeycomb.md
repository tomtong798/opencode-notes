# Honeycomb

## 是什么

**高基数分析型观测平台**：为"任意维度组合的查询"优化（宽事件模型，一行 = 一个事件的全部维度），吃 OTLP 标准 数据。

## 业界类比

- 对比 Prometheus/Grafana（指标聚合型）：Honeycomb 存**宽事件**，任意维度切片下钻不预聚合
- "把日志、指标、追踪统一成事件表"的代表

## 在 OpenCode 中的角色

- opencode 遥测的**观测后端**：客户端/服务埋点经 OTel 导出至此
- stats 体系有 Honeycomb 回填任务（`packages/stats/core`）
- 基础设施以 provider 集成进 SST（`infra/monitoring.ts`，有独立 stage `vimtor`）

## 与其他技术的关系

- **OpenTelemetry**：数据来源标准（见 [opentelemetry.md](7_opentelemetry.md)）
- **SST**：资源编排
- 仓库**没有** Prometheus/Grafana/Sentry——观测栈就这一条链

## 相关源码

- `infra/monitoring.ts`、`packages/stats/core/`
