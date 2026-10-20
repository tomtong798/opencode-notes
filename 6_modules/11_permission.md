# 权限子系统详解（声明式规则 + 评估器 + 询问协议 + 记住决策的四面体）

> 一句话：**Permission 是一个"声明式规则（action/effect/resource 三元组）→ 通配评估器（findLast 末位胜出 + deny 前置一票否决）→ 询问协议（进程内 pending Map + ephemeral 总线事件）→ saved 持久化（always 决策落 SQLite 后升级为 allow 规则）"的四层子系统；托管策略（ManagedPolicy）是另一条正交的目录过滤轴，不参与运行时评估。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：`packages/schema/src/permission.ts` + `packages/core/src/permission.ts`（无独立 spec 文件）
> [返回 README](README.md) | 前置 [9_tool.md](9_tool.md) §6 权限接缝（assert 时机 + decline 隧道，本篇不重复） | 关联 [6_context.md](6_context.md)（agent 规则来源字段） | [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)（TUI 侧 blocker/hydrate）

---

## 0. 模块地图

```
packages/
├─ schema/src/
│   ├─ permission.ts        ★ 全部公共形状：ID/Request/Reply/Event/Effect/Rule/Ruleset（67 行）
│   ├─ permission-saved.ts  saved 决策的 ID/Info（schema 侧）
│   ├─ config/policy.ts     ConfigPolicy：托管策略语句形状（13 行）
│   ├─ config.ts:55-57      配置文件顶层 permissions 字段（Ruleset）
│   └─ session-event.ts:122-126 session.permissions 事件（会话级规则覆盖）
├─ core/src/
│   ├─ permission.ts        ★ Effect 服务：evaluate/merge/assert/ask/reply/pending（347 行）
│   ├─ permission/
│   │   ├─ saved.ts         ★ PermissionSaved 服务：SQLite CRUD（88 行）
│   │   └─ sql.ts           drizzle 表定义 permission（20 行）
│   ├─ managed-policy.ts    ★ Console 下发的组织策略状态 + decision()（75 行）
│   ├─ config/plugin/policy.ts  托管策略落为目录过滤 transform（providers/servers/skills/tools）
│   ├─ config/plugin/agent.ts:84-126  配置 permissions 展开并 push 进各 agent
│   ├─ v1/config/permission.ts  V1 兼容形状（42 行）
│   └─ util/wildcard.ts     通配匹配器（14 行，全仓共用）
└─ 消费端
    ├─ tui/src/mini/runtime.ts:267-274        onPermissionReply → sdk.permission.reply
    ├─ tui/src/mini/stream-v2.transport.ts:1339-1346  asked/replied 事件 → 渲染
    ├─ cli/src/run/noninteractive.ts:158-183  无人值守：自动 reject + 反馈
    └─ 各工具叶子里 assert（见 9_tool.md §4，本篇不重复）
```

无 `permission/AGENTS.md`；设计意图散见 `permission.ts` 内注释（:248-252 defect tunnel、:289-296 批量反馈）与根 AGENTS.md。

动作名目录（action 的实际取值，来自各工具 assert 调用点实测，规则书写时按此对齐）：

| action | assert 位置 | resources 语义 | save 惯例 |
|---|---|---|---|
| `write` / `edit` / `patch` | write.ts:78 / edit.ts:275 / patch.ts:196 | 目标文件绝对路径 | 该文件路径（always 记住路径） |
| `shell` | shell.ts:138 | **逐条解析后的子命令**（`a && b` 分两条） | 命令串 |
| `skill` | skill.ts:51 | `skill.id` | agent.id |
| `question` | question.ts | `["*"]` | 无 save（问了就问了） |
| `webfetch` 等 | websearch.ts:49 | URL/域名 | 视工具 |
| `mcp-resource` | mcp-resource.ts:36,65 | 资源 URI | — |
| `file-access` | file-access.ts:135,154 | 路径 | — |
| `skill`（目录过滤） | skill.ts:33-34（直接调 `Permission.evaluate`，非 assert） | skill.id | — |

## 1. 心智模型

四个部件构成一个"判定流"：

