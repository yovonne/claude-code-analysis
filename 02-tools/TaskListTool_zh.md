# TaskListTool

## 用途

TaskListTool 列出任务列表（Todo V2 系统）中的所有任务，并提供摘要信息。它提供每个任务的紧凑概览，包括 ID、主题、状态、负责人和未解决的依赖项。内部/元数据任务会自动过滤。已解决（完成）的依赖项也会从 `blockedBy` 列表中过滤，以保持输出专注于可操作的阻塞项。

## 位置

- `restored-src/src/tools/TaskListTool/TaskListTool.ts` — 主要工具定义（约 117 行）
- `restored-src/src/tools/TaskListTool/constants.ts` — 工具名称常量（2 行）
- `restored-src/src/tools/TaskListTool/prompt.ts` — 工具提示和描述（50 行）

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `TaskListTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `TASK_LIST_TOOL_NAME` | 常量：`'TaskList'` |
| `DESCRIPTION` | 常量：`'List all tasks in the task list'` |
| `getPrompt()` | 生成使用提示，针对 Agent Swarms 上下文进行调整 |
| `Output` | Zod 推断的输出类型：`{ tasks: TaskSummary[] }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  // 无参数 — 列出所有任务
}
```

### 输出 Schema

```typescript
{
  tasks: {
    id: string,           // 任务标识符
    subject: string,      // 简短任务描述
    status: TaskStatus,   // 'pending' | 'in_progress' | 'completed'
    owner?: string,       // 如果已分配则为代理 ID，未分配时缺失
    blockedBy: string[],  // 必须先完成的开放任务 ID（已解决 deps 已过滤）
  }[]
}
```

## 工具定义属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'TaskList'` | 内部工具标识符 |
| `searchHint` | `'list all tasks'` | 工具搜索提示 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `userFacingName()` | `'TaskList'` | UI 中的显示名称 |
| `shouldDefer` | `true` | 工具执行可以延迟 |
| `isEnabled()` | `isTodoV2Enabled()` | 仅在 Todo V2 启用时活动 |
| `isConcurrencySafe()` | `true` | 可安全并发运行 |
| `isReadOnly()` | `true` | 只读操作（无副作用） |
| `renderToolUseMessage()` | `null` | 无自定义工具使用消息渲染 |

## 任务列表流程

```
1. 接收输入
   - 模型返回带有 {}（无参数）的 TaskList tool_use

2. 任务列表
   - getTaskListId() 解析当前任务列表
   - listTasks(taskListId) 读取所有任务文件
   - 过滤内部任务（metadata?._internal === true）

3. 依赖过滤
   - 构建已完成任务 ID 的集合
   - 从 blockedBy 数组中过滤已解决的依赖
   - 仅显示未解决的阻塞项

4. 结果映射
   - 每个任务映射到摘要：{ id, subject, status, owner, blockedBy }
```

## 实现细节

### 任务列表

工具委托给 `utils/tasks.ts` 中的 `listTasks()`，它从任务列表目录读取所有任务文件并返回数组。

### 内部任务过滤

具有 `metadata._internal` 设置为 `true` 的任务被过滤掉：
```typescript
const allTasks = (await listTasks(taskListId)).filter(
  t => !t.metadata?._internal,
)
```

这会将系统内部任务隐藏于模型视野之外，保持任务列表清洁并专注于面向用户的工作项。

### 已解决依赖过滤

已完成的任务从 `blockedBy` 列表中移除：
```typescript
const resolvedTaskIds = new Set(
  allTasks.filter(t => t.status === 'completed').map(t => t.id),
)

const tasks = allTasks.map(task => ({
  ...task,
  blockedBy: task.blockedBy.filter(id => !resolvedTaskIds.has(id)),
}))
```

这确保模型只看到活跃的阻塞项。如果依赖已完成，它不再出现在 blockedBy 列表中，使得哪些任务实际被阻塞变得清晰。

### 结果格式化

`mapToolResultToToolResultBlockParam` 方法将结果格式化为人类可读的文本：

**找到任务：**
```
#${task.id} [${task.status}] ${task.subject}${owner}${blocked}
```

其中：
- `owner` = 如果已分配则为 ` (${task.owner})`，否则为空
- `blocked` = 如果 blockedBy 非空则为 ` [blocked by #${id1}, #${id2}]`

**无任务：**
```
No tasks found
```

### 只读分类

工具实现 `isReadOnly() => true`，标记为只读操作，无副作用。

### Agent Swarms 集成

当 `isAgentSwarmsEnabled()` 返回 true 时，提示包括队友特定的工作流指导：
- TaskList 用于在完成任务后查找可用工作
- 查找状态为 `pending`、无负责人且 `blockedBy` 为空的任务
- 按 ID 顺序（最低优先）处理任务以获得顺序上下文
- 使用 TaskUpdate（设置 `owner`）认领任务或等待负责人分配

## 错误处理

| 错误条件 | 处理 |
|----------------|----------|
| 列表中无任务 | 返回空数组，格式化为 "No tasks found" |
| Todo V2 禁用 | 工具不可用（`isEnabled()` 返回 false） |

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | 简单模式——可能影响工具可用性 |

### 功能标志

| 标志 | 用途 |
|------|---------|
| Todo V2（`isTodoV2Enabled()`） | 控制工具可用性 |
| Agent Swarms（`isAgentSwarmsEnabled()`） | 在提示中启用队友工作流指导 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/tasks.ts` | `listTasks`、`getTaskListId`、`isTodoV2Enabled`、`TaskStatusSchema` |
| `utils/lazySchema.ts` | 延迟 schema 评估 |
| `utils/agentSwarmsEnabled.ts` | Agent Swarms 功能检测 |

### 外部

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入/输出 schema 验证 |

## 数据流

```
Model Request
    |
    v
TaskListTool Input {}
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ TASK LISTING                                                     │
│ - getTaskListId() → resolve task list                            │
│ - listTasks(taskListId) → read all task files                    │
│ - Filter out internal tasks (metadata._internal)                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ DEPENDENCY FILTERING                                             │
│ - Build set of completed task IDs                                │
│ - Filter resolved dependencies from blockedBy                    │
│ - Map to summary format                                          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                                │
│ - Format each task as: #ID [status] subject (owner) [blocked]    │
│ - Empty list → "No tasks found"                                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { tasks: [{ id, subject, status, owner?, blockedBy }] }
    |
    v
Model Response: formatted task list
```

## 与任务系统的集成

TaskListTool 提供 Todo V2 任务系统的概览层：

1. **任务发现**：发现可用任务、其状态和依赖项的主要方式
2. **进度跟踪**：通过任务状态显示整体项目进度
3. **依赖可见性**：显示未解决的阻塞项，以便模型可以优先处理解除阻塞工作
4. **队友协调**：在 Agent Swarms 模式下，队友使用 TaskList 查找未分配、未阻塞的任务来认领
5. **ID 顺序偏好**：提示建议按 ID 顺序（最低优先）处理任务，因为早期任务通常为后期任务设置上下文
6. **完成后导航**：完成任务后，模型调用 TaskList 查找新解除阻塞的工作或下一个可用任务

该工具过滤已解决的依赖项以防止混乱——已完成的任务不再显示为阻塞项，保持输出专注于可操作的信息。
