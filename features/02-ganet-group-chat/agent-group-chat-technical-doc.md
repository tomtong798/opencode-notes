# Agent Group Chat 技术文档

## 1. 功能概述

Agent Group Chat 是一个 QQ群聊风格的 Agent 执行过程可视化面板，用于在 OpenCode Desktop 应用中展示多 Agent 协作执行流程。该功能解决了以下核心问题：

- **可视化多 Agent 协作**：清晰展示主 Agent 和 Subagent 的执行流程
- **时间线展示**：按时间顺序展示所有 Agent 的执行过程，精确到秒
- **层级结构**：通过缩进和边框区分 Agent 层级关系
- **内容折叠**：长内容自动折叠，保持界面整洁
- **跨会话持久化**：解决子会话在页面刷新后丢失的问题

### 1.1 功能特性

| 特性 | 说明 |
|------|------|
| 双视图模式 | 支持简洁视图和详细视图切换 |
| 简洁视图 | 每个 Agent 输出一行简短总结 |
| 详细视图 | 展示完整的 Part 执行过程 |
| 时间线展示 | 所有 Part 按时间戳排序显示，精确到秒 |
| Agent 头像 | 每个 Agent 有唯一颜色标识的头像 |
| Subagent 标记 | 子 Agent 显示 "(sub)" 后缀 |
| 内容折叠 | 超过 300 字符或 3 行自动折叠 |
| Agent 列表面板 | 右侧面板显示所有参与 Agent，按 primary/subagent 分组 |
| 层级缩进 | 子 Agent 通过缩进和边框展示层级关系 |
| 国际化支持 | 支持中英文切换 |

## 2. 架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    AgentGroupChatPanel                       │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    AgentsHeader                          │ │
│  │  - 显示参与 Agent 数量                                    │ │
│  │  - 可展开查看所有 Agent 列表                              │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    ScrollView                            │ │
│  │  ┌─────────────────────────────────────────────────────┐│ │
│  │  │              AgentGroupView (主 Agent)               ││ │
│  │  │  - Avatar + 名称 + 时间戳                            ││ │
│  │  │  - Part 列表 (Tool/Reasoning/Text/Step)             ││ │
│  │  └─────────────────────────────────────────────────────┘│ │
│  │  ┌─────────────────────────────────────────────────────┐│ │
│  │  │            AgentGroupView (Subagent)                 ││ │
│  │  │  - 缩进显示 + "(sub)" 标记                           ││ │
│  │  │  - Part 列表                                        ││ │
│  │  └─────────────────────────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 文件结构

```
packages/app/src/
├── components/
│   ├── agent-group-chat-panel.tsx    # 主面板组件
│   └── agent-group-chat-button.tsx   # 触发按钮组件
├── pages/session/
│   └── session-side-panel.tsx        # Tab 集成点
├── context/
│   ├── sync.tsx                      # 数据同步层 (新增 loadChildren)
│   └── global-sync/
│       └── event-reducer.ts          # 事件处理器 (修复 session.created)
└── utils/
    └── agent.ts                      # Agent 颜色工具函数
```

### 2.3 数据流

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Backend    │────▶│  SSE Events  │────▶│  Frontend    │
│  (Database)  │     │              │     │    Store     │
└──────────────┘     └──────────────┘     └──────────────┘
       │                                          │
       │                                          ▼
       │                                 ┌──────────────┐
       │ session.children() API          │  AgentGroup  │
       └────────────────────────────────▶│ ChatPanel    │
                                         └──────────────┘