```
工具 execute 内（叶子，先于一切副作用）
  permission.assert({ sessionID, action, resources, save?, metadata?, source? })
    │
    ▼ ① 规则收集（configured, permission.ts:158-163）
    │    agent.permissions（缺省 = deny-all 兜底 :19）
    │    ++ session.permissions（session.permissions 事件投影）
    │
    ▼ ② deny 前置一票否决（denied(), :165-167）
    │    任一 resource 命中 effect==="deny" 的规则 → 直接 BlockedError
    │    （此时尚未混入 saved 规则 → deny 永远压过 saved allow）
    │
    ▼ ③ 全量评估（evaluateInput, :173-189）
    │    all = configured ++ savedRules(仅 allow)
    │    每个 resource 独立 evaluate()：findLast 末位胜出，无命中默认 "ask"
    │    多 resource 结果合并："ask" 传染（任一 ask 即整体 ask, :178）
    │    plugin hook "permission.evaluate" 可改写 effect/message（:179-187）
    │
    ├─ deny   → BlockedError（typed error, :237-243）
    ├─ allow  → 直接放行（:245）
    └─ ask ──→ ④ 询问协议
         create(): pending Map 登记 + Deferred + publish permission.asked（ephemeral）
           → UI 三端应答 permission.reply
              ├─ once   → Deferred.succeed → assert 返回（本次放行）
              ├─ always → saved.add + Deferred.succeed + 连锁复核其余 pending（:310-322）
              └─ reject → Deferred.fail(DeclinedError|CorrectedError)
                          → assert 出口 catchTag → Effect.die（decline 隧道，引 9_tool §6）
```

关键认知：**规则是声明式数据（Ruleset 数组），评估是纯函数（`evaluate` 可独立 import 使用——skill.ts:33-34 就在目录过滤里直接调它），询问是带进程内状态的协议（pending Map + Deferred），saved 是把"always"应答物化回规则数据的闭环。**

## 2. 规则结构

`packages/schema/src/permission.ts`：

```ts
// :60-64
export const Rule = Schema.Struct({
  action:  Schema.String,     // 动作名，如 "write" / "shell" / "skill" / "question"
  resource: Schema.String,   // 资源标识，如文件路径 / agent.id / 命令串
  effect:  Effect,           // "allow" | "deny" | "ask"（:56）
})
export const Ruleset = Schema.Array(Rule)   // :66 —— 纯数组，无嵌套结构
```

逐字段：

- **action**（string，支持通配）：被评估的输入侧是工具声明的动作名。`Wildcard.match(input.action, rule.action)`——规则侧是 pattern。`"*"` 匹配一切（missingAgentPermissions 即 `[{action:"*", resource:"*", effect:"deny"}]`，permission.ts:19）。
- **resource**（string，支持通配）：输入侧逐 resource 评估（一个 assert 可带多个 resources，如 shell 逐命令、write 逐文件）。
- **effect** 三值：`allow` 放行 / `deny` 拒绝（模型收到 BlockedError）/ `ask` 挂起询问。

**通配语法不是 glob 库，是手写正则变换**（`util/wildcard.ts:3-14`，实测）：

```ts
sed -n '3,14p' packages/core/src/util/wildcard.ts
//   function match(input: string, pattern: string) {
//     const normalized = input.replaceAll("\\", "/")        // 反斜杠归一为 /
//     let escaped = pattern
//       .replaceAll("\\", "/")
//       .replace(/[.+^${}()|[\]\\]/g, "\\$&")   // 先转义正则元字符
//       .replace(/\*/g, ".*")                    // * → .*
//       .replace(/\?/g, ".")                     // ? → 任意单字符
//     if (escaped.endsWith(" .*")) escaped = escaped.slice(0, -3) + "( .*)?"  // 尾部" *"可选
//     return new RegExp("^" + escaped + "$", ...win32加i标志...).test(normalized)  // 全匹配
//   }
```

即：**全锚定（`^...$`）的类 glob**——`*` 跨任意字符（含 `/`），`?` 单字符，Windows 下大小写不敏感，路径分隔符统一 `/`。`"foo *"` 这类尾部星号是"可选后缀"语义。没有 `**` 特殊处理（`*` 本身就跨目录）。例如规则 `resource: "/src/*.ts"` 匹配任意深度的 src 下 .ts 文件（`*` 跨 `/`）；`"/src/**/x"` 里的 `**` 只是两个连续 `.*.*`，与单个 `*` 等价。

