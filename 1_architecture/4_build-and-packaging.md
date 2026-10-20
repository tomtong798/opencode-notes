# 构建与打包详解（Build & Packaging）

> 一句话：**源码到产物的三条路——CLI 双路径编译（Bun 原生 + Node SEA）、Electron 三入口打包、Web 双部署（Cloudflare + 内嵌 CLI）；发布由 publish.yml 八段流水线串联。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16）。
> [返回索引](README.md)

---

## 1. 全景图

```
源码（monorepo，多数包源码直跑）
│
├─ CLI 产物 ────────── 路径 A：Bun.build({compile:true}) → 12 平台原生二进制
│                      路径 B：Vite bundle + node --build-sea → 5 平台 SEA 二进制
│                      发布形态：@opencode/cli（伞包）+ @opencode/cli-{target}（平台包）
│
├─ 桌面产物 ────────── electron-vite（main/preload/renderer 三入口）
│                      electron-builder → dmg/zip、nsis、AppImage/deb/rpm
│                      内嵌 CLI 二进制作为 sidecar 服务
│
└─ Web 产物 ────────── vite build（+PWA）→ 双去向：
                       a) SST 部署到 Cloudflare（app.opencode.ai）
                       b) brotli 压缩成 AppArchive 内嵌进 CLI 二进制
```

**关键认知**：多数包（tui、session-ui、gui-extensions、httpapi-codegen 等）**无构建步骤、源码直跑**（Bun 原生执行 TS）；真正有 `build` 的约 23 个包，集中在"要发布到 npm"或"要出最终产物"的场景。

---

## 2. CLI：双路径编译

### 路径 A：Bun 原生二进制（`packages/cli/script/build.ts`）

- `Bun.build({ compile: true })` 产出原生可执行文件
- **12 个目标**：linux/darwin/win32 × arm64/x64，加 `-baseline`（无 AVX2）与 `-musl`（Alpine）变体
- 内嵌三样东西：
  - **Web UI**：先 `vite build` 构建 `packages/app`，全部产物 brotli 压缩（quality 11）编码为 `AppArchive` 二进制 blob，经 `virtual:opencode-app-assets` 虚拟模块打进二进制——**CLI 的 server 运行时直接服务完整 Web UI**（`script/app-assets.ts`）
  - **原生绑定**：node-pty、parcel-watcher、FFF 二进制（Bun 插件内联）
  - Bun 运行时本体（`BUN_COMPILE_RELEASE=bun-v1.4.2` 从 GitHub releases 拉取）
- 产物：`dist/cli-{target}/bin/opencode`

### 路径 B：Node SEA（`packages/cli/script/build-node.ts`）

- 存在意义：**没有 Bun 运行时也能跑**的兜底产物（部分受限环境）
- 两步：Vite 把 `src/node/index.ts` 打成 `dist-node/opencode.mjs` → `node --build-sea` 生成单可执行文件（Node 26.4 SEA）
- **5 个目标**（无 macOS x64——Node SEA 不支持）：linux-arm64/x64、darwin-arm64、windows-arm64/x64
- `vite.node.config.ts` 的 nodePrelude 在运行时把 SEA 资产解包到用户级缓存目录

### npm 发布形态（`packages/cli/script/publish.ts`）

```
@opencode/cli-linux-x64（等 12+5 个平台包）── 只含二进制
        ↑ optionalDependencies
@opencode/cli（伞包）── bin/opencode.cjs 跨平台 wrapper：
                        运行时遍历 node_modules 找到本平台的二进制并执行
```

- 伞包还带 `postinstall.mjs`（安装时校验平台包）与 `bin/opencode2.cjs`（向后兼容别名）
- curl 安装脚本（根 `install`）：从 npm 拉 `@opencode/cli-{target}`，解压到 `~/.opencode/bin/opencode`，写 shell 配置加 PATH

---

## 3. 桌面：Electron 三入口

### 构建（`packages/desktop`）

- `build`：`electron-vite build`（`electron.vite.config.ts` 三入口）
  - **main**：`src/main/index.ts`（打包 Effect/drizzle，externalize node-pty）
  - **preload**：`src/preload/index.ts`（CJS——Electron sandbox 要求）
  - **renderer**：复用 `@opencode/app/vite` 共享配置（**与 Web 端同一份 Vite 配置**，SolidJS+Tailwind+PWA）
