# PPT 模板生成插件（ppt-agent）

> 状态：P0 + P1 全量已实施（2026-10-22，P1 = B2 样式借鉴 + B1 占位符填充 + ppt_edit）；P2-P3 待做
> 目标仓库：`/home/donghe/gitlab/opencode`（opencode fork，分支 user/donghe/dev）
> 对标：腾讯 WorkBuddy PPT 能力（2026-09 升级版）

## 功能全貌（当前交付形态）

插件形态：内置插件（`packages/opencode/src/plugin/ppt/`），4 个工具 + 系统提示注入，主干触碰仅 `plugin/index.ts` 2 行注册，全程本地渲染、零外部服务依赖。

```
用户需求（主题 / Word 文档 / 参考 PPT）
   ↓ SKILL.md 引导（补问场景→读材料→出大纲确认→选模板→生成→续改）
ppt_read_doc          读 Word（标题层级→markdown）
   ↓
大纲 JSON（7 种布局，Zod 硬校验：title/section/bullets/two_column/chart/quote/closing，3-40 页）
   ↓
ppt_generate ── 三级模板解析 ──────────────────────
   ├─ 内置主题（business-blue / tech-dark / education-warm）
   ├─ B2 样式借鉴：任意 .pptx → 提取 clrScheme+fontScheme → 动态主题（明暗自适应）
   └─ B1 占位符填充：含 {{key}} 的模板 → 保留版式/母版/配色只换内容
        （页数不足自动克隆模板页；未用槽位填空；run 拆分归一化）
   ↓
可编辑 .pptx + sidecar（.outline.json，唯一事实源 + 编辑历史）
   ↓
ppt_edit（update/insert/remove/reorder/change_template/set_title → 重渲染）
```

验证状态：27 个测试全绿（P0 9 + P1 B2 9 + B1 9）、typecheck 零错误、结构校验通过（rels/Content_Types/sldIdLst 完整性）、全链路端到端验证（Word+参考PPT→新PPT 样式确认 `style borrowed: true`；克隆压力测试：3 页模板渲染 5 页大纲全填充）。

## 与市面智能体 PPT 能力对比（2026-10-22）

| 能力 | 本插件 | WorkBuddy | 豆包 | Kimi/WPS/讯飞 | Gamma |
|---|---|---|---|---|---|
| 主题→大纲→生成主链路 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 生成前补问场景/对象 | ✅ | ✅ | 部分 | 部分 | ❌ |
| 大纲确认后渲染 | ✅ | ✅ | ✅ | ✅ | ✅ |
| Word 文档导入 | ✅ 本地解析 | ✅ 云端 | ✅ 云端 | ✅ 云端 | ✅ |
| 结构化校验防丢内容 | ✅ **Zod 硬校验** | 未知 | ❌ 偶发丢页 | 未知 | 未知 |
| 内置模板 | 3 套 | 多场景库 | ~20 | 20万+/300+/200+ | 主题系统 |
| **公司模板保留版式只换内容** | ✅ B1（{{key}} 约定） | ✅ | ❌ | 部分 | ❌ |
| **任意 PPT 借鉴配色字体** | ✅ B2（无前置要求） | ✅ | ❌ | ❌ | ❌ |
| 页数扩展（模板页克隆） | ✅ | ✅ | ✅ | ✅ | N/A |
| 图表页 + 结论 + 演讲备注 | ✅ bar/line/pie | ✅ | ✅ 基础 | ✅ | ✅ |
| 对话式续改 | ✅ ops+历史 | ✅ 框选级 | ✅ | ✅ | ✅ |
| 输出可编辑 .pptx | ✅ | ✅ | ⚠️ 付费墙 | ⚠️ 部分 VIP | ⚠️ 变形 |
| 本地/离线/免费 | ✅ **全本地** | ❌ 云端 | ❌ 云端 | ❌ 云端 | ❌ 云端 |
| 智能配图/AI 生图 | ❌ | ✅ | ✅ | ✅ | ✅ |
| 前端预览/元素级精改 | ❌ | ✅ | ✅ | ✅ | ✅ |
| 表格布局 | ❌ | ✅ | ✅ | ✅ | ✅ |

### 结论

**已达 WorkBuddy 核心能力线**：其最被称道的两个卖点——"提交公司模板保留主题色/字体层级/结构只换内容"（B1）与"参考任意 PPT 整套借鉴配色版式"（B2）——均已落地；且三点结构性占优：

