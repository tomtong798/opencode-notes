# OpenCode 专家（Expert）实现方案

> 更新时间：2026-10-12
> 范围：`packages/opencode/src/agent/expert/`、`packages/core/src/plugin/skill/`、`packages/app/src/pages/abilities/experts-tab.tsx`

---

## 1. 背景与动机

专家（Expert）是面向"能力中心（Abilities Hub）"的一类特殊 agent：在普通 agent 的基础上打上
`expert: true` 标记，从而出现在桌面端 Experts 标签页中，供用户一键"召唤"（以该专家身份开启新会话）。

设计目标：

- **专家即 agent**：不做新的运行时，专家完全复用 agent 的 schema、权限模型和调用路径，`expert`
  标记只控制 UI 展示位置（对比 WorkBuddy 的独立专家运行时，这是刻意的简化）。
- **内置开箱即用**：随产品分发 5 个内置专家，覆盖常见职业角色。
- **用户可自建**：通过内置 `expert-creator` 技能，在会话中由模型生成一个 markdown 文件即完成创建。

## 2. 高层设计

```
┌─────────────────────────── 数据来源（三层合并） ───────────────────────────┐
│                                                                           │
│  ① 内置专家        ② 磁盘文件                ③ 全局配置                   │
│  Expert.builtins() ~/.config/opencode/     opencode.jsonc                 │
│  代码硬编码         agent/<name>.md         agent.<name> 键               │
│  expert:true        expert:true(前端声明)    任意字段覆盖                  │
│  native:true        native:false            disable:true 可禁用           │
│                                                                           │
│  └──────────── Agent.Service（/agent API 返回合并结果）────────┘          │
└───────────────────────────────────────────────────────────────────────────┘
                                    │
                     GET /agent（含 native / expert / hidden 字段）
                                    │
                                    ▼
              ┌─────────── 桌面端 ExpertsTab（abilities 页）───────────┐
              │  过滤: expert && !hidden                               │
              │  分区: native → "内置专家"；!native → "我创建的专家"    │
              │  卡片: 召唤按钮（TooltipV2）+ 内置Tag + mode/model Tag  │
              └────────────────────────────────────────────────────────┘
```

### 创建链路（用户自建专家）

```
用户点击"新建专家"
  → 新会话草稿，自动注入 expert-creator 技能指引
  → 模型按技能文档收集信息（名称/描述/mode/人设/工具策略）
  → 模型写 ~/.config/opencode/agent/<name>.md（frontmatter + 系统提示词）
  → （当前需重启应用）Agent.Service 重扫 → 出现在"我创建的专家"区
```

## 3. 模块分解

| 文件 | 职责 |
|------|------|
| `packages/opencode/src/agent/expert/index.ts` | 内置专家单一数据源 `DEFINITIONS`（5 个）：name/description/prompt/denyEdit；`builtins()` 产出内置 agent 记录（`native:true, expert:true, mode:"all"`，advisory 角色追加 `edit:* deny` 权限）；`catalog()` 供 `/agent/catalog` API |
| `packages/opencode/src/agent/agent.ts` | agent 合并器：`Object.assign(agents, Expert.builtins(...))` 后依次 merge 磁盘 md 与 jsonc 配置（`item.expert = value.expert ?? item.expert`） |
| `packages/schema/src/i18n-keys.ts` | `ABILITY_EXPERT_NAMES`（内置名单，schema 层单一来源）；`abilityExpertNameKey/DescriptionKey` 生成 i18n 键 |
| `packages/core/src/plugin/skill.ts` | 注册内置技能 `expert-creator`（新旧两套 Skill 服务均注册）；`ExpertCreatorContent` 在模块初始化时将 `{{BUILTIN_EXPERT_NAMES}}` 占位符替换为 `ABILITY_EXPERT_NAMES` 渲染结果 |
| `packages/core/src/plugin/skill/expert-creator.md` | 技能正文：信息收集清单、文件路径规范（`~/.config/opencode/agent/`）、frontmatter 格式、提示词结构、权限预设、同名冲突警告 |
| `packages/opencode/src/skill/index.ts` | 旧版 Skill 服务（会话/tool 实际消费方）的内置技能注册点，`customize-opencode` 与 `expert-creator` 均在此注册（`location:"<built-in>"`） |
| `packages/app/src/context/agents.tsx` | 前端 agent store：`/agent` API + 本地 config 合并；`add()` 创建时全量同名拒绝（内置/md/jsonc 一律拦截） |
| `packages/app/src/pages/abilities/experts-tab.tsx` | Experts 页 UI：按 `native` 分区渲染、搜索过滤、召唤按钮、卡片 Tag |

