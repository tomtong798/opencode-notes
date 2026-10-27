# XDG Base Directory

## 是什么

**XDG** = X Desktop Group，制定 Linux 桌面环境通用标准的组织（今 freedesktop.org 前身）；其规范统称 XDG 规范。日常说的 "XDG" 几乎都指其中最常用的 **XDG Base Directory Specification（基本目录规范）**：定义一套标准化的用户文件存放路径，让应用知道配置/缓存/数据该放哪，避免各自在 `~` 下乱建隐藏目录（`~/.mozilla`、`~/.vimrc` 时代的教训）。

### 四个核心环境变量

| 变量 | 用途 | 默认值 |
|---|---|---|
| `XDG_CONFIG_HOME` | 用户配置文件 | `~/.config` |
| `XDG_CACHE_HOME` | 缓存 | `~/.cache` |
| `XDG_DATA_HOME` | 应用数据 | `~/.local/share` |
| `XDG_STATE_HOME` | 运行状态（日志、历史，较新加入） | `~/.local/state` |

另有 `XDG_RUNTIME_DIR`（运行时文件如 socket，`/run/user/<uid>`，必须属主专有）。

规范语义要点：**变量是覆盖，默认值是回退**——读 `XDG_*_HOME`，未设则用默认路径；相对路径值按未设置处理。

### XDG 家族其他成员（了解即可）

- **XDG Desktop Entry**：`.desktop` 文件标准（应用菜单/启动器的"名片"）
- **XDG Desktop Portal**：Flatpak/Snap 沙箱应用安全访问系统资源的跨桌面接口
- **XDG User Directories**：`~/Documents`、`~/Downloads` 等用户目录的本地化命名

## 业界类比

- "Linux 家目录的分区表"——应用各归其位，备份 `~/.config` 即备份全部配置
- 类比容器体系的 volume 分类：config ≈ ConfigMap、cache ≈ emptyDir、data ≈ PersistentVolume

## 在 OpenCode 中的角色

**opencode 的全局路径体系完全建立在 XDG 之上**——单一派生源在 `util/src/global-roots.ts:5-8`：

```
data   = XDG_DATA_HOME   || ~/.local/share    →  ~/.local/share/opencode
cache  = XDG_CACHE_HOME  || ~/.cache          →  ~/.cache/opencode
config = XDG_CONFIG_HOME || ~/.config         →  ~/.config/opencode
state  = XDG_STATE_HOME  || ~/.local/state    →  ~/.local/state/opencode
```

四目录的分工（详表见 [../../3_ops/2_deployment-layout.md](../../3_ops/2_deployment-layout.md)）：

| 目录 | 存什么 | 典型内容 |
|---|---|---|
| data | 数据库与日志 | SQLite 库（按 channel 分）、事件日志 |
| config | 配置与凭证 | `opencode.json(c)`、服务密码 |
| state | 易失状态 | **服务注册文件**（url/pid/password，0600——发现契约）、TUI 状态 |
| cache | 可丢弃重生物 | staging 的 CLI 二进制 |

三个 XDG 在关键机制里出场：

1. **服务发现/ensure 协议**（`client/src/effect/service.ts`）：注册文件落在 `XDG_STATE_HOME` 下——桌面/TUI 多客户端竞争启动后靠它互相发现
2. **桌面测试隔离**（`desktop/src/main/lifecycle/configure.ts:52-73` `createTestRoot`）：E2E 测试整套重定向四个 `XDG_*_HOME` 到临时根，沙箱里跑真实应用
3. **isolated 服务模式**（`background-service.ts:41`）：`XDG_STATE_HOME` 重定向到 userData，让开发模式服务不污染全局注册文件

## 与其他技术的关系

- **Electron userData**：桌面端的应用数据走 Electron 自己的 `app.getPath("userData")`（`appData/<APP_ID>`），**不**归 XDG 管——opencode 是"桌面壳 Electron 一套 + 后端服务 XDG 一套"的双体系（两套日志并存的原因见 [../../3_ops/1_log-locations.md](../../3_ops/1_log-locations.md)）
- **daemon-client 架构**（[../2.6_architecture-styles/2_daemon-client.md](../2.6_architecture-styles/2_daemon-client.md)）：注册文件放 state 而非 cache/data，是"易失但可发现"语义的选择
- **configure.ts 测试根**：XDG 覆盖语义（环境变量优先）使测试隔离零侵入——不改代码只改变量

## 相关源码

- `packages/util/src/global-roots.ts`（5-8 行四变量派生，全仓唯一权威）
- `packages/desktop/src/main/lifecycle/configure.ts:52-73`（测试根 XDG 重定向）
- `packages/client/src/effect/service.ts:16-20`（注册文件即发现契约）
- 权威规范：[XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)（freedesktop.org）

---

*素材来源：freedesktop.org 规范 + 元宝 AI 解析（2026-10-20，经仓库实测校准——原文的四变量表缺 `XDG_STATE_HOME`，opencode 重度使用它，已补）。*