1. **无付费墙/无云端依赖**：豆包的 AiPPT 套壳付费墙、各家的云端渲染依赖都不存在，断网可用
2. **确定性渲染**：Zod 校验 + 确定性模板映射，不会出现豆包式的丢内容问题
3. **模板来源翻转**：市面产品靠平台模板库（数量竞赛），本插件让用户自己的任意 .pptx 都是模板（B2 零要求即用，B1 加占位符即全保真）——对企业场景（有 VI 规范的公司）比公共模板库更实用

真实差距（按价值排序，对应 P2/P3）：智能配图（市面标配，需引入生图/图库依赖）、前端预览与框选精改（产品级 UI）、表格布局与更多图表类型（小改动）、内置模板数量（纯内容工作）。

## P1(B1) 实施记录（2026-10-22）

已交付并通过验证（B1 测试 9/9 绿、全量回归 27/27 绿、typecheck 零错误、结构校验 OK）：

| 文件 | 内容 |
|---|---|
| `plugin/ppt/template-fill/manifest.ts` | 模板扫描：逐页提取 `{{key}}` 占位符（run 归一化后匹配）+ 角色推断（title/section/bullets/.../unknown）+ `isPlaceholderTemplate` 判定 |
| `plugin/ppt/template-fill/fill.ts` | 填充引擎：run 归一化合并（`</a:t>` 与 `<a:t>` 边界清除，首 run 格式胜出）→ XML 替换（XML 实体转义）→ `leftoverTokens` 残留校验；`loadZip`/`writeZip` |
| `plugin/ppt/template-fill/clone.ts` | 幻灯片克隆：slide xml + `_rels`（含 notesSlide 复制与重指向）+ `[Content_Types]` override + sldIdLst/rels 注册（编号自增避撞） |
| `plugin/ppt/render/template-render.ts` | B1 主流程：outline→模板页映射（同角色优先，不足克隆原型页）→ 槽位填充（未用槽位填空串）→ sldIdLst 重建（未引用模板页剔除） |
| `plugin/ppt/tools/generate.ts` | 三级模板解析：内置 id → 占位符模板（B1，`fill:` 前缀）→ 样式参考（B2，`file:` 前缀）；输出未填充槽位提示 |
| `plugin/ppt/tools/edit.ts` | `ppt_edit` 工具：6 种 ops（update_slide/insert_slide/remove_slide[3页下限]/reorder/change_template/set_title）+ sidecar 历史记录 + 三模式重渲染（fill:/file:/内置） |
| `plugin/ppt/skill/ppt-agent.md` | 工作流 v3：三种模板模式选择指引 + 占位符命名约定（`{{title}}/{{item1..7}}/{{left_heading}}/...`）+ 续改优先 ppt_edit |
| `plugin/ppt/index.ts` | 注册 4 工具（+ppt_edit） |
| `test/ppt-plugin-b1.test.ts` | 9 测试：manifest 扫描/角色推断、跨 run 替换、实体转义、克隆（rels/notes/Content_Types/sldIdLst）、B1 端到端、generate 自动检测、ppt_edit（含 fill 模式/3 页下限） |

关键设计点：

- **三级解析顺序**是接口契约：同一 `.pptx` 参数先探占位符（B1 保真填充），否则走样式借鉴（B2）——模型侧无需指定模式
- **未用槽位填空串**而非保留 token 或报错（WorkBuddy 式体验）；真正无法映射的槽位才在输出中提示
- **克隆只增不删**：未引用的模板页从 sldIdLst 剔除但 part 保留，避免 ZIP 内部引用悬空
- **编辑历史**：ops 以 JSON 字符串数组追加进 sidecar `history`，保留审计轨迹

## P1(B2) 实施记录（2026-10-22）

已交付并通过验证（P1 测试 9/9 绿 + P0 回归 18/18 绿、typecheck 零错误、端到端全链路：Word + 参考 PPT → 新 PPT 样式借鉴确认 `style borrowed: true`）：

| 文件 | 内容 |
|---|---|
| `plugin/ppt/docx/read.ts` | JSZip 解包 .docx → 抽 `<w:t>` 文本 + pStyle 标题层级 → markdown 渲染 |
| `plugin/ppt/tools/read-doc.ts` | `ppt_read_doc` 工具（read 工具不支持 Office 格式的补位） |
| `plugin/ppt/template-fill/extract-theme.ts` | B2 样式提取：theme1.xml 的 clrScheme（dk1/lt1/dk2/lt2/accent1-4）+ fontScheme（latin/Hans）→ 动态 Theme；lt1 亮度判定明暗主题翻转文字色 |
| `plugin/ppt/render/render.ts` | 新增 `renderWithTheme`/`renderWithThemeToFile`（动态 Theme 渲染路径） |
| `plugin/ppt/tools/generate.ts` | template 参数扩展：内置 id 或 .pptx 路径（`file:` 前缀识别）；sidecar 记 `file:<绝对路径>` 供续改重提取 |
| `plugin/ppt/tools/templates.ts` | 新增 `template_file` 参数：预览参考文件的色板/字体/派生样式 |
| `plugin/ppt/skill/ppt-agent.md` | 工作流 v2：读参考材料（Word→ppt_read_doc / 参考 PPT→预览确认）→ 大纲（忠实原文、数据转 chart）→ 选模板（参考路径直用） |
| `test/ppt-plugin-p1.test.ts` | 9 测试：docx 解析（标题层级/实体/非法文件）、B2 提取、file 模板生成端到端、sidecar 重提取 |
| package.json | jszip@3.10.2 |

