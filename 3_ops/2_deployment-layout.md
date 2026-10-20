# 部署目录与文件清单（Deployment Layout）

> 一句话：**一切路径由 XDG 基本目录派生（`util/src/global-roots.ts`）——data 存库与日志、config 存配置与密码、state 存服务注册与 TUI 状态、cache 存二进制；项目级 `.opencode/` 从 cwd 向上发现。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16）。
> [返回索引](README.md) | 姊妹篇：[log-locations.md](1_log-locations.md)

---

## 1. 路径总源：XDG 基本目录（`util/src/global-roots.ts`）

```ts
data   = XDG_DATA_HOME   || ~/.local/share    →  ~/.local/share/opencode
cache  = XDG_CACHE_HOME  || ~/.cache          →  ~/.cache/opencode
config = XDG_CONFIG_HOME || ~/.config         →  ~/.config/opencode
state  = XDG_STATE_HOME  || ~/.local/state    →  ~/.local/state/opencode
tmp    = os.tmpdir()                          →  /tmp/opencode
```

`global.ts` 在其上派生：`log = 2.3_data/log`、`bin = cache/bin`、`repos = 2.3_data/repos`。

### 各平台速查（Linux / macOS / Windows）

| 路径 | Linux | macOS | Windows |
|---|---|---|---|
| data | `~/.local/share/opencode` | 同左 | `%LOCALAPPDATA%\opencode` |
| cache | `~/.cache/opencode` | 同左 | `%LOCALAPPDATA%\opencode` |
| config | `~/.config/opencode` | 同左 | `%APPDATA%\opencode` |
| state | `~/.local/state/opencode` | 同左 | `%LOCALAPPDATA%\opencode` |
| tmp | `/tmp/opencode` | 同左 | `%TEMP%\opencode` |

**覆盖变量**：四个 `XDG_*_HOME`、`OPENCODE_CONFIG_DIR`（只覆盖配置目录）、`OPENCODE_TEST_HOME`（测试覆盖 home）。

---

## 2. 全局布局清单（装完 opencode 后磁盘上有什么）

```
~/.local/share/opencode/            ← data
├── opencode.db                     ← 主 SQLite（见 §3 按通道分库）
├── log/opencode-{channel}.log      ← 后端日志
└── repos/                          ← 克隆的 provider 包等

~/.config/opencode/                 ← config
├── opencode.json / opencode.jsonc  ← 全局用户配置
├── agents.json / agents.jsonc      ← agent 配置
└── service.json                    ← 服务配置（password/hostname/port/CORS/env）

~/.local/state/opencode/            ← state
├── service.json                    ← 服务注册文件（id/version/url/pid/password，0600）
│                                      （非标准通道为 service-{channel}.json）
└── {channel}/
    ├── tui/*.json                  ← TUI 持久状态（sessions/tabs…每 key 一文件）
    └── locks/                      ← 文件锁

~/.cache/opencode/                  ← cache
└── bin/                            ← 更新器等二进制

~/.opencode/bin/                    ← curl 安装脚本的安装位
└── opencode                        ← 平台二进制（+ opencode2 兼容别名）
```

### 系统级配置目录（config/discovery.ts，优先级最低）

| 平台 | 路径 |
|---|---|
| macOS | `/Library/Application Support/opencode/`（+ MDM：`/Library/Managed Preferences/{user}/ai.opencode.managed.plist`） |
| Linux | `/etc/opencode/` |
| Windows | `%ProgramData%\opencode\` |

### 兼容目录（discovery.ts:69-70）

`~/.claude/`（Claude Code 技能兼容）、`~/.agents/`（agent 目录）。

---

## 3. SQLite 数据库（`cli/src/database-path.ts`）

```ts
文件名 = OPENCODE_DB
      ?? (标准通道 latest/dev/beta/next/prod 或 OPENCODE_DISABLE_CHANNEL_DB=1
          ? "opencode.db"
          : `opencode-${channel}.db`)
