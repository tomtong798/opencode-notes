# Distill 工作流扫描与识别过程

本文档记录一次 `distill` 扫描的完整过程：如何从轨迹数据库中发现重复工作流，
并决定是否将其打包为可复用的 skill 或 subagent。

## 数据源

| 数据源 | 位置 | 用途 |
|---|---|---|
| 轨迹数据库 | `opencode.db`（本环境位于 `/root/.local/share/opencode/opencode.db`） | 会话标题、消息、工具调用 payload |
| 记忆文件 | `.opencode/memory/` | 跨会话沉淀的规则与决策 |
| 已有资产 | skills / agents / commands 清单 | 避免重复造轮子 |

关键表：`session`（含 `title`）、`message`、`part`（工具调用 JSON 存在 `data` 字段）。

---

## 第 1 步：拉取会话标题，归一化聚类

从 `session.title` 拉取全部标题。原始标题存在大量同义变体和 fork 噪音，需先归并：

- 剥离 `(fork #N)`、`(child)`、`New session - <时间戳>` 等后缀
- 合并同义词：「今天 / 今日」「查询 / 查询今天」

归一化后浮现出两个高频簇：

### 簇 A：城市 + 天气查询

```
天津今日天气查询 ×14      望京今日天气查询 ×6      北京望京天气查询 ×4
今天三亚天气 / 青岛今日天气 / 上海 / 北京 …
```

### 簇 B：城市 + 150万买房

```
南京150万购房咨询 ×6        三亚150万可以买什么房子
石家庄150万购房选择          在天津150万可以买什么房子
上海150万购房选择            北京150万左右房价分区查询
```

---

## 第 2 步：翻工具调用 payload，看真实执行路径

标题只是入口，真正的模式藏在 `part.data` 里实际发出的请求中。这一步揭示了两类浪费。

### 天气类：全部命中 `wttr.in`，但 format 字符串有 15+ 种临时写法

```
wttr.in/Tianjin?format=3
wttr.in/Tianjin?format=j1
wttr.in/Tianjin?format=%C+%t+%w+%h&lang=zh
wttr.in/Wangjing,Beijing?format=4&m&lang=zh
wttr.in/Wangjing+Beijing?…          ← 用 + 号分隔城区（错误写法）
wttr.in/Wangjing?format=j1          ← 漏掉省份，结果可能不准确
```

**结论**：同一个目标（查天气）被写成了 15+ 种不一致的调用形式，说明缺少一个统一的规范。

### 买房类：渠道可靠性差异巨大

按 URL 出现次数统计：

```
lianjia.com/   ×128   ← 唯一稳定可用，每次都拿到数据
anjuke.com/sale/ ×33  ← 触发 antibot 验证码，webfetch 被拦截
google.com     ×20    ← 搜索返回空 / 无用内容
```

**结论**：存在一条稳定的执行路径（链家/房天下），但每次都重复踩「安居客反爬」「搜索引擎无效」的坑。

---

## 第 3 步：盘点已有资产，避免重复造轮子

检查现有 skills / agents / commands 清单，确认：

- 天气查询、房源搜索两个流程**没有任何现存封装** → 需要新建
- `dream` / `checkpoint-writer` / `distill` 等重复标题 → 是记忆插件自身的自动化，跳过
- GitHub 相关流程 → 已被 `duplicate-pr` / `triage` 等 agent 覆盖，跳过

---

## 第 4 步：按「频率 × 可复用性」打分，只落地高置信项

| 候选 | 频率 | 可复用性 | 结论 |
|---|---|---|---|
| 中文天气查询 | 极高频 | 强（单一 `wttr.in` 调用） | ✅ 打包为 skill |
| 中文房源搜索 | 高频 | 强（渠道选择 + 响应格式） | ✅ 打包为 skill |
| 代码库目录梳理 | 1 次 | 弱 | ❌ 跳过（结果已进 MEMORY.md） |
| 北京旅行规划 | 1 次 | 弱 | ❌ 跳过 |
| 天气多日预报 | 仅出现为追问 | 待观察 | ⏸ 证据不足，暂不打包 |
| 房源细分筛选 | 仅出现为追问 | 待观察 | ⏸ 证据不足，暂不打包 |

---

## 产出

最终落地两个 skill，把反复踩坑的结论固化下来：

### `.opencode/skills/chinese-weather-lookup/SKILL.md`

- 统一 `wttr.in` 调用：单行 `format=%C+%t+%w+%h&lang=zh`
- JSON `j1` 作为结构化回退
- 城区用逗号分隔，如 `Wangjing,Beijing`
- 避免搜索引擎的无效轮次

### `.opencode/skills/chinese-real-estate-search/SKILL.md`

- 固化渠道可靠性：链家/房天下可靠，安居客被反爬拦截，搜索引擎无效
- 城市子域名模式（如 `nj.lianjia.com`）
- 三档响应格式：市中心老破小 / 近郊次新三房 / 远郊大户型四房

---

## 判定准则小结

一个重复工作流是否值得打包为 skill，需同时满足：

1. **频率高**：同类会话在轨迹库中多次出现
2. **可复用性强**：存在一条稳定的、可标准化的执行路径
3. **尚未被封装**：不在已有 skills / agents / commands 中
4. **有明确规范价值**：过去执行中存在不一致或反复踩坑，固化成规范能省轮次

满足 1–3 但第 4 点证据不足的，标记为「待观察」，等再次出现再扩展。