```

## 3. 核心实现

### 3.1 数据模型

#### ChronologicalPart

按时间排序的 Part 数据结构：

```typescript
type ChronologicalPart = {
  part: Part                          // Part 原始数据
  time: number                        // Part 时间戳
  messageID: string                   // 所属 Message ID
  messageRole: "user" | "assistant"   // Message 角色
  messageAgent: string                // Agent 名称
  messageTime: number                 // Message 创建时间
  depth: number                       // 层级深度 (0 = 主 Agent)
  sessionID: string                   // 所属 Session ID
}
```

#### AgentGroup

Agent 执行分组：

```typescript
type AgentGroup = {
  name: string              // Agent 名称
  displayName: string       // 显示名称 (含 "(sub)" 标记)
  isSub: boolean            // 是否为 Subagent
  startTime: number         // 开始时间
  parts: ChronologicalPart[] // 按 Part 列表
  depth: number             // 层级深度
  sessionID: string         // 所属 Session ID
}
```

### 3.2 组件实现

#### AgentGroupChatPanel (主组件)

**位置**: `packages/app/src/components/agent-group-chat-panel.tsx`

**职责**:
1. 加载子会话数据 (`loadChildren`)
2. 收集所有 Agent 执行分组
3. 按时间排序展示

**关键代码**:

```typescript
export function AgentGroupChatPanel(props: AgentGroupChatPanelProps) {
  const sync = useSync()
  const [childSessionsLoaded, setChildSessionsLoaded] = createSignal(false)

  // 组件挂载时加载子会话 - 从服务器获取，因为子会话不会在 store 初始化时加载
  createEffect(on(() => props.sessionID, (sessionID) => {
    if (!sessionID) return
    setChildSessionsLoaded(false)
    sync.session.loadChildren(sessionID)
      .then(() => setChildSessionsLoaded(true))
      .catch((err) => {
        console.error('[AgentGroupChat] loadChildren failed:', err)
        setChildSessionsLoaded(true)
      })
  }))

  // 收集所有 Agent 分组
  const groups = createMemo(() => {
    const id = props.sessionID
    if (!id) return []

    const providers = sync.data.provider.all
    const visited = new Set<string>()
    const result = collectGroups(id, sync, providers, 0, visited)

    result.sort((a, b) => a.startTime - b.startTime)
    return result
  })

  return (
    <div class="flex flex-col h-full bg-background-base">
      <AgentsHeader groups={groups()} />
      <ScrollView class="flex-1 min-h-0">
        <For each={groups()}>{(group) => <AgentGroupView group={group} />}</For>
      </ScrollView>
    </div>
  )
}
```

#### collectGroups (递归收集函数)

**职责**: 递归收集主会话和所有子会话中的 Agent 执行分组

```typescript
function collectGroups(
  sessionID: string,
  sync: ReturnType<typeof useSync>,
  providers: any,
  depth: number,
  visited: Set<string>
): AgentGroup[] {
  if (visited.has(sessionID)) return []
  visited.add(sessionID)

  const msgs = sync.data.message[sessionID] ?? []
  const result: AgentGroup[] = []

  // 处理当前会话的所有 assistant message
  for (const msg of msgs) {
    if (msg.role !== "assistant") continue

    const msgParts = sync.data.part[msg.id] ?? []
    if (msgParts.length === 0) continue

    const agentName = (msg as AssistantMessage).agent || "UNKNOWN AGENT"
    const isSub = isSubagent(agentName, providers)

    // 收集并排序所有 Part
    const groupParts: ChronologicalPart[] = msgParts
      .filter(part => part.type !== "agent")
      .map(part => ({
        part,
        time: getPartTime(part),
        messageID: msg.id,
        messageRole: "assistant",
        messageAgent: agentName,
        messageTime: msg.time.created,
        depth,
        sessionID,
      }))
      .sort((a, b) => a.time - b.time)

    if (groupParts.length > 0) {
      result.push({
        name: agentName,
        displayName: depth > 0 ? `${agentName} (sub)` : (isSub ? `${agentName} (sub)` : agentName),
        isSub: depth > 0 || isSub,
        startTime: earliestTime,
        parts: groupParts,
        depth,
        sessionID,
      })
    }
  }

  // 递归处理子会话
  const childSessions = sync.data.session.filter(s => s.parentID === sessionID)
  for (const child of childSessions) {
    const childGroups = collectGroups(child.id, sync, providers, depth + 1, visited)
    result.push(...childGroups)
  }

  return result
}
```

#### AgentGroupView (分组视图)

**职责**: 渲染单个 Agent 执行分组

**特性**:
- 根据 `depth` 计算缩进
- 主 Agent 有底部边框，子 Agent 有左边框
- 支持展开/折叠

```typescript
function AgentGroupView(props: { group: AgentGroup }) {
  const [collapsed, setCollapsed] = createSignal(false)

  const indentStyle = () => ({
    "margin-left": `${props.group.depth * 16}px`,
  })

  const borderStyle = () => {
    if (props.group.depth === 0) return "border-b border-border-weak-base"
    return "border-l-2 border-border-weaker"
  }

  return (
    <div class={`py-4 px-3 ${borderStyle()}`} style={indentStyle()}>
      <div class="flex gap-3">
        <AgentAvatar name={props.group.name} size="large" />
        <div class="flex-1 min-w-0">
          <div class="flex items-baseline gap-2 mb-2">
            <span class="text-14-medium">{props.group.displayName}</span>
            <span class="text-11-regular text-text-weak">
              {formatTime(props.group.startTime)}
            </span>
            <button onClick={() => setCollapsed(!collapsed())}>
              {collapsed() ? "(展开)" : "(折叠)"}
            </button>
          </div>
          <Show when={!collapsed()}>
            <For each={props.group.parts}>
              {(item) => <PartView item={item} />}
            </For>
          </Show>
        </div>
      </div>
    </div>
  )
}
```

### 3.3 Part 类型展示

系统支持 5 种 Part 类型，每种有独特的展示方式：

| Part 类型 | 图标 | 说明 |
|-----------|------|------|
| ToolPart | 🛠️ | 显示工具名称、状态、输入输出 |
| ReasoningPart | 💭 | 显示推理过程文本 |
| TextPart | 💬 | 显示输出文本 |
| StepStartPart | ▶️ | 标记步骤开始 |
| StepFinishPart | ✅ | 显示步骤完成信息、费用、Token 统计 |

### 3.4 内容折叠逻辑

```typescript
function shouldCollapse(content: string): boolean {
  return content.length > 300 || content.split("\n").length > 3
}

