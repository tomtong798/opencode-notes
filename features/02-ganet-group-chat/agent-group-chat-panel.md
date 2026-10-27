# Agent Group Chat Panel 设计方案

## 功能概述

Agent Group Chat Panel 是一个群聊风格的界面，用于展示多 Agent 协作执行过程。支持显示主 Agent 和 Subagent 的完整执行流程，包括工具调用、推理过程、输出内容等。

## 架构设计

### 数据模型

```
Session 层级结构:
┌─────────────────────────────────────────────┐
│ 主 Session (sessionID)                       │
│   ├── AssistantMessage (agent: "build")     │
│   │   ├── ToolPart (task)                   │
│   │   ├── ReasoningPart                     │
│   │   └── TextPart                          │
│   │                                          │
│   └── 子 Session (parentID = sessionID)     │
│       ├── AssistantMessage (agent: "librarian")
│       │   ├── ToolPart (grep, read...)      │
│       │   └── TextPart (分析结果)            │
│       │                                      │
│       └── 孙 Session (parentID = childID)   │
│           └── ...                            │
└─────────────────────────────────────────────┘
```

### 核心类型定义

```typescript
type AgentGroup = {
  name: string           // Agent 名称
  displayName: string    // 显示名称 (包含 sub 标记)
  isSub: boolean         // 是否为 subagent
  startTime: number      // 开始时间
  parts: ChronologicalPart[]  // 执行内容
  depth: number          // 嵌套深度 (0=主 session, 1+=子 session)
  sessionID: string      // 来源 session ID
}

type ChronologicalPart = {
  part: Part             // 原始 part 数据
  time: number           // 时间戳
  messageID: string      // 消息 ID
  messageRole: "user" | "assistant"
  messageAgent: string   // Agent 名称
  messageTime: number    // 消息创建时间
  depth: number          // 嵌套深度
  sessionID: string      // 来源 session ID
}
```

### 数据收集流程

```
collectGroups(sessionID, depth=0)
    │
    ├── 遍历 session 的所有 AssistantMessage
    │   ├── 提取 agent 字段作为 Agent 名称
    │   ├── 收集所有 Part (tool/reasoning/text/step-*)
    │   ├── 计算最早时间作为 startTime
    │   └── 创建 AgentGroup (depth = 当前 depth)
    │
    ├── 查找子 session (parentID === sessionID)
    │   └── collectGroups(childID, depth + 1)
    │
    └── 返回所有 AgentGroup
```

### 响应式更新机制

```typescript
const messagesKey = createMemo(() => {
  // 1. 收集所有相关 session ID (主 session + 递归子 session)
  // 2. 为每个 session 的每个 message + part 生成唯一 key
  // 3. 组合成完整 key 字符串
  return keys.join('|')
})

const groups = createMemo(() => {
  const key = messagesKey()  // 建立依赖关系
  // ... 数据收集逻辑
})
```

## 显示逻辑

### 层级可视化

| Depth | 样式 |
|-------|------|
| 0 | 无缩进，底部边框 |
| 1+ | 左侧缩进 16px * depth，左侧竖线边框 |

### Agent 标识

- 主 session agent: 直接显示名称 (如 "build")
- 子 session agent: 显示名称 + "(sub)" 后缀
- Provider 配置为 subagent 的 agent: 显示名称 + "(sub)" 后缀

### Part 类型与显示

| Part Type | 图标 | 显示内容 |
|-----------|------|----------|
| tool | 🛠️ | 工具名 + 状态 + 输入/输出 |
| reasoning | 💭 | 推理文本 |
| text | 💬 | 输出文本 |
| step-start | ▶️ | 步骤开始标记 |
| step-finish | ✅ | 步骤完成 + token 统计 + 费用 |

### 内容折叠规则

- 单个 Part 内容 > 300 字符 或 > 3 行时折叠
- 每个 Agent Group 整体可折叠
- 工具输入/输出独立折叠

## 技术实现要点

### 1. Session 层级遍历