关键设计点：

- **docx 解析是刻意子集**：段落 + 标题层级够用大纲生成，表格/脚注拍平为文本；不引外部转换器（pandoc/python），与 B2 共用 jszip 依赖
- **B2 只读不写**：仅解析参考文件 theme1.xml，不触碰其母版/版式/内容；参考文件路径进 sidecar，续改时重提取（用户改模板后重新生成即生效）
- **明暗自适应**：lt1 亮度 >0.6 判定为浅色主题，深色主题时 primary 取 accent1、正文色取 lt2，避免深底黑字
- **端到端体验**：「这是总结.docx，照着公司模板.pptx 做汇报」→ ppt_read_doc 读内容 → 大纲确认 → ppt_generate(template: 公司模板.pptx) → 内容来自 Word、配色字体来自参考 PPT 的可编辑 .pptx

## P0 实施记录（2026-10-21）

已交付并通过验证（测试 9/9 绿、typecheck 零错误、端到端生成 7 页含图表/备注/sidecar 的 .pptx）：

| 文件 | 内容 |
|---|---|
| `packages/opencode/src/plugin/ppt/outline.ts` | 7 布局 Zod schema + Sidecar schema（LLM 契约） |
| `packages/opencode/src/plugin/ppt/render/themes.ts` | 3 内置主题（business-blue / tech-dark / education-warm） |
| `packages/opencode/src/plugin/ppt/render/layouts.ts` | 7 种布局渲染器（含页码游标、footer、notes） |
| `packages/opencode/src/plugin/ppt/render/render.ts` | outline → .pptx + sidecar 主流程 |
| `packages/opencode/src/plugin/ppt/tools/templates.ts` | `ppt_templates` 工具 |
| `packages/opencode/src/plugin/ppt/tools/generate.ts` | `ppt_generate` 工具（Zod 校验 + sidecar 写入） |
| `packages/opencode/src/plugin/ppt/skill/ppt-agent.md` | WorkBuddy 式工作流引导 |
| `packages/opencode/src/plugin/ppt/index.ts` | 插件入口（工具注册 + system.transform 注入） |
| `packages/opencode/src/plugin/index.ts` | internalPlugins() 注册（+2 行，无条件加载，无开关） |
| `packages/opencode/test/ppt-plugin.test.ts` | 9 个测试 |
| `packages/opencode/package.json` | pptxgenjs@4.0.1 依赖 |

实施中的实现决策（与方案的差异）：

- **无配置开关**：插件随内置插件始终加载（用户决策），不提供 `ppt.enabled` 配置项；唯一退出方式是全局 `--disable-default-plugins`（禁用所有内置插件）
- **SKILL 注入方式**：不用 `skills.paths` 磁盘扫描，改用 `experimental.chat.system.transform` hook + `.md` 编译期内联 import（memory 插件先例）。原因：单二进制 compile 构建下磁盘 SKILL.md 不会被打包，system.transform 是唯一编译安全的注入路径
- **权限默认**：插件工具无匹配规则时默认 `ask`（安全默认）；用户可在 opencode.json 配置放行
- **页码游标**：pptxgenjs 类型未暴露 slides 集合，用 `RenderState.pageNumber` 跨布局共享计数
- **既有测试失败说明**：`test/config` 的 93 个失败在基线（未含本次改动）同样存在，为仓库既有问题，与 PPT 插件无关

## 〇、关键决策（2026-10-21 补充）

### D1：做成内置插件（memory 插件模式）

采用 fork 内 memory 插件的内置方式：在 `packages/opencode/src/plugin/index.ts` 的 `internalPlugins()` 数组注册，插件代码自包含于 `packages/opencode/src/plugin/ppt/`，仅依赖 `@opencode-ai/plugin` 公开 API。

对主干触碰面收敛为 3 处（内置的必要代价，与 memory 插件同款）：