**Request / Reply / 事件**（schema :26-54）：

```ts
// Request 字段（:26-39）
sessionID / action / resources / save?(string[], "always" 时物化的 resource 子集)
  / metadata?(Record, 如 write 的 diff 预览) / source?(目前仅 {type:"tool", messageID, id})
// Reply（:42）：字面量 "once" | "always" | "reject"
// Event（:45-54）：permission.asked（payload=Request 全字段）、permission.replied（sessionID/requestID/reply）
//   —— 均为 ephemeral() 事件：**不落 durable 事件流**，只有进程内总线订阅者可见
```

⚠️ 纠偏：asked/replied 是 `ephemeral`（schema/permission.ts:45-53，构造器来自 event.ts:128），**不是 durable 事件**——重启后无事件重放可依赖，恢复语义见 §10。

## 3. 评估算法

评估核心是两个纯函数 + 一个带 hook 的包装：

```ts
// permission.ts:87-97 —— 末位胜出（findLast），不是 deny 优先！
sed -n '87,97p' packages/core/src/permission.ts
//   export function evaluate(action, resource, ...rulesets): Permission.Rule {
//     return rulesets.flat()
//       .findLast((rule) => Wildcard.match(action, rule.action)
//                        && Wildcard.match(resource, rule.resource))
//       ?? { action, resource: "*", effect: "ask" }   // 无命中 → 默认 ask
//   }
// :99-101  merge = rulesets.flat()   —— 合并就是拼接，无去重无排序
```

三条实测结论：

1. **匹配序 = 数组序，末位胜出**（`findLast`）。规则按来源拼接后的物理顺序决定优先级——后面的规则覆盖前面的。
2. **默认拒绝为"询问"**：无任何规则命中时合成 `{effect:"ask"}`（:92-95）——白名单心智，而不是默认放行。
3. **deny 的优先权不在 evaluate 里，而在调用方的前置一票否决**：

```ts
// :165-167
function denied(input, rules) {
  return input.resources.some((r) => evaluate(input.action, r, rules).effect === "deny")
}
// :173-178 evaluateInput
//   const rules = yield* configured(...)          // 只有 configured（agent+session）
//   if (denied(input, rules)) return { effect: "deny", rules }   // ← deny 短路
//   const all = [...rules, ...(yield* savedRules())]             // saved 之后才混入
//   const effects = input.resources.map((r) => evaluate(action, r, all).effect)
//   const effect = effects.includes("ask") ? "ask" : "allow"     // ← ask 传染
```

即：**deny 只在 configured 规则集上检测（saved 永远是 allow，混入前后无差），deny → ask → allow 的短路链在 evaluateInput 内完成**。多 resource 合并取"最严者"：任一 ask 则整体 ask；deny 已在前置过滤。最后 plugin hook `permission.evaluate`（:179-187）拿到 `{effect, ...}` 可改写效应并附 message（deny 时的 reason）——这是插件介入授权的唯一钩子。

`ask`（:223-229）是 assert 的非阻塞变体：同样评估、effect==="ask" 时也会 create 挂起请求（publish asked），但**不等 Deferred**，直接返回 `{id, effect}`——供"预询问"场景（如提前告知 UI 将要发生什么）。

**一个完整数值例子走一遍 evaluate**（规则拼接结果如下，验证末位胜出与 ask 传染）：

```
configured（agent）:
  [0] {action:"edit",  resource:"/src/*",      effect:"allow"}
  [1] {action:"edit",  resource:"/src/secret/*", effect:"deny"}
  [2] {action:"*",     resource:"*",           effect:"ask"}
configured（session 追加）:
  [3] {action:"edit",  resource:"/src/tmp/*",  effect:"allow"}
saved（project 级）:
  [4] {action:"edit",  resource:"/src/tmp/a.ts", effect:"allow"}（物化自某次 always）

assert({action:"edit", resources:["/src/x.ts"]})
  → denied? evaluate(...rules[0..3]) 命中[0] allow、[2] ask → findLast 匹配 [2]? 
    否——findLast 扫描全部规则取最后一个匹配：[2] 在 [0] 之后 → effect "ask"，非 deny → 放行
  → all=[0..4]，evaluate 命中 [2](ask，比 [0] 靠后) → "ask" → 挂起询问
assert({action:"edit", resources:["/src/secret/k.ts"]})
  → denied: 命中 [1] deny → BlockedError（saved 永远救不回）
assert({action:"edit", resources:["/src/tmp/b.ts","/docs/x"]})
  → denied? /src/tmp/b.ts 命中[3] allow、/docs/x 命中[2] ask → 无 deny
  → 全量评估：前者 [3] allow（[2] 之后）→ allow；后者 [2] ask
  → effects=[allow,ask] → ask 传染 → 整体挂起（一次询问批两个 resource）
assert({action:"edit", resources:["/src/tmp/a.ts"]})
  → 全量评估命中 [4]（最末位）→ allow → 直接放行（saved 覆盖 [2] 的 ask）
```