## 4. 核心流程

### 4.1 专家发现（服务端）

```
启动/重载
  → Agent.Service state 初始化
  → Expert.builtins() 注入 5 个内置（native:true, expert:true）
  → ConfigAgent.load 扫描 {agent,agents}/**/*.md
      目录: ~/.config/opencode（全局）+ 项目 .opencode
  → cfg.agent（jsonc）merge（可覆盖任意字段，disable 可禁用）
  → GET /agent 返回全量
```

关键合并语义：

- 同名时磁盘 md / jsonc **覆盖**内置字段（定制内置专家是合法用法）
- `native` 字段不随覆盖改变（内置永远是内置）
- `expert: true` 必须由 frontmatter 或 jsonc 显式声明

### 4.2 专家创建（expert-creator 技能）

技能文档约束模型执行以下流程：

1. **收集信息**：名称（kebab-case）、描述、mode（primary/subagent/all）、人设提示词、工具策略
2. **同名检查**：检查用户已有 agent，含内置 5 个（名单由 schema 动态注入，见 §5）；同名文件会静默覆盖而非新建
3. **写入文件**：全局 `~/.config/opencode/agent/<name>.md`（项目级 `.opencode/agent/` 仅在用户明确要求时）
4. **frontmatter 格式**：

```yaml
---
description: <做什么、何时用>
mode: all
expert: true
permission:            # 可选，advisory 类专家建议加
  edit:
    "*": deny
---
<系统提示词正文>
```

5. **提示词结构**：身份 → 工作方法 → 任务要求，全文 150-500 词
6. **收尾**：告知用户文件路径，说明需重载后生效

### 4.3 召唤（Summon）

```
点击卡片召唤按钮
  → 解析目标项目目录（默认项目 → 最近项目 → 第一个项目）
  → tabs.newDraft() 创建会话草稿
  → navigate(draftHref?agent=<name>)
  → 会话以该专家 agent 身份运行
```

无可用项目时 toast 报错，不产生半状态。

## 5. 关键设计决策

### 5.1 内置名单的单一来源

`ABILITY_EXPERT_NAMES`（`packages/schema/src/i18n-keys.ts`）被三处消费：

| 消费方 | 用途 |
|--------|------|
| `expert-creator` 技能内容 | 构建时 `replaceAll` 注入内置名单（占位符 `{{BUILTIN_EXPERT_NAMES}}`），模型创建时据此避免同名 |
| 桌面端 `experts-tab.tsx` | `BUILTIN_EXPERTS` 集合：内置专家名/描述走 i18n 翻译 |
| `packages/schema` i18n 键生成 | `abilities.experts.<name>.name/description` |

注意：`packages/opencode/src/agent/expert/index.ts` 的 `DEFINITIONS` 是运行时权威（含 prompt/权限），
其 name 列表与 schema 名单**手动同步**。目前两份列表靠约定一致，未来内置专家数量变化时需同步修改。

### 5.2 双 Skill 服务注册

仓库存在两套 skill 服务：

| 服务 | 消费方 | expert-creator 注册 |
|------|--------|--------------------|
| 旧版 `Skill.Service`（`packages/opencode/src/skill/index.ts`） | 会话/模型（`Skill.require`、system prompt 的 `<available_skills>`） | ✅（`location:"<built-in>"`） |
| 新版 `SkillV2.Service`（`packages/core/src/skill.ts` + plugin） | 渐进迁移中的新架构 | ✅（EmbeddedSource） |

会话实际走旧版，因此**两处都必须注册**。历史教训：只在 SkillV2 注册时，desktop 会话找不到该技能。

### 5.3 同名防护（双层）

| 层 | 机制 | 覆盖范围 |
|----|------|---------|
| UI 硬拦截 | `agents.add()` 同时查 jsonc（`configAgents()`）与 `/agent` 全量（`store.agents`），同名一律 toast 拒绝 | settings 表单创建路径 |
| 技能软约束 | `expert-creator.md` 要求创建前检查已有 agent + 内置名单，并说明同名 = 静默覆盖的后果 | 会话内模型创建路径 |