1. `plugin/index.ts` 加 import + 一行 `plugins.push(pptHooks)`（可按 `cfg.ppt?.enabled` 开关，默认开启）
2. 新目录 `packages/opencode/src/plugin/ppt/`（自包含）
3. 可选：config 增加 `ppt?: { enabled?: boolean }` 开关

`internalPlugins()` 数组是设计好的扩展点，往扩展点加条目符合开闭原则；渲染引擎/schema/skill 全部封在插件目录内，不碰 tools/registry/skill 现有文件。

目录结构相应调整为（不再使用 `plugins/ppt-agent/` 独立目录）：

```
packages/opencode/src/plugin/ppt/
├── index.ts              # 插件入口：注册 3 个工具 + config hook 注入 skill
├── outline.ts            # 大纲 Zod schema
├── tools/
│   ├── templates.ts      # ppt_templates
│   ├── generate.ts       # ppt_generate
│   └── edit.ts           # ppt_edit
├── render/
│   ├── themes.ts         # 内置主题
│   ├── layouts.ts        # 7 种布局渲染器（pptxgenjs）
│   └── render.ts         # outline → .pptx
├── template-fill/        # P1：用户 .pptx 模板填充
│   ├── manifest.ts
│   ├── fill.ts
│   └── clone.ts
└── skill/
    └── ppt-agent/
        └── SKILL.md      # 工作流引导（config hook 注入 skills.paths，或按 memory 模式经 system context 注入）
```

### D2：不定义 agent

PPT 生成是用户当前会话内的工具调用流（skill 引导 → 调工具 → 继续对话），无后台子任务、无独立上下文需求，**工具 + 技能即可，不需要定义 agent**。对照：memory 插件定义 3 个 agent 是因为它有 checkpoint-writer 后台写入和 dream/distill 自主工作流。

已验证的扩展路径（留作 P3 可选）：config hook 可动态注册 agent（memory 先例：`agents["dream"] = { mode: "subagent", prompt, permission }`）和命令（`commands["dream"] = { template, subtask: false }`）。若将来要做"PPT 精修 agent"（独立上下文逐页打磨）或 `/ppt` 斜杠命令，走此路径。

### D3：不涉及前端修改

- 插件注册的工具进入 V1 工具注册表后，TUI/Web 对 tool part 按"工具名 + 状态 + 输出文本"**通用渲染**，前端无需认识具体工具；工具返回 `{ title, output, metadata }` 原样展示
- SKILL.md 是模型侧机制，发现与注入全在后端，前端技能列表页读的是后端数据
- 生成的 .pptx 落在工作目录，用户自行打开

唯一需动前端的场景是 P3 可选的"WorkBuddy 式页面预览/框选续改"（产品级 UI），明确在范围外。

## 一、背景与调研结论

### 1.1 需求

在 opencode fork（opencode 智能体）中以**插件方式**新增"模板生成 PPT"能力，要求：

- 尽量与原生功能解耦，遵循开闭原则（不改动 `packages/` 下任何代码）
- 对标 WorkBuddy：一句话出初稿、生成前补问、大纲确认、模板选择、原文件续改、输出可编辑 .pptx

### 1.2 市场调研摘要（2025-2026）

主流产品均采用「主题/文档 → LLM 生成大纲 → 选模板 → 填充渲染」范式：

| 阵营 | 代表 | 特点 |
|---|---|---|
| Native 派（输出可编辑 .pptx） | Microsoft Copilot、WPS AI 灵犀、Kimi PPT、讯飞智文、通义、百度文库 | 大纲 → 模板库 → 占位符填充 |
| Web-first 派 | Gamma、Tome、Beautiful.ai | 卡片/块布局引擎渲染网页，导出 PPTX 变形、丢失可编辑性 |

**WorkBuddy**（2026-09 升级）：对话直说"生成 PPT"或 `/tencent-pptx` 技能调用；支持提交公司/学校模板，保留主题色、字体层级、封面/目录/流程页结构，只替换内容；"人机双写"续改（框选元素下指令，改一处不影响其他页）；输出标准 .pptx。其能力以 **skill + 工具调用**实现，与本 fork 架构同构。

**豆包**（反面教材）：大纲生成后跳转第三方 AiPPT.cn 编辑/下载，付费墙、体验割裂——渲染引擎外包第三方之祸。本方案全本地渲染，天然避开。

技术路线选型：

- **PptxGenJS**（纯 TS，Bun 原生，无 Python 运行时）——内置主题渲染
- **JSZip**（XML 占位符替换）——用户模板填充，不动 theme/master XML 即保留原样式
- 否决 Marp/Slidev（导出为图片不可编辑）、python-pptx（引入 Python 运行时依赖）
- Gamma 核心启示：**把内容约束为有限布局类型（7 种），LLM 输出才可靠**

