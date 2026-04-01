# TaskGetTool

## 用途

TaskGetTool 从任务列表（Todo V2 系统）中按 ID 检索单个任务。它返回完整的任务详细信息，包括主题、描述、状态和依赖信息（blocks/blockedBy）。这是单个任务的只读查找工具，是对 TaskListTool（提供所有任务的摘要视图）的补充。

## 位置

- `restored-src/src/tools/TaskGetTool/TaskGetTool.ts` — 主要工具定义（约 129 行）
- `restored-src/src/tools/TaskGetTool/constants.ts` — 工具名称常量（2 行）
- `restored-src/src/tools/TaskGetTool/prompt.ts` — 工具提示和描述（25 行）

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `TaskGetTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `TASK_GET_TOOL_NAME` | 常量：`'TaskGet'` |
| `DESCRIPTION` | 常量：`'Get a task by ID from the task list'` |
| `PROMPT` | 模型的使用说明 |
| `Output` | Zod 推断的输出类型：`{ task: Task | null }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  taskId: string,    // 必需：要检索的任务的 ID
}
```

### 输出 Schema

```typescript
{
  task: {
    id: string,           // 任务标识符
    subject: string,      // 任务标题
    description: string,  // 详细需求/上下文
    status: TaskStatus,   // 'pending' | 'in_progress' | 'completed'
    blocks: string[],     // 此任务阻止的任务 ID
    blockedBy: string[],  // 阻止此任务的任务 ID
  } | null                // 如果任务未找到则为 null
}
```

## 工具定义属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'TaskGet'` | 内部工具标识符 |
| `searchHint` | `'retrieve a task by ID'` | 工具搜索提示 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `userFacingName()` | `'TaskGet'` | UI 中的显示名称 |
| `shouldDefer` | `true` | 工具执行可以延迟 |
| `isEnabled()` | `isTodoV2Enabled()` | 仅在 Todo V2 启用时活动 |
| `isConcurrencySafe()` | `true` | 可安全并发运行 |
| `isReadOnly()` | `true` | 只读操作（无副作用） |
| `toAutoClassifierInput()` | `input.taskId` | 使用 taskId 进行自动模式分类 |
| `renderToolUseMessage()` | `null` | 无自定义工具使用消息渲染 |

## 任务检索流程

```
1. 接收输入
   - 模型返回带有 { taskId } 的 TaskGet tool_use

2. 任务查找
   - getTaskListId() 解析当前任务列表
   - getTask(taskListId, taskId) 读取任务文件
   - 返回任务对象或 null（如果未找到）

3. 结果格式化
   - 如果任务未找到：{ task: null }
   - 如果找到：{ task: { id, subject, description, status, blocks, blockedBy } }
```

## 实现细节

### 任务查找

工具委托给 `utils/tasks.ts` 中的 `getTask()`，它从任务列表目录读取任务文件。任务文件包含持久化为 JSON 的所有任务字段。

### 空处理

如果任务 ID 不对应现有任务，工具返回 `{ task: null }` 而不是抛出错误。这是一个良性条件——它允许模型优雅地处理缺失的任务（例如任务已被删除或 ID 不正确）。

### 结果格式化

`mapToolResultToToolResultBlockParam` 方法将结果格式化为人类可读的文本：

**找到任务：**
```
Task #${task.id}: ${task.subject}
Status: ${task.status}
Description: ${task.description}
Blocked by: #${id1}, #${id2}     // 仅当 blockedBy 非空时
Blocks: #${id1}, #${id2}         // 仅当 blocks 非空时
```

**任务未找到：**
```
Task not found
```

### 只读分类

工具实现 `isReadOnly() => true`，标记为只读操作。这意味着：
- 它不修改任何状态
- 可以安全调用而无需副作用
- 在某些权限模式下可能自动批准

## 错误处理

| 错误条件 | 处理 |
|----------------|----------|
| 任务 ID 未找到 | 返回 `{ task: null }` — 不是错误 |
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

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/tasks.ts` | `getTask`、`getTaskListId`、`isTodoV2Enabled`、`TaskStatusSchema` |
| `utils/lazySchema.ts` | 延迟 schema 评估 |

### 外部

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入/输出 schema 验证 |

## 数据流

```
Model Request
    |
    v
TaskGetTool Input { taskId }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ TASK LOOKUP                                                      │
│ - getTaskListId() → resolve task list                            │
│ - getTask(taskListId, taskId) → read task file                  │
│ - Returns task object or null                                    │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                                │
│ - Task found: format with subject, status, description, deps     │
│ - Task not found: "Task not found"                              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { task: { id, subject, description, status, blocks, blockedBy } | null }
    |
    v
Model Response: formatted task details or "Task not found"
```

## 与任务系统的集成

TaskGetTool 在 Todo V2 生态系统中提供单个任务访问：

1. **补充 TaskListTool**：虽然 TaskListTool 显示所有任务的摘要，但 TaskGetTool 检索特定任务的完整详细信息
2. **更新前验证**：提示建议在更新之前通过 TaskGet 读取任务的最新状态（防止陈旧）
3. **依赖感知**：返回 `blocks` 和 `blockedBy` 数组，以便模型可以理解任务依赖链
4. **状态检查**：允许模型在工作或更新之前验证任务的当前状态
5. **分配上下文**：返回完整描述字段，其中可能包含分配工作的详细需求

该工具直接从磁盘读取任务文件，确保无论内存缓存如何，始终返回最新状态。
