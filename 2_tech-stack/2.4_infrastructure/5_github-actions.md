# GitHub Actions

## 是什么

GitHub 原生 CI/CD：仓库 `.github/workflows/` 里的 YAML 定义事件触发的自动化流水线（build/test/publish）。

## 业界类比

- GitLab CI / Jenkins 的 GitHub 原生版
- 开源项目的默认选择（公开仓库免费额度）

## 在 OpenCode 中的角色

- **全部 CI/CD**：类型检查、lint、测试矩阵、桌面端发布（`publish.yml`，badge 指向它）、云侧 SST 部署
- 提交历史可见优化痕迹：`chore(ci): speed up and stabilize the test workflow`
- 工作流运行在**预烘容器**里（见 [docker.md](6_docker.md)）：`job.container` 指定 `packages/containers` 构建的镜像

## 与其他技术的关系

- **Docker（containers 包）**：CI 环境镜像
- **Turborepo**：CI 内的 typecheck/test 经 turbo 编排（并发 `--concurrency=3`）
- **SST / Wrangler**：部署步骤的执行器
- **Husky / oxlint**：本地侧对应物（提交前），CI 侧跑 `bun run check` 全量

## 相关源码

- `.github/workflows/`、`packages/containers/script/build.ts`