function CollapsibleContent(props: { content: string }) {
  const [collapsed, setCollapsed] = createSignal(shouldCollapse(props.content))

  return (
    <Show
      when={!collapsed()}
      fallback={
        <button onClick={() => setCollapsed(false)}>
          {props.content.slice(0, 300)}...
          <span class="text-accent-base ml-1">(展开)</span>
        </button>
      }
    >
      <pre class="text-12-regular whitespace-pre-wrap">
        {props.content}
      </pre>
      <Show when={shouldCollapse(props.content)}>
        <button onClick={() => setCollapsed(true)}>(收起)</button>
      </Show>
    </Show>
  )
}
```

## 4. 子会话持久化修复

### 4.1 问题分析

**现象**: Subagent 执行过程只在 Desktop 打开时显示，页面刷新后消失。

**根本原因**:

1. **前端初始化加载机制**: `loadRootSessionsWithFallback` 使用 `roots: true` 参数，只加载根会话，不加载子会话

```typescript
// packages/app/src/context/global-sync/session-load.ts
const result = await client.session.list({
  projectID: project.id,
  roots: true,  // <-- 只加载根会话
  limit,
})
```

2. **实时事件接收**: 子会话在运行时通过 `session.created` 事件到达前端，但页面刷新后这些数据丢失

3. **事件处理问题**: `trimSessions` 函数可能在处理 `session.created` 事件时错误地移除子会话

### 4.2 解决方案

#### 修复 1: 事件处理器 - 跳过子会话的 trim 操作

**文件**: `packages/app/src/context/global-sync/event-reducer.ts`

```typescript
case "session.created": {
  const info = (event.properties as { info: Session }).info
  const result = Binary.search(input.store.session, info.id, (s) => s.id)
  if (result.found) {
    input.setStore("session", result.index, reconcile(info))
    break
  }
  const next = input.store.session.slice()
  next.splice(result.index, 0, info)
  
  // 修复: 对于子会话，不执行 trim 操作
  // 因为父会话可能不在 keepRootIds 中，但仍需保留新创建的子会话
  if (info.parentID) {
    input.setStore("session", reconcile(next, { key: "id" }))
  } else {
    const trimmed = trimSessions(next, { limit: input.store.limit, permission: input.store.permission })
    input.setStore("session", reconcile(trimmed, { key: "id" }))
  }
  
  if (!info.parentID) input.setStore("sessionTotal", (value) => value + 1)
  break
}
```

#### 修复 2: 添加 loadChildren 方法

**文件**: `packages/app/src/context/sync.tsx`

```typescript
async loadChildren(parentID: string) {
  const directory = sdk.directory
  const client = sdk.client
  const [store, setStore] = globalSync.child(directory)
  
  // 调用后端 API 获取子会话
  const children = await retry(() => client.session.children({ sessionID: parentID }))
  const childSessions = children.data ?? []
  
  if (childSessions.length === 0) return
  
  // 将子会话添加到 store
  setStore(
    "session",
    produce((draft) => {
      for (const child of childSessions) {
        const match = Binary.search(draft, child.id, (s) => s.id)
        if (!match.found) {
          draft.splice(match.index, 0, child)
        }
      }
    }),
  )
  
  // 加载子会话的消息
  await Promise.all(
    childSessions.map((child) => 
      loadMessages({
        directory,
        client,
        setStore,
        sessionID: child.id,
        limit: messagePageSize,
      })
    )
  )
}
```

#### 修复 3: 组件挂载时加载子会话

**文件**: `packages/app/src/components/agent-group-chat-panel.tsx`

```typescript
export function AgentGroupChatPanel(props: AgentGroupChatPanelProps) {
  const sync = useSync()
  const [childSessionsLoaded, setChildSessionsLoaded] = createSignal(false)

  // 组件挂载时加载子会话
  createEffect(on(() => props.sessionID, (sessionID) => {
    if (!sessionID) return
    setChildSessionsLoaded(false)
    sync.session.loadChildren(sessionID)
      .then(() => setChildSessionsLoaded(true))
      .catch((err) => {
        console.error('[AgentGroupChat] loadChildren failed:', err)
        setChildSessionsLoaded(true)
      })
  }))

  // 等待数据加载完成
  const ready = createMemo(() => {
    const id = props.sessionID
    if (!id) return true
    if (!sync.data.message[id]) return false
    if (!childSessionsLoaded()) return false
    
    // 检查所有子会话的消息是否已加载
    const childSessions = sync.data.session.filter(s => s.parentID === id)
    for (const child of childSessions) {
      if (!sync.data.message[child.id]) return false
    }
    return true
  })

  // ...
}
```

### 4.3 后端 API

**文件**: `packages/opencode/src/session/index.ts`

```typescript
export const children = fn(Identifier.schema("session"), async (parentID) => {
  const project = Instance.project
  const rows = Database.use((db) =>
    db
      .select()
      .from(SessionTable)
      .where(and(
        eq(SessionTable.project_id, project.id), 
        eq(SessionTable.parent_id, parentID)
      ))
      .all(),
  )
  return rows.map(fromRow)
})
```

### 4.4 数据流图

```
页面刷新
    │
    ▼