### 1.3 fork 现状探查结论

- 工具系统双轨：V2（`packages/core/src/tool/`，Effect Schema）与 V1（`packages/opencode/src/tool/`，Zod）
- V1 插件机制（`@opencode-ai/plugin`）完全满足需求且**零侵入**：
  - 插件 = 返回 Hooks 的 async 函数；`tool()`（`packages/plugin/src/tool.ts`）用 Zod 定义参数
  - `loader.ts` 按 `plugin: ["file:<路径>"]` 加载 file 插件，跳过版本兼容门禁
  - `config` hook 可在启动时改写 Config——用于注入插件自带的 skill 目录到 `skills.paths`
- skill catalog 中已有 `pptx` 条目（来自 anthropics/skills，仅 LLM 指导，非结构化生成能力）

## 二、总体架构

```
用户: "帮我做一份Q3汇报PPT"
        │
        ▼
┌─ SKILL.md (ppt-agent 技能，由插件 config hook 注入) ─┐
│  引导 LLM: 补问受众/场景/页数 → 产出大纲 JSON        │
│  → 调 ppt_templates 选模板 → 调 ppt_generate        │
│  → 续改调 ppt_edit                                  │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─ 插件 tools 层 ────────────────────────────────────┐
│  ppt_templates  ppt_generate  ppt_edit             │
└──────┬─────────────────────────────────────────────┘
       ▼
┌─ 渲染引擎（插件内，纯 TS，无外部 shell 依赖）────────┐
│  路径A: code themes (P0)    路径B: 模板填充 (P1)    │
│  pptxgenjs 逐布局渲染        JSZip 解包 .pptx       │
│  3套主题×7种布局             {{占位符}} XML 替换     │
│  + sidecar outline.json     + 幻灯片克隆            │
└─────────────────────────────────────────────────────┘
```

**Sidecar 模式（续改核心设计）**：`ppt_generate` 输出 `.pptx` 同时写 `<name>.outline.json`（大纲 + 模板引用 + 编辑历史）。所有续改作用于 outline JSON 后整体重渲染。避开 pptxgenjs 只写不读的限制，天然支持"只改一处不影响其他页"。

## 三、目录结构（已按 D1 调整，见"〇、关键决策"）

```
packages/opencode/src/plugin/ppt/          ← fork 内置插件目录，自包含
├── index.ts              # 插件入口：注册 3 个工具 + config hook
├── outline.ts            # 大纲 Zod schema（LLM 契约，渲染引擎输入）
├── tools/
│   ├── templates.ts      # ppt_templates 工具
│   ├── generate.ts       # ppt_generate 工具
│   └── edit.ts           # ppt_edit 工具
├── render/
│   ├── themes.ts         # 内置主题定义（business/tech/education）
│   ├── layouts.ts        # 7 种布局渲染器（pptxgenjs）
│   └── render.ts         # outline JSON → .pptx 主流程
├── template-fill/        # P1
│   ├── manifest.ts       # 解析用户 .pptx → 占位符清单
│   ├── fill.ts           # XML 占位符替换 + 重打包
│   └── clone.ts          # 幻灯片克隆
└── skill/
    └── ppt-agent/
        └── SKILL.md      # WorkBuddy 式工作流引导
```

用户模板目录（运行时约定）：

```
~/.config/opencode/ppt-templates/    # 用户放自定义 .pptx 模板
```

## 四、工具面设计（3 个工具）

### 4.1 `ppt_templates` — 列出可用模板

```ts
args: {
  filter: z.enum(["all", "builtin", "user"]).default("all"),
}
```

返回：内置主题列表（名称/风格描述/预览色板）+ 用户模板列表（扫描模板目录，含 manifest 摘要：每页布局类型、可用占位符）。

### 4.2 `ppt_generate` — 大纲 → .pptx（核心）

```ts
args: {
  outline: OutlineSchema,        // 结构化大纲
  template: z.string(),          // 内置主题 id 或用户模板文件名
  output: z.string(),            // 输出路径，相对 session.directory 解析
  title: z.string().optional(),  // 供 sidecar 命名
}
```

返回：`{ output: "已生成 xxx.pptx（12页）: 路径", metadata: { path, slides, sidecar } }`

### 4.3 `ppt_edit` — 续改（对标 WorkBuddy"原文件续改"）