### 5.4 权限预设

`DEFINITIONS.denyEdit` 控制内置专家是否追加 `edit:* deny`（如法律顾问、行业研究等 advisory 角色）。
用户自建专家由技能文档指引在 frontmatter 中显式声明 deny 规则，且**只 deny 必须禁止的**（专家权限与用户全局权限 compose）。

## 6. 接口

### API

| 端点 | 说明 |
|------|------|
| `GET /agent` | 全量 agent 列表（含 `native`/`expert`/`hidden`/`disable`），UI 分区数据源 |
| `GET /agent/catalog` | `Expert.catalog()`，内置专家 id + i18n 描述键 |
| `GET /skill` | 技能列表（含内置 `expert-creator`，`location:"<built-in>"`） |
| `GET /global/config` | 全局配置（`add()` 同名校验数据源之一；注意不含磁盘 md agent） |

### 文件路径

| 路径 | 说明 |
|------|------|
| `~/.config/opencode/agent/<name>.md` | 用户自建专家（全局，推荐） |
| `.opencode/agent/<name>.md` | 项目级专家 |
| `~/.config/opencode/opencode.jsonc` → `agent.<name>` | 配置式 agent / 覆盖内置 |

### i18n 键

| 键 | 用途 |
|----|------|
| `abilities.experts.builtIn` / `abilities.experts.mine` | 分区标题（"内置专家"/"我创建的专家"） |
| `abilities.experts.<name>.name` / `.description` | 内置专家翻译（`ABILITY_EXPERT_NAMES` 生成） |
| `abilities.experts.summon` / `create` / `empty` / `noMatch` / `disabled` 等 | 页面文案 |

非英语 locale 缺键时回退英文（`{...base, ...locale}` 合并），无需 65 个文件全量补齐。

## 7. 非功能性

- **性能**：专家发现复用 agent 合并器，无独立 I/O；磁盘扫描仅在 InstanceState 初始化执行一次
- **一致性**：`native` 永不被配置覆盖，UI 分区不会因定制内置专家而错乱
- **可发现性**：同名覆盖若发生，内置专家行为变化在"内置专家"区可见（描述被替换），不是完全静默

## 8. 已知限制与改进方向

| # | 限制 | 影响 | 改进方向 |
|---|------|------|---------|
| 1 | 创建后需重载（agent 目录不在文件监视范围，InstanceState 缓存 TTL 无限） | 用户写完文件看不到，必须重启应用 | Watcher 订阅 agent 目录后 `disposeAllInstances`；或技能收尾自动触发 dispose API |
| 2 | 无格式校验：frontmatter 写错 → 静默不出现 | 排查成本高 | 轻量方案：技能收尾让模型 read 回读自检；重量方案：服务端校验 + 报错上报 |
| 3 | `DEFINITIONS` 与 `ABILITY_EXPERT_NAMES` 手动同步 | 漂移风险（skill 注入的名单 ≠ 实际内置） | 将 DEFINITIONS 的 name 列表下沉到 schema |
| 4 | 专家 = 单 prompt，无附属技能/多角色 | 表达力弱于竞品（见 §9） | 如需"专家团"再升级为目录包结构 |
| 5 | `getGlobal()` 不含磁盘 md agent | 前端 `configAgents()` 数据不全（`add()` 已用 `store.agents` 补齐，其他消费方注意） | 视需求统一 |

## 9. 与 WorkBuddy 的对比

> 依据：WorkBuddy Desktop v5.7.6 实机解包（`app.asar` + `~/.workbuddy` 用户目录），2026-10 分析。

### 9.1 文件位置对照

| 类型 | WorkBuddy | OpenCode |
|------|-----------|----------|
| 用户自建 | `~/.workbuddy/plugins/marketplaces/my-experts/plugins/<name>/` | `~/.config/opencode/agent/<name>.md` |
| 注册表/可见性 | `marketplace.json`（`register_expert.py` 写入后才可见） | 无注册表，目录扫描 + 重载 |
| 市场安装 | `~/.workbuddy/plugins/marketplaces/cb_teams_marketplace/plugins/<name>/` | 无市场（内置 + 自建） |
| 内置清单 | app.asar 内 `curated-experts.json`（100+，云端提示词） | 代码硬编码 5 个（本地完整） |
| 会话状态 | `expert-history.json` + sqlite | 会话系统自带 |