- `prebuild`：`scripts/prepare.ts` 拷贝图标、metainfo，并**把 CLI 二进制拷进 resources/**
- `package`：electron-builder（`electron-builder.config.ts`）
  - macOS：`dmg`+`zip`，hardened runtime + 公证
  - Windows：`nsis`，Azure Trusted Signing
  - Linux：`AppImage`+`deb`+`rpm`
  - appId 按通道区分：`ai.opencode.desktop` / `.dev` / `.beta`

### Sidecar 概念的现状

sidecar 演进现状：**sidecar = 完整 CLI 二进制**（`resources/opencode-cli`），不是独立的 JS sidecar 文件。

```
desktop 主进程
├─ DesktopCli（desktop-cli.ts）：解析内置 CLI 二进制，staging 到 userData/cli/{version}-{arch}/
├─ sidecarProbe（sidecar-probe.ts）：启动前先探测已存在的兼容后台服务（Service.discover）
├─ BackgroundService（background-service.ts）：spawn CLI 为后台进程：opencode serve --service
└─ SidecarCredentials（sidecar-credentials.ts）：主进程持有 HTTP Basic 密码并注入请求头
                                          ——渲染进程永远见不到密码（CORS-simple GET 直连）
```

### 更新器

electron-updater；更新清单 `latest.yml`（win）/ `latest-mac.yml` / `latest-linux{,-arm64}.yml`，发布到 `https://opencode.ai/update/api/{channel}/desktop/opencode/`（`services/update` 提供）。

---

## 4. Web：一份构建，两个去处

- 构建：`packages/app` 的 `vite build`（vite-plugin-pwa + vite-plugin-solid + @tailwindcss/vite；共享配置 `vite.js` 导出为 `@opencode/app/vite`）
- 去处 a：`.github/workflows/deploy.yml` → `bun sst deploy --stage={dev|beta|production}` → Cloudflare（app.opencode.ai）
- 去处 b：被 CLI 构建的 `app-assets.ts` 内嵌（见 §2）——所以 **CLI 单二进制也自带完整 Web UI**

---

## 5. 发布流水线（`.github/workflows/publish.yml`）

触发：push 到 `ci`/`dev`/`beta` 或手动。八段：

```
version（script/version.ts 定版本）
  → build-cli（并行：Bun 12 目标 + Node SEA 5 目标矩阵）
  → sign-cli-macos（Developer ID 签名）
  → build-node-app-archive（Web UI 归档，供 SEA）
  → build-node-cli（SEA 矩阵）
  → sign-cli-windows（Azure Trusted Signing）
  → build-electron（6 平台矩阵：下载 CLI → prepare → electron-vite → electron-builder → 上传 latest*.yml）
  → publish（script/publish.ts：
             逐包发 npm：schema→codemode→theme→ai→util→protocol→client→cli→plugin→plugin-browser→core→simulation→server→sdk→ui
             桌面产物发布到更新服务器；latest 通道提交 WinGet；打 git tag）
```

其他工作流：`deploy.yml`（SST）、`containers.yml`（CI 镜像到 GHCR）、`publish-vscode.yml`。

---

## 6. 哪些包有构建步骤（turbo 视角）

**有 build（23 个）**：cli（compile）、desktop（electron-vite）、app（vite）、web/posts（astro）、console/stats 应用（vite）、storybook、以及发布到 npm 的库（schema/protocol/2.2_server/core/2.1_client/sdk/ai/plugin 等——多为 tsc 或自写 script/build.ts 产出 dist）

**无 build、源码直跑**：tui、session-ui、gui-extensions、function、httpapi-codegen、merman、latex、plugin-browser（部分）……

`turbo.json`：`build` 依赖 `transit`（把 workspace 依赖折叠进测试哈希的 no-op 任务）；typecheck 有显式依赖链（cli/2.2_server/tui/sdk/simulation → `@opencode/core#typecheck`）。

---

## 7. 设计取舍

- **为什么 CLI 内嵌 Web UI**：单二进制分发 `opencode` 即含完整浏览器界面（`--port` 起服务家人人可用），不用单独装桌面端
- **为什么双编译路径**：Bun 二进制覆盖主流 + 性能；Node SEA 兜底"Bun 不可用"的受限环境
- **为什么平台子包 + 伞包**：npm optionalDependencies 是 JS 生态分发原生二进制的标准解（esbuild/rust-analyzer 同款），避免单包塞全部平台
- **为什么桌面端不直接嵌 JS 而嵌 CLI 二进制**：桌面与 CLI 共享同一个版本化、签名过的服务实现——一套服务端代码三种宿主（CLI/桌面/SDK）

## 相关篇

- [overview.md](1_overview.md) / [package-map.md](2_package-map.md)
- [../2_tech-stack/2.4_infrastructure/github-actions.md](../2_tech-stack/2.4_infrastructure/5_github-actions.md)
- [../2_tech-stack/2.1_client/electron.md](../2_tech-stack/2.1_client/8_electron.md)
- [v1-to-v2-evolution.md](6_v1-to-v2-evolution.md)（"Build legacy CLI" 里的 packages/opencode 是旧 CLI 残留）