```ts
args: {
  sidecar: z.string(),           // .outline.json 路径（生成时返回给 LLM）
  ops: z.array(z.discriminatedUnion("op", [
    z.object({ op: z.literal("update_slide"), index: z.number(), patch: SlidePatch }),
    z.object({ op: z.literal("split_slide"), index: z.number() }),        // "这页拆两页"
    z.object({ op: z.literal("insert_slide"), after: z.number(), slide: Slide }),
    z.object({ op: z.literal("remove_slide"), index: z.number() }),
    z.object({ op: z.literal("reorder"), from: z.number(), to: z.number() }),
    z.object({ op: z.literal("change_template"), template: z.string() }), // 整套换风格
  ])),
  regenerate: z.boolean().default(true),
}
```

执行：ops 应用到 sidecar JSON → 重新渲染 .pptx。`change_template` 对应 WorkBuddy"整套借鉴配色版式"。

## 五、大纲 JSON Schema（LLM 契约）

7 种受限布局（Gamma 启示：有限布局类型让 LLM 输出可靠）：

```ts
// src/outline.ts
export const Slide = z.discriminatedUnion("layout", [
  z.object({
    layout: z.literal("title"),
    title: z.string(), subtitle: z.string().optional(),
    meta: z.string().optional(),            // "汇报人 | 日期"
  }),
  z.object({ layout: z.literal("section"), title: z.string() }),
  z.object({
    layout: z.literal("bullets"),
    title: z.string(),
    items: z.array(z.object({ text: z.string(), level: z.number().default(0) })).max(7),
    notes: z.string().optional(),           // 演讲备注
  }),
  z.object({
    layout: z.literal("two_column"),
    title: z.string(),
    left: z.object({ heading: z.string(), items: z.string[] }),
    right: z.object({ heading: z.string(), items: z.string[] }),
  }),
  z.object({                                // P1: 图表
    layout: z.literal("chart"),
    title: z.string(),
    chart: z.object({
      type: z.enum(["bar", "line", "pie"]),
      categories: z.string[],
      series: z.array(z.object({ name: z.string(), values: z.number[] })),
    }),
    insight: z.string().optional(),         // 图旁结论
  }),
  z.object({ layout: z.literal("quote"), text: z.string(), source: z.string().optional() }),
  z.object({ layout: z.literal("closing"), title: z.string(), contact: z.string().optional() }),
])

export const Outline = z.object({
  title: z.string(),
  subtitle: z.string().optional(),
  slides: z.array(Slide).min(3).max(40),
})
```

工具 `args.outline` 用此 schema；Zod 校验失败返回精确错误路径，LLM 自动修正——结构化工具调用优于纯 skill 的关键点。

## 六、渲染引擎两条路径

### 6.1 路径 A：代码主题（P0）

`themes.ts` 定义 3 套主题（纯数据：主色/辅色/背景/标题字体/正文字体/强调色），`layouts.ts` 每种布局一个渲染函数：

```ts
// layouts.ts 骨架（pptxgenjs API）
import PptxGenJS from "pptxgenjs"

export function renderBullets(pptx: PptxGenJS, slide: BulletsSlide, theme: Theme) {
  const s = pptx.addSlide({ masterName: theme.master.content })
  s.addText(slide.title, {
    x: 0.6, y: 0.4, w: 8.8, h: 0.8,
    fontSize: 28, bold: true, color: theme.primary, fontFace: theme.fontHeading,
  })
  s.addText(
    slide.items.map((it) => ({
      text: it.text,
      options: { bullet: { indent: 12 }, indentLevel: it.level, fontSize: it.level === 0 ? 16 : 14 },
    })),
    { x: 0.8, y: 1.5, w: 8.4, h: 3.8, color: theme.text, fontFace: theme.fontBody, valign: "top" },
  )
  if (slide.notes) s.addNotes(slide.notes)
}
```

主题经 `pptx.defineSlideMaster` 定义（背景色、页码、装饰线条），master + 主题常量保证多页一致性（WorkBuddy 强调点）。

### 6.2 路径 B：用户模板填充（P1）——"保留样式只换内容"

pptx = ZIP + XML。纯 TS 实现 python-pptx 式占位符填充：

1. **manifest 生成**：`JSZip.loadAsync(template)` → 遍历 `ppt/slides/slideN.xml` → 正则抽取 `<a:t>...</a:t>` 中 `{{name}}` 占位符 → manifest（每页：布局类型推断、占位符名列表）
2. **填充**：XML 字符串替换。**关键**：占位符可能被 PowerPoint 拆成多 run（`<a:t>{{ti</a:t><a:t>tle}}</a:t>`），需先做段内相邻 run 合并归一化再匹配；表格 `<a:tc>` 同样处理
3. **重打包**：JSZip 压缩写 .pptx
4. **幻灯片克隆**（大纲页数 > 模板页数）：复制 slideN.xml + `[Content_Types].xml` 加 override + `ppt/_rels/presentation.xml.rels` 加关系 + `presentation.xml` 的 `sldIdLst` 插入 sldId——克隆页 100% 保留原样式