路径 = {data}/{文件名}
```

- **按 channel 分库**：dev 玩家与 stable 用户互不污染数据
- `OPENCODE_DB=:memory:` 供 SDK/测试
- 库内表：`project`、`workspace`、`kv`（models.dev 缓存等）、`session_v2`、`session_message`、`session_inbox`、`event`…（全表见 [../1_architecture/overview.md](../1_architecture/1_overview.md) 附录 C）

---

## 4. 项目级 `.opencode/`

发现规则（`tui/src/util/config-directories.ts`）：**从 cwd 向上走到项目根**（以 `.git`/`.hg` 判定），沿途每个祖先目录的 `.opencode/` 都参与合并——monorepo 子目录可各自带配置。

```
{project}/.opencode/
├── opencode.json        ← 项目配置
├── agent/  command/  skill/  tool/   ← 项目级 agent/命令/技能/工具定义
├── plugin/              ← 项目插件
└── tui.json             ← TUI 行为配置（主题/键位/插件）
```

配置合并优先级（低→高）：系统目录 → 全局 → …→ 项目 `.opencode/` → 显式 `--config` → 内联 content → managed settings（MDM 最高）。详见 [../1_architecture/overview.md §2](../1_architecture/1_overview.md)。

---

## 5. 桌面端专属布局（Electron userData）

```
{userData}/（平台路径见 log-locations.md §3，按通道分 APP_ID）
├── logs/{timestamp}/           ← 桌面日志（main/renderer/netlog）
├── drafts.sqlite               ← 桌面草稿库（window state 另存 window-{id}.json）
├── cli/{stage}/                ← staging 的 CLI 二进制（按 version-arch 缓存，免重复展开 200MB）
├── opencode/service-local.json ← 隔离服务注册（OPENCODE_DESKTOP_ISOLATED_SERVER=1 时）
├── compile-cache/              ← V8 字节码缓存
└── Crashpad/                   ← 崩溃转储
```

桌面后端仍写全局 XDG 日志（两套并存的原因见 log-locations.md）。

---

## 6. 缓存与生命周期

| 数据 | 清理机制 |
|---|---|
| 后端日志 | 原地压缩（50MB→25MB，每小时） |
| 桌面日志目录 | 7 天删除 |
| `opencode uninstall` | **只删 cache 目录与服务注册文件**；2.3_data/config/state（会话、凭据、设置、历史）**保留** |
| SQLite / TUI 状态 / 草稿库 | **无自动 GC**——会话持续累积，磁盘敏感时手动清理 data 目录 |

---

## 7. 环境变量总表（路径相关）

| 变量 | 作用 |
|---|---|
| `XDG_{DATA,CACHE,CONFIG,STATE}_HOME` | 覆盖 XDG 基本目录 |
| `OPENCODE_CONFIG_DIR` | 只覆盖配置目录 |
| `OPENCODE_DB` | 覆盖数据库路径（可 `:memory:`） |
| `OPENCODE_DISABLE_CHANNEL_DB=1` | 强制单库 opencode.db |
| `OPENCODE_CHANNEL` | 通道（影响库名/日志名/注册文件名/TUI 状态目录） |
| `OPENCODE_MODELS_URL` / `OPENCODE_MODELS_PATH` / `OPENCODE_DISABLE_MODELS_FETCH` | models.dev 目录覆盖 |
| `OPENCODE_PASSWORD` / `OPENCODE_SERVER_PASSWORD` | 服务鉴权密码 |
| `OPENCODE_DESKTOP_ISOLATED_SERVER=1` | 桌面隔离服务（独立 state） |
| `OPENCODE_TEST_HOME` | 测试覆盖 home |

---

## 相关源码

- `packages/util/src/global-roots.ts`（XDG 解析，含 workerd 变体）、`global.ts`（派生路径）
- `packages/core/src/config/discovery.ts`（配置发现与优先级）
- `packages/cli/src/database-path.ts`（分库规则）、`services/service-config.ts`（注册/服务文件）
- `packages/tui/src/context/storage.tsx`（TUI 状态）、`util/config-directories.ts`（项目目录发现）
- `packages/desktop/src/main/lifecycle/configure.ts`（userData 设置）