### 9.2 专家载体

| 维度 | WorkBuddy | OpenCode |
|------|-----------|----------|
| 载体 | **插件包**：`.codebuddy-plugin/plugin.json`（元数据/分类/双语展示字段）+ `agents/*.md`（多角色）+ `skills/*/SKILL.md`（附属技能） | **单 md**：frontmatter + prompt |
| 类型 | Agent 型 / Team 型（多角色协作） | 仅单角色 |
| 双语 | 所有展示字段强制 `{en, zh}` | 文件名即名，描述单语 |
| 附带能力 | 专家可捆绑专属 skills、bin 脚本 | 无 |
| 头像 | 强制（avatar-spec 生成） | 无 |

### 9.3 创建流程

| 维度 | WorkBuddy | OpenCode |
|------|-----------|----------|
| 创建入口 | 会话技能 `expert-manager`（交互/资料转化/修改三场景） | 会话技能 `expert-creator`（交互式） |
| 路径约束 | **SKILL.md 写死 + 明确禁止其他目录** | 文档指引（历史上有路径指错 bug，已修） |
| 校验 | `validate_expert.py`：拒绝 `[TODO]` 残留、字段缺失 | 无校验，全靠模型自觉 |
| 注册 | `register_expert.py` 写 marketplace.json，未注册不可见 | 写对路径 + 重载即出现 |
| 打包/分享 | `package_expert.py` + 分享链接 + 云同步 | 无 |
| 同名处理 | 改名需重建，name/目录名/agentName 禁改 | UI 硬拦截 + 技能软约束（同名 = 覆盖内置语义） |

### 9.4 评价与借鉴

**OpenCode 的优势**：

- 简单透明：一个 md 就是一个专家，开发者友好，无注册表/插件层
- 专家即 agent：无独立运行时，权限/工具/调用全复用
- 内置专家本地完整（提示词不依赖云端）

**WorkBuddy 更优、值得借鉴的点**：

| 点 | WorkBuddy 做法 | 借鉴建议 |
|----|---------------|---------|
| ① 校验前置 | validate 脚本挡格式错误/TODO 残留，注册时二次校验 | 优先级最高：技能收尾加"read 回读自检"（零成本），中期服务端校验 |
| ② 路径硬约束 | SKILL.md 写死 + 拒绝违规 | 我们路径已修正；可在技能中加"写完后向用户报告路径"强化 |
| ③ 可见性确定性 | 注册制：不注册不可见，无"写了但不出现"的静默失败 | 与 §8-1 重载问题一并考虑：dispose API 或注册表 |
| ④ 专家=包 | 专家可捆绑技能/团队/脚本 | 按需演进；若做"专家团"再引入目录结构 |
| ⑤ 双语展示字段 | 强制 `{en,zh}`，面向 C 端 | 面向开发者可不做；若出海再议 |

**结论**：OpenCode 当前方案在"简单、开发者友好"的定位下方向正确，核心差距是
**创建链路的确定性**（校验 + 可见性），而非功能丰富度。优先补 ①③ 两项即可对齐 WorkBuddy
的核心体验优势，暂不需要引入插件包架构。

---

## 附：WorkBuddy 实测路径速查（Windows）

```
C:\Users\<user>\.workbuddy\
├── experts\custom\<uid>\experts.json          # 自建专家 id 清单
├── plugins\
│   ├── marketplaces\
│   │   ├── my-experts\plugins\<name>\         # 自建专家包（SKILL.md 指定目录）
│   │   └── cb_teams_marketplace\plugins\<name>\  # 市场安装
│   ├── cache\<source>\<name>\<version>\       # 安装缓存
│   ├── installed_plugins.json                 # 安装清单（scope/version/时间）
│   └── configured-plugins.json
├── app\data\expert-history.json               # 最近使用
└── storage\user-<uid>\expert.json             # recents
内置清单: <安装目录>\resources\app.asar → resources\builtin-expert-recommendations\curated-experts.json
创建技能: app.asar → resources\plugins\workbuddy-builtin\skills\expert-manager\
```