此路径**不动 theme1.xml / slideMaster**，主题色、字体层级、母版动画全保留。

### 6.3 Sidecar 格式

```json
{
  "version": 1,
  "template": "business-blue",
  "outline": { "...": "Outline JSON" },
  "history": [{ "at": "...", "ops": ["..."] }]
}
```

## 七、SKILL.md 工作流（WorkBuddy 式引导）

```markdown
---
name: ppt-agent
description: 模板化生成可编辑 PPT。当用户要求制作PPT、汇报演示文稿、slides时使用。
---

# 工作流（严格按序执行）

1. **补问**（信息不足时，只问必要问题，已明确的不重复确认）：
   - 汇报对象与场景（评审/答辩/路演/政务）
   - 页数期望、必须包含的章节
   - 是否有参考材料（文件路径）或指定模板
2. **大纲**：根据回答产出 Outline JSON（7 种布局，bullets 每页 ≤7 条），
   展示给用户确认。传入文档时：压缩长文、提取关键结论、数据转 chart 布局。
3. **选模板**：调 ppt_templates 展示内置主题和用户模板，让用户选。
4. **生成**：调 ppt_generate，把 sidecar 路径告知用户。
5. **续改**：用户说"第三页拆两页/换配色/加一页数据"时，调 ppt_edit 传 ops，
   不要从头重新生成。
6. 输出 .pptx 为标准格式，可在 PowerPoint/WPS 继续编辑。
```

插件入口通过 config hook 注入：

```ts
// src/index.ts 骨架
import { Plugin } from "@opencode-ai/plugin"
import { tool } from "@opencode-ai/plugin/tool"
import path from "node:path"

export const PptAgentPlugin: Plugin = async (_ctx) => {
  const skillDir = path.resolve(import.meta.dir, "../skill")
  return {
    config: async (input) => {
      input.skills = { ...input.skills, paths: [...(input.skills?.paths ?? []), skillDir] }
    },
    tool: {
      ppt_templates: templatesTool,
      ppt_generate: generateTool,
      ppt_edit: editTool,
    },
  }
}
```

> 注：config hook 的 `skills.paths` 字段名需在实现时对照 `packages/opencode/src/config` 实际结构微调。降级方案：README 指引用户把 SKILL.md 软链到 `~/.agents/skills/`。

## 八、安装与配置（用户视角）

内置插件（D1）模式下用户零配置：

- 插件随 fork 构建自动加载（默认开启；`opencode.json` 可设 `"ppt": { "enabled": false }` 关闭）
- 无需 `plugin: ["file:..."]` 配置，无需 bun install（依赖进 `packages/opencode/package.json`）

放自定义模板：`cp 公司模板.pptx ~/.config/opencode/ppt-templates/`

## 九、测试方案

- 插件目录内独立 `bun test`（不违反仓库"根目录不跑测试"规则）
- 单测：outline schema 校验（合法/非法用例）、每种布局渲染出非空 .pptx、sidecar 读写、edit ops 的 JSON 变换
- 集成验证：生成真实 .pptx → `unzip -l` 检查结构 → 若环境有 python-pptx 则回读校验页数/文本（仅测试用，非运行时依赖）
- 手工验收：生成结果在 WPS/PowerPoint 打开确认可编辑

## 十、分期实施

| 阶段 | 内容 | 工作量估计 |
|---|---|---|
| **P0** | 插件骨架 + outline schema + 3 内置主题 + 7 布局 pptxgenjs 渲染 + `ppt_generate`/`ppt_templates` + SKILL.md + 测试 | ✅ 已完成（2026-10-21） |
| **P1** | `ppt_edit`（ops + sidecar 重渲染）+ chart 布局 + 用户模板 manifest/填充/克隆 + B2 样式借鉴 + `ppt_read_doc` | ✅ 已完成（2026-10-22） |
| **P2** | `{{img:...}}` 图片占位符（media + rels 注入）、多模板索引目录、表格布局 | 1-2 天 |
| **P3** | 预览体系（见"预览架构"节）：outline→HTML 预览渲染器 + 前端通用工具附件渲染组件；可选 `/ppt` 命令与精修 agent | 2-3 天 |

## 十一、风险与边界