```typescript
function collectGroups(
  sessionID: string,
  sync: ReturnType<typeof useSync>,
  providers: any,
  depth: number,
  visited: Set<string>
): AgentGroup[] {
  // 防止循环引用
  if (visited.has(sessionID)) return []
  visited.add(sessionID)

  // ... 收集当前 session 的 groups

  // 递归处理子 session
  const childSessions = sync.data.session.filter(s => s.parentID === sessionID)
  for (const child of childSessions) {
    const childGroups = collectGroups(child.id, sync, providers, depth + 1, visited)
    result.push(...childGroups)
  }

  return result
}
```

### 2. 时间排序

所有 groups 按 `startTime` 升序排序，确保执行顺序正确：
```typescript
result.sort((a, b) => a.startTime - b.startTime)
```

### 3. Agent 类型判断

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

## 使用示例

### 显示效果

```
┌─────────────────────────────────────────┐
│ Group Chat (3 Agents)                    │
├─────────────────────────────────────────┤
│ [build] 10:30:45                         │
│   🛠️ task (completed)                    │
│   💭 推理过程...                          │
│   💬 输出内容...                          │
├─────────────────────────────────────────┤
│   │ [librarian (sub)] 10:30:50           │
│   │   🛠️ grep (completed)                │
│   │   🛠️ read (completed)                │
│   │   💬 找到相关文档...                   │
│   │                                      │
│   │ [explore (sub)] 10:31:05             │
│   │   🛠️ grep (completed)                │
│   │   💬 搜索结果...                       │
├─────────────────────────────────────────┤
│ [build] 10:31:30                         │
│   💬 综合分析结果...                       │
└─────────────────────────────────────────┘
```

## 文件结构

```
packages/app/src/components/
├── agent-group-chat-panel.tsx    # 主组件
├── agent-group-chat-panel.md     # 本文档
└── agent-group-chat-button.tsx   # 入口按钮
```

## 集成点

- `session-side-panel.tsx` Line 376: `<AgentGroupChatPanel sessionID={params.id ?? ""} />`

## 注意事项

1. **循环引用防护**: `collectGroups` 使用 `visited` Set 防止无限递归
2. **响应式依赖**: `messagesKey` 必须包含所有子 session 的消息 ID 才能正确触发更新
3. **类型安全**: 使用 `as const` 确保字面量类型推断
4. **空数据处理**: 过滤空 part 列表，避免创建空 group

## 关键问题修复

### 问题 1: 页面刷新后子 Session 消失

**问题**: 页面刷新后，子 session (subagent) 执行过程不显示。

**根因**: 
1. `loadRootSessionsWithFallback` 使用 `roots: true` 参数，只加载 root sessions
2. 子 session 虽然持久化在后端数据库，但前端启动时不会加载它们
3. 只有实时事件 (`session.created`) 会将子 session 添加到 store

**解决方案**: 
1. 在 `sync.tsx` 添加 `session.loadChildren(parentID)` 方法，调用后端 API 获取子 sessions
2. 在 `AgentGroupChatPanel` 组件挂载时，调用 `loadChildren` 加载当前 session 的子 sessions

```typescript
// sync.tsx
async loadChildren(parentID: string) {
  const children = await client.session.children({ sessionID: parentID })
  // Add children to store and load their messages
}

// agent-group-chat-panel.tsx
createEffect(on(() => props.sessionID, (sessionID) => {
  sync.session.loadChildren(sessionID)
}))
```

### 问题 2: session.created 事件处理时子 Session 被 Trim

**问题**: 当 subagent 创建时，`session.created` 事件到达前端，但子 session 可能被 trim 掉。

**解决方案**: 对于子 session（有 `parentID`），跳过 trimming：

```typescript
case "session.created": {
  const info = (event.properties as { info: Session }).info
  // ...
  if (info.parentID) {
    input.setStore("session", reconcile(next, { key: "id" }))
  } else {
    const trimmed = trimSessions(next, { limit: input.store.limit, permission: input.store.permission })
    input.setStore("session", reconcile(trimmed, { key: "id" }))
  }
}
```
```