（注意例中 `[0]` 被 `[2]` 覆盖：宽泛的 `*→ask` 规则若写在 allow 之后，会吃掉前面的 allow——**通配宽窄不构成优先级，只有顺序构成**。）

## 4. 规则来源合并

三个来源，按拼接顺序（后者覆盖前者）：

```
[agent.permissions] ++ [session.permissions] ++ [saved(allow-only)]
 ←configured(:158-163)→   ←savedRules(:148-156)→
```

- **agent 内嵌**（configured :158-163）：`agents.resolve(agentID ?? session.agent)` 取 agent 定义；`agent?.permissions ?? missingAgentPermissions`——**agent 未声明 permissions 时兜底 deny-all**（:19，安全缺省：新 agent 默认全锁）。agent 的 permissions 字段本身在配置装载时由 `config/plugin/agent.ts:84-92` 把顶层 `permissions`（config.ts:55-57）push 进**每一个** agent，`:122` 再叠加 per-agent 覆盖块的展开（`expandPermissions`，含 `~/` 路径展开）。字段来源细节归 [6_context.md](6_context.md)/8_agent，此处只记合并位点。
- **session 级覆盖**：`session.permissions`（行投影，由 `session.permissions` durable 事件写入——session/session.ts:82-88 的 `setPermissions` → projector.ts:596 `set({permission: event.data.permissions})`）。拼接在 agent 之后（:162 `merge(agent规则, session.permissions ?? [])`）→ **会话规则覆盖 agent 规则**。
- **saved 决策**：`savedRules()`（:148-156）把当前 project 的所有 saved 行映射为 `{action, resource, effect:"allow"}` 追加在最后——**saved 覆盖一切 ask**；但 deny 已在混入前短路，**saved 不能翻盘 deny**。

另有一条 V1 兼容入口：`config/normalize.ts:180-183` 把旧 `permission`/`tool` 字段经 `migratePermissions`（:535+）与原生 `permissions` 合并为单一 Ruleset——纯配置装载期迁移，与运行时无关。

## 5. 询问协议

**协议三要素**：pending Map（进程内，:130）、Deferred（一对一唤醒，:118）、ephemeral 总线事件（asked/replied）。

```
assert(effect=ask)
  → create(:204-221, uninterruptible)
      Deferred.make → pending.set(id,{request,agent,deferred})
      → bus.publish(Permission.Event.Asked, request)   ← UI 全靠订阅这个
      （publish 失败 → 回滚 pending.delete; 重复 ID → Effect.die）
  → restore(Deferred.await(deferred))                   ← assert 内唯一可中断点(:247)
      catchTag(DeclinedError → Effect.die)               ← decline 隧道（9_tool §6）
      ensuring(pending.delete)                           ← 无论成败都清理(:254-258)

reply(input: {requestID, reply, message?})   (:265-325, uninterruptible)
  ├─ pending 无此 ID → NotFoundError（幂等保护 :269）
  ├─ publish Replied（先出版再改状态，UI 先看到）
  ├─ reject:
  │    message 有 → CorrectedError{feedback}（typed，叶子转 ToolFailure，模型可读）
  │    message 无 → DeclinedError（die 隧道，Step 中断）
  │    ★级联：同 session 其余 pending 全部 reject(:282-295)
  │      注释 :289 "Feedback applies to the whole batch, so parallel asks
  │               don't end the step" —— 并行 ask 视为一批
  ├─ once:  Deferred.succeed → 本次放行
  └─ always:
       save?.length → saved.add({projectID, action, resources: save})(:299-305)
       → 本请求 succeed
       → ★连锁复核：遍历全部 pending，evaluateInput 重评，effect 变 "allow" 的
         自动 publish Replied("always") + succeed(:310-322)
```