loadRootSessionsWithFallback (roots: true)
    │
    ├── 加载根会话 ✅
    └── 子会话未加载 ❌
    │
    ▼
AgentGroupChatPanel 挂载
    │
    ▼
sync.session.loadChildren(parentID)
    │
    ├── client.session.children({ sessionID }) API 调用
    │
    ▼
后端查询 SessionTable (WHERE parent_id = ?)
    │
    ▼
返回子会话列表
    │
    ▼
前端 Store 更新
    │
    ├── 添加子会话到 session 列表
    └── 加载子会话消息
    │
    ▼
UI 渲染完成 ✅
```

## 5. Tab 集成

### 5.1 Tab 注册

**文件**: `packages/app/src/pages/session/session-side-panel.tsx`

Tab 集成通过现有的 Tab 系统实现，无需显式注册：

```typescript
// 检查 Tab 是否打开
const agentGroupChatOpen = createMemo(() => 
  tabs().active() === "agent-group-chat" || 
  tabs().all().includes("agent-group-chat")
)

// 在 Tab 内容区渲染
<Show when={agentGroupChatOpen()}>
  <Tabs.Content value="agent-group-chat" class="flex flex-col h-full overflow-hidden">
    <Show when={activeTab() === "agent-group-chat"}>
      <div class="relative pt-2 flex-1 min-h-0 overflow-hidden">
        <AgentGroupChatPanel sessionID={params.id ?? ""} />
      </div>
    </Show>
  </Tabs.Content>
</Show>
```

### 5.2 触发按钮

**文件**: `packages/app/src/components/agent-group-chat-button.tsx`

```typescript
export function AgentGroupChatButton() {
  const layout = useLayout()
  const params = useParams()

  const sessionKey = createMemo(() => 
    `${params.dir}${params.id ? "/" + params.id : ""}`
  )
  const tabs = createMemo(() => layout.tabs(sessionKey))
  const view = createMemo(() => layout.view(sessionKey))

  const openPanel = () => {
    // 如果已打开，则关闭
    if (tabs().active() === "agent-group-chat") {
      tabs().close("agent-group-chat")
      return
    }
    // 打开右侧面板并激活 Tab
    if (!view().reviewPanel.opened()) view().reviewPanel.open()
    tabs().open("agent-group-chat")
    tabs().setActive("agent-group-chat")
  }

  return (
    <Show when={params.id}>
      <Tooltip value="Group Chat" placement="top">
        <Button variant="ghost" onClick={openPanel}>
          <Icon name="bubble-5" />
        </Button>
      </Tooltip>
    </Show>
  )
}
```

## 6. 工具函数

### 6.1 Agent 颜色

**文件**: `packages/app/src/utils/agent.ts`

```typescript
const defaults: Record<string, string> = {
  ask: "var(--icon-agent-ask-base)",
  build: "var(--icon-agent-build-base)",
  docs: "var(--icon-agent-docs-base)",
  plan: "var(--icon-agent-plan-base)",
}