| 雃关 | 应对 |
|---|---|
| V1 插件 API 将来被 V2 取代 | 业务逻辑（schema/渲染/填充）与 tool 薄壳分离，迁移只重写壳 |
| pptxgenjs 在 Bun 下的兼容性 | 纯 JS（vendor 自带 ZIP 写入），Bun 兼容良好；P0 第一天先冒烟验证 |
| 占位符 run 拆分导致漏替换 | 归一化合并 + 校验替换后无残留 `{{`，残留则报具体页码 |
| LLM 产出超页数/超条目 | Zod `.max()` 硬约束，错误信息带回让 LLM 自修 |
| 大文件/多图内存 | JSZip 流式处理；40 页上限兜底 |

## 十二、对标 WorkBuddy 能力覆盖表

| WorkBuddy 能力 | 本方案对应实现 | 阶段 |
|---|---|---|
| 一句话出初稿（先补问对象/场景/目标） | SKILL.md 引导 LLM 补问 + `ppt_generate` | P0 |
| 大纲确认后再生成 | 大纲 JSON 是 LLM 产出物，用户确认后传入工具 | P0 |
| 内置多风格模板 | 代码定义主题（配色/字体/版式），零资产文件 | P0 |
| 提交公司/学校模板，保留主题色、字体层级，只替换内容 | 用户 .pptx 模板 + XML 占位符替换（JSZip） | P1 |
| 数据转图表、生成演讲备注 | outline schema 内置 chart 布局 + `addNotes` | P1 |
| 原文件续改（"这页拆两页""表格转图"） | sidecar outline JSON 为唯一事实源，编辑 = 改 JSON 再重渲染 | P1 |
| 输出标准 .pptx 可继续编辑 | PptxGenJS / XML 直写，本地渲染无第三方依赖 | P0 |

**关键优势**：WorkBuddy/豆包依赖云端渲染引擎（豆包还外包 AiPPT 致付费墙），本方案全部在插件进程内完成，无网络依赖、无付费墙、对主干零改动。

## 十三、预览架构（2026-10-21 补充，P3）

### 事实核查结论

- 当前方案 100% 后端：工具在 agent 进程内渲染 .pptx，前端只显示工具文本输出与 metadata
- 浏览器**不能原生渲染 .pptx**（ZIP+XML 二进制）；WorkBuddy 的预览是自研引擎把 PPT 重绘为 Web 页面，不是"打开文件"——任何预览方案都需要一个"可渲染表示"
- 管道已存在一半：插件工具返回的 `attachments: [{type:"file", mime, url}]` 经 registry（registry.ts:171-177）透传 → session 层转 FilePart（tools.ts:427-462）→ Web 端 session-message.ts:330-353 已提取 `attachments` 字段；图片类 data URL 前端可直接显示
- 缺口：packages/app 的 timeline tool row **目前没有消费 attachments 的渲染组件**（仅 prompt-input 用户上传侧有），需要一小块新前端代码，但可做成通用能力

### 分层方案（按成本递增）

| 层 | 做法 | 改动位置 | 效果 |
|---|---|---|---|
| L1 零前端 | `ppt_generate` 返回 .pptx 路径，desktop 用户 `file://` 用系统 WPS/PowerPoint 打开 | 无 | 够用但不算预览 |
| **L2 同源 HTML 预览（推荐核心）** | 预览不从 .pptx 渲染，而是从 sidecar outline + 主题渲染 HTML 预览页（全 TS，插件内实现）：`render.ts` 暴露 `renderPptx(outline, theme)` 与 `renderPreviewHtml(outline, theme)` 双 renderer；预览经 `metadata.previewUrl` 或 html attachment 返回 | 仅后端（插件内） | 预览与 .pptx 同源同风格，改稿重渲染时两视图同步——这是 WorkBuddy 预览的真正精髓（同一份数据的两个视图） |
| L3 通用前端预览组件 | timeline tool row 新增附件渲染：image 附件直接展示、html 附件 iframe 容器翻页 | 前端一个**通用**组件（非 ppt 专属） | WorkBuddy 式体验；图表/PDF/HTML 等所有生成类插件受益 |

### 设计要点

- L2 是解耦关键：预览渲染器与 pptx 渲染器共享 outline schema 与主题定义，二者永远不会漂移；后续"框选续改"也可在 HTML 预览上叠加交互（点击元素 → 映射回 outline 节点 → 生成 ppt_edit ops）
- L3 做成通用"工具产物预览"组件而非 ppt 专属组件，符合 D3 的前端零侵入原则——前端只认识 file/html/image 附件，不认识 ppt
- 图表等复杂元素在 HTML 预览中用 CSS/SVG 近似还原即可，目标是"确认内容与版式"而非像素级一致