要点：

- **无超时**：pending 无 TTL，唯一的全量清理是服务 `close`（:133-146）——Layer 关闭时对所有 pending publish reject + `Deferred.fail(DeclinedError)`，uninterruptible 保证跑完。进程不死则等待可以无限期。
- **挂起时 Step 的状态**：工具 execute 停在 `Deferred.await`；Step 层并不感知"权限等待"这一原因，只有结果出来后（decline）才被 classify（9_tool §6 链路）。UI 侧由 `permission.asked` 事件 + `permission.list` hydrate 汇入 blocker 视图（TUI 篇 5-6 `syncBlockers`）。
- **中断语义**：`Deferred.await` 是 assert 内唯一被 `restore` 恢复可中断的位置（:247，`uninterruptibleMask` 包裹创建与登记）——用户 Ctrl-C 可以打断"等待应答"本身，但 create/reply 的登记与清理段不可打断，pending Map 不会因中断残留半状态（且 `ensuring` 兜底删除，:254-258）。
- **CorrectedError vs DeclinedError 的协议意义**：同一枚 reject 按有无 message 分流（:276-280）——有反馈=「别这么做，应该…」（模型继续、自我修正），无反馈=「就此打住」（Step 终止）。这让"拒绝"成为一个可表达的沟通信道而不只是闸门。
- **三端应答**（各一句）：
  - **TUI**：`mini/runtime.ts:267-274` `onPermissionReply` → `sdk.permission.reply`；asked/replied 由 `stream-v2.transport.ts:1339-1346` 从事件流路由进渲染（子代理流 `stream-v2.subagent.ts:883-889` 同构）。
  - **CLI 非交互**：`cli/src/run/noninteractive.ts:158-183`——无人可答，`--auto` 时一律 `once` 放行，否则**自动 reject 且附反馈**（`PERMISSION_REJECTED_FEEDBACK`，CorrectedError 路径）让模型以普通失败继续；V1 兼容模式则纯 reject + interrupt。
  - **Desktop**：复用 TUI/SDK 的 permission.reply 通道（见 [../4_desktop/1_multi-process.md](../4_desktop/1_multi-process.md) 的进程拓扑），无独立判定逻辑。

## 6. Saved 决策

"记住我的选择" = `always` 应答 + Request 上非空 `save` 数组的合取：

```
permission/re/sql.ts:162-175（实测）
  表 "permission"：id PK / project_id FK(cascade) / action / resource / Timestamps
  uniqueIndex(project_id, action, resource)   ← 幂等键

saved.ts
  list({projectID?})   :109-128   按 project 过滤行 → Info（含 created/updated）
  add({projectID, action, resources})  :130-145
    空数组直接返回；逐 resource 插入，onConflictDoNothing —— 重复"always"无副作用
  remove(id)           :147-149   手动撤销入口（SDK/UI 提供）
```

- **粒度**：save 是 Request 里的**resource 子集**（工具自决哪些 resource 值得记住——如 write 的文件路径、shell 的整条命令）；action 整条存。
- **作用域**：project 级（location.project.id），跨 session、跨 agent；装载在每次 `savedRules()` 调用时实时 list（无缓存，:148-156）——remove 立即生效。
- **语义升格**：saved 行在评估时就是 allow 规则（§4），且因 findLast 末位胜出而**覆盖所有 configured ask**；唯一压不住的是前置短路的 deny。
- **连锁触发**：§5 的 always 连锁复核使"always 一个文件"能瞬间放行当时挂起的其他同规则请求。

## 7. 托管策略（ManagedPolicy）

`managed-policy.ts` 与 Permission 评估器**完全正交**——它是企业/组织管控轴，作用在**目录（catalog）过滤**而不是运行时判定：

```ts
// managed-policy.ts:185-190
/** Policy statements the connected OpenCode Console compiled for whoever it authenticated. */
State = { statements: ConfigPolicy.Info[], organization? }
// :225-239 statements()：文档策略（"Blocked by configuration policy"）
//   ++ 组织策略（"Blocked by ${org}'s policy"）——文档倒序使 user-global 压过 repository
// :241-249 decision()：action 精确等值 + resource 通配，findLast，默认 "allow"
```