export function agentColor(name: string, custom?: string) {
  if (custom) return custom
  return defaults[name] ?? defaults[name.toLowerCase()]
}
```

### 6.2 Subagent 判断

```typescript
function isSubagent(agentName: string, providers: any): boolean {
  for (const providerID in providers) {
    const provider = providers[providerID]
    if (provider?.agents) {
      for (const agent of provider.agents) {
        if (agent.id === agentName || agent.name === agentName) {
          return agent.mode === "subagent"
        }
      }
    }
  }
  return false
}
```

### 6.3 时间格式化

```typescript
function formatTime(time: number): string {
  const date = new Date(time)
  return date.toLocaleTimeString(undefined, { timeStyle: "short" })
}
```

## 7. 性能优化

### 7.1 响应式更新

使用 SolidJS 的 `createMemo` 实现高效的响应式更新：

```typescript
// 仅在相关数据变化时重新计算
const groups = createMemo(() => {
  const key = messagesKey()
  if (!key) return []
  // ... 计算
})

// 使用 key 精确跟踪依赖
const messagesKey = createMemo(() => {
  const keys: string[] = []
  for (const sid of allSessionIds) {
    const msgs = sync.data.message[sid] ?? []
    for (const m of msgs) {
      const parts = sync.data.part[m.id] ?? []
      keys.push(`${sid}:${m.id}:${parts.map(p => p.id).join('.')}`)
    }
  }
  return keys.join('|')
})
```

### 7.2 防止重复计算

```typescript
// 使用 visited Set 防止循环引用
function collectGroups(
  sessionID: string,
  // ...
  visited: Set<string>
): AgentGroup[] {
  if (visited.has(sessionID)) return []
  visited.add(sessionID)
  // ...
}
```

## 8. 测试验证

### 8.1 手动测试步骤

1. 启动 Desktop 应用
2. 执行一个会触发 Subagent 的任务（例如使用 `task()` 工具）
3. 点击 Group Chat 按钮打开面板
4. 验证：
   - 主 Agent 和 Subagent 都显示
   - Subagent 有 "(sub)" 标记
   - 时间线正确排序
   - 内容折叠功能正常
5. **刷新页面**
6. 再次打开 Group Chat 面板
7. 验证 Subagent 执行过程仍然显示

### 8.2 预期结果

| 测试项 | 预期结果 |
|--------|----------|
| 主 Agent 显示 | ✅ 显示名称、头像、执行过程 |
| Subagent 显示 | ✅ 显示名称+"(sub)"、缩进、执行过程 |
| 时间排序 | ✅ 按 Part 时间戳正确排序 |
| 内容折叠 | ✅ 超过 300 字符/3 行自动折叠 |
| Agent 统计 | ✅ 表头显示正确的 Agent 数量 |
| 页面刷新后 | ✅ Subagent 执行过程仍然显示 |

## 9. 未来优化方向

1. **虚拟滚动**: 当 Part 数量过多时，实现虚拟滚动优化性能
2. **搜索过滤**: 添加搜索功能，快速定位特定 Agent 或 Part
3. **时间范围筛选**: 按时间范围过滤显示内容
4. **导出功能**: 支持导出为 Markdown 或 JSON 格式
5. **深色模式优化**: 进一步优化深色主题下的显示效果

## 10. 相关文件索引

| 文件路径 | 说明 |
|----------|------|
| `packages/app/src/components/agent-group-chat-panel.tsx` | 主面板组件 |
| `packages/app/src/components/agent-group-chat-button.tsx` | 触发按钮 |
| `packages/app/src/pages/session/session-side-panel.tsx` | Tab 集成点 |
| `packages/app/src/context/sync.tsx` | 数据同步层 (loadChildren 方法) |
| `packages/app/src/context/global-sync/event-reducer.ts` | 事件处理器 |
| `packages/app/src/utils/agent.ts` | Agent 工具函数 |
| `packages/opencode/src/session/index.ts` | 后端 Session API |

---

**文档版本**: v1.0  
**最后更新**: 2026-04-08  
**作者**: OpenCode AI Agent
