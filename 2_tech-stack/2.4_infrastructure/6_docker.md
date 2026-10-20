# Docker

## 是什么

容器化标准：把运行环境（OS 依赖、工具链）打进可复制的镜像。

## 业界类比

- "可运输的操作系统快照"
- CI 环境一致性的通用解

## 在 OpenCode 中的角色

- **仅用于 CI 预烘镜像**（`packages/containers`），产品本身**不做容器部署**——opencode 是装在用户电脑上的本地工具：

| 镜像 | 内容 |
|---|---|
| `base` | Ubuntu 24.04 + 构建工具 |
| `bun-node` | + Bun + Node 24 |
| `rust` | + Rust stable |
| `tauri-linux` | + Tauri 依赖 |
| `publish` | + Docker CLI + AUR 工具 |

- Linux-only（`job.container`）、多架构（amd64 + arm64）
- 目的：把大依赖烘进镜像，CI 每次免下载提速

## 与其他技术的关系

- **GitHub Actions**：镜像的消费者（`job.container`）
- **Kubernetes/编排**：仓库**没有**——无容器化服务部署

## 相关源码

- `packages/containers/`（`script/build.ts`）