- **语句形状**（schema/config/policy.ts:5-12）：action 只有三个字面量 `provider.use | tool.use | integration.use`，effect 只有 `allow|deny`（**无 ask**——管控不商量）。
- **消费点**在 `config/plugin/policy.ts`：作为配置 transform 在目录装载期执行——`provider.use` deny → providers.remove（:30-33）；`integration.use` deny → MCP server / skill 移除（:39,:45）；`tool.use` deny → 工具注册时 `find` 命中即移除（:54-58，resource 匹配 `"${action}:${resource}"` 复合键）。
- **状态管理**：`set()` 整体替换（:196 注释 "statements never merge across connections"，Console 重连不叠加旧策略），深比较无变化不 publish；`changes()` Stream 供目录 transform 之外的重算订阅。
- 取舍对比：**Permission 规则面向"模型行为"（细粒度、可询问、可记住），ManagedPolicy 面向"租户能看见什么"（粗粒度、只删不问、连接级生命周期）**。工具被 policy 删掉后连 `whollyDisabled` 目录过滤都到不了（9_tool §6 第 1 点的更上游）。

## 8. 与 Form/Question 的关系

question 工具先 `permission.assert({action:"question", resources:["*"]})` 再 `forms.ask`（tool/plugin/question.ts，行号见 9_tool.md §4）——即"向用户提问"本身也被建模为一种权限动作（可被规则静默 allow/deny，如 headless 配置直接 deny 免问）。Form 机制本体（form.created/reply/cancel、全局表单）与 Permission 并行，**唯一的接缝就是 question 工具这一次 assert**；其取消语义的 defect 隧道与 Permission decline 同构（question.ts:90-95 注释，9_tool §6 已述）。

## 9. 不变量与设计取舍

从代码与注释提取（9_tool §6 已列的三点不重复）：

1. **默认最严**：无规则命中 → `ask`（evaluate :92-95）；agent 无 permissions → deny-all 兜底（:19）。配置疏漏不会变成静默放行。
2. **deny 不可翻盘**：deny 检测前置且只看 configured（:175）——saved/后续 allow 无论拼接多靠后都救不回 deny。可控的越权路径只有改配置或删规则。
3. **末位胜出而非显式优先级**：`findLast` + `flat`（:87-101）让"来源顺序 = 优先级"零成本成立（agent < session < saved），代价是读代码必须知道拼接序——本篇 §4 即是那份隐式契约的显式化。
4. **挂起状态进程内、协议事件 ephemeral**：pending 是 `Map`（:130），asked/replied 是 ephemeral 事件（§2 纠偏）——询问协议被明确设计为**可丢弃的运行时状态**，恢复靠补偿而非重放（§10）。
5. **durable 先于副作用在权限面的体现**：asked 事件 publish 成功才真正挂起等待（create :216-218 失败回滚）；reply 先 publish Replied 再动 Deferred（:270-274）——UI 看到的事实与执行侧状态机不会错序。
6. **reject 是批量的、always 是连锁的**：同 session 并行 ask 一批同生死（:282-295 注释），always 复核放行（:310-322）——两个方向的"多请求一致性"都由 reply 单点维护，不需要 UI 协调。
7. **ask 传染**：多 resource 任一 ask 整体 ask（:178）——不做部分授权，避免"批一半"的模糊态。

## 10. 失败与恢复

- **询问期间进程崩溃**：pending Map 与 ephemeral 事件随进程消失；等待中的 `Deferred.await` 纤维根本不再存在。durable 侧只停在 `tool.called`——重启后 `resumeSuspendedSessions`（session/execution/restart.ts，见 4_session.md）发现未完结 Step 按中断处理，**权限请求不会重放**（无 durable asked 事件可重放）。代价：用户重启前没答的 ask 等于被丢弃，工具调用按失败收场。
- **UI 侧补偿（hydrate）**：TUI 启动/重连时 `permission.list`（Service.list :327-329 返回全量 pending）拉平客户端视图——5-6 篇 `syncBlockers` 的 hydrate 通道；CLI 非交互在流建立时同样对存量 asked 批量 `replyPermission`（noninteractive.ts:752）。即：**协议不持久化，靠 list-补偿对齐**。
- **服务正常关闭**：close（:133-146）把所有 pending 显式 reject（publish Replied + DeclinedError），不留孤儿 Deferred。
- **重复 reply / 未知 requestID**：NotFoundError typed 失败（:269）；同 ID 重复 create 直接 `Effect.die`（:213-214）——缺陷级，因为这是编程错误而非运行时故障。
- **publish Asked 失败**：回滚 pending 登记（:218），不留"UI 不知道但执行在等"的死锁态。

## 演进注记

- 旧笔记的 **PermissionV2**（survey 期"计划中的重写"）即当前形态：Rule/Ruleset 声明式化 + ephemeral 事件协议。现码已无 V1 痕迹于运行时，仅剩 `v1/config/permission.ts`（42 行）与 `migratePermissions`（normalize.ts:535+）作配置兼容。
- 旧配置 `permission`/`tool` 字段 → 新 `permissions` Ruleset 的迁移在配置装载期一次完成（normalize.ts:180-183），三路合并 `[...toolRules, ...permissionRules, ...nativePermissions]`。
- managed-policy 的 organization 字段与 `set()` 整体替换语义（:196）表明 Console 集成是新近补的轴；`Source` 目前只有 `tool` 一个成员（schema :17-23），为将来非工具来源（如 hook/技能）预留。
- 推测走向（无代码佐证，仅形状暗示）：`AssertInput.id` 的 optional（:43）说明调用方可自带请求 ID（幂等重试）；`Source` 联盟的扩展位；hook `permission.evaluate` 的 message 字段为"拒绝理由可编程"铺路。

## 附：关键源码抽查记录（sed 实测，2026-10-19）

```
$ wc -l packages/core/src/permission.ts packages/core/src/permission/saved.ts \
       packages/core/src/permission/sql.ts packages/core/src/managed-policy.ts \
       packages/schema/src/permission.ts packages/core/src/util/wildcard.ts
  347 / 88 / 20 / 75 / 67 / 14

$ sed -n '87,97p' packages/core/src/permission.ts        # evaluate: findLast + 默认 ask
$ sed -n '165,178p' packages/core/src/permission.ts      # denied 前置 / relevant / evaluateInput 主体
$ sed -n '231,263p' packages/core/src/permission.ts      # assert 全体（含 :248-253 defect tunnel 注释）
$ sed -n '265,325p' packages/core/src/permission.ts      # reply 全体（reject 级联 :282-295、always 连锁 :310-322）
$ sed -n '133,146p' packages/core/src/permission.ts      # close：pending 全量 reject
$ sed -n '42,54p'  packages/core/src/permission.ts       # AssertInput / ReplyInput
$ sed -n '10,14p'  packages/schema/src/permission.ts     # ID "per_" 前缀 + ascending
$ sed -n '45,56p'  packages/schema/src/permission.ts     # ephemeral Asked/Replied + Effect 三值
$ sed -n '60,67p'  packages/schema/src/permission.ts     # Rule / Ruleset
$ sed -n '3,14p'   packages/core/src/util/wildcard.ts    # 通配 → 正则变换（全锚定）
$ sed -n '5,12p'   packages/schema/src/config/policy.ts  # ConfigPolicy：三 action、无 ask
$ sed -n '156,175p' packages/core/src/permission/sql.ts # permission 表 + 唯一索引
$ sed -n '84,92p'   packages/core/src/config/plugin/agent.ts  # 顶层 permissions push 进各 agent
$ sed -n '158,183p' packages/cli/src/run/noninteractive.ts    # CLI 自动 reject + 反馈
$ sed -n '267,274p' packages/tui/src/mini/runtime.ts          # TUI onPermissionReply
$ sed -n '225,249p' packages/core/src/managed-policy.ts       # statements / decision
```

## 相关篇

- [9_tool.md](9_tool.md) §6：assert 时机（叶子）+ decline 隧道三处注释互证（本篇 §5 仅引用不重复）
- [6_context.md](6_context.md)：agent 解析与 permissions 字段的配置装载（§4 合并位点的上游）
- [4_session.md](4_session.md)：session.permissions 投影、resumeSuspendedSessions 崩溃恢复
- [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)：TUI blocker 视图与 permission.list hydrate
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)：Step 内工具等待的时序位置
