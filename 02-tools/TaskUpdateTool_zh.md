# TaskUpdateTool

## 目的

TaskUpdateTool 修改任务列表（Todo V2 系统）中的现有任务。它支持更新任务状态（包括特殊的 `deleted` 操作）、主题、描述、active form、owner、元数据以及依赖关系（blocks/blockedBy）。它在任务被标记为完成时执行完成后的钩子，通过邮箱处理队友分配通知，并在没有验证步骤的情况下关闭任务时提供验证提醒。

## 位置

- `restored-src/src/tools/TaskUpdateTool/TaskUpdateTool.ts` — 主工具定义（约 407 行）
- `restored-src/src/tools/TaskUpdateTool/constants.ts` — 工具名称常量（2 行）
- `restored-src/src/tools/TaskUpdateTool/prompt.ts` — 工具提示和描述（78 行）

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `TaskUpdateTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `TASK_UPDATE_TOOL_NAME` | 常量：`'TaskUpdate'` |
| `DESCRIPTION` | 常量：`'Update a task in the task list'` |
| `PROMPT` | 带状态工作流、字段描述和示例的使用说明 |
| `Output` | Zod 推断的输出类型：`{ success, taskId, updatedFields, error?, statusChange?, verificationNudgeNeeded? }` |

## 输入/输出模式

### 输入模式

```typescript
{
  taskId: string,           // 必需：要更新的任务的 ID
  subject?: string,         // 可选：新任务标题
  description?: string,     // 可选：新任务描述
  activeForm?: string,      // 可选：用于 spinner 的现在进行时形式
  status?: TaskStatus | 'deleted',  // 可选：新状态或 'deleted' 操作
  addBlocks?: string[],     // 可选：此任务阻塞的任务 ID
  addBlockedBy?: string[],  // 可选：阻塞此任务的任务 ID
  owner?: string,           // 可选：新任务 owner（agent 名称）
  metadata?: Record<string, unknown>,  // 可选：要合并的元数据（null 删除键）
}
```

### 输出模式

```typescript
{
  success: boolean,                    // 更新是否成功
  taskId: string,                      // 被更新的任务
  updatedFields: string[],             // 实际更改的字段列表
  error?: string,                      // success 为 false 时的错误消息
  statusChange?: {                     // 仅在状态更改时存在
    from: string,
    to: string,
  },
  verificationNudgeNeeded?: boolean,   // 当关闭 3+ 个任务没有验证时为 true
}
```

## 工具定义属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'TaskUpdate'` | 内部工具标识符 |
| `searchHint` | `'update a task'` | 工具搜索提示 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `userFacingName()` | `'TaskUpdate'` | UI 中的显示名称 |
| `shouldDefer` | `true` | 工具执行可以延迟 |
| `isEnabled()` | `isTodoV2Enabled()` | 仅在启用 Todo V2 时激活 |
| `isConcurrencySafe()` | `true` | 可以安全并发运行 |
| `toAutoClassifierInput()` | `input.taskId + input.status + input.subject` | 用于自动模式分类的组合 |
| `renderToolUseMessage()` | `null` | 无自定义工具使用消息渲染 |

## 任务更新流程

```
1. 接收输入
   - Model 返回 TaskUpdate tool_use，附带 { taskId, ...updates }

2. UI 自动展开
   - 在 app state 中自动展开任务列表视图

3. 任务存在检查
   - getTask(taskListId, taskId) 检索当前任务
   - 如果未找到：返回 { success: false, error: 'Task not found' }

4. 字段更新（仅当与当前值不同时）
   - subject, description, activeForm, owner
   - metadata（合并，null 值删除键）
   - 队友标记 in_progress 时自动设置 owner（Agent Swarms）

5. 状态处理
   - 'deleted'：deleteTask() 并提前返回
   - 'completed'：在应用前执行 executeTaskCompletedHooks()
   - 其他：如果与当前状态不同则应用

6. 应用更新
   - 如果有任何更改则 updateTask(taskListId, taskId, updates)

7. Owner 通知（Agent Swarms）
   - writeToMailbox() 通知新 owner 分配

8. 依赖更新
   - addBlocks：每个新阻塞调用 blockTask()
   - addBlockedBy：每个新阻塞者调用 blockTask()（反向方向）

9. 验证提醒
   - 检查 3+ 个任务是否在没有验证步骤的情况下完成
   - 设置 verificationNudgeNeeded 标志

10. 返回结果
    - { success, taskId, updatedFields, statusChange?, verificationNudgeNeeded? }
```

## 实现细节

### 字段更新逻辑

每个字段仅在满足以下条件时更新：
1. 值已提供（不是 `undefined`）
2. 值与当前任务值不同

这可以防止不必要的写入并保持 `updatedFields` 准确：
```typescript
if (subject !== undefined && subject !== existingTask.subject) {
  updates.subject = subject
  updatedFields.push('subject')
}
```

### 元数据合并

元数据与现有元数据合并，`null` 值删除键：
```typescript
const merged = { ...(existingTask.metadata ?? {}) }
for (const [key, value] of Object.entries(metadata)) {
  if (value === null) {
    delete merged[key]
  } else {
    merged[key] = value
  }
}
```

### 状态工作流

状态通过以下流程：`pending` → `in_progress` → `completed`

特殊的 `deleted` 状态从任务列表中永久删除任务。

#### 删除

当 `status === 'deleted'` 时：
```typescript
const deleted = await deleteTask(taskListId, taskId)
return { success: deleted, taskId, updatedFields: deleted ? ['deleted'] : [] }
```

提前返回 — 不更新其他字段。

#### 完成钩子

当 `status === 'completed'` 时，工具在应用状态更改之前运行 `executeTaskCompletedHooks()`：
- 生成器产生可能包含 `blockingError` 标志的结果
- 如果任何阻塞错误发生：返回 `{ success: false, error: combined messages }` 而不应用状态更改
- 这确保完成钩子可以在需要时否决状态更改

### 自动 Owner 分配（Agent Swarms）

当启用 Agent Swarms 且任务被标记为 `in_progress` 但没有明确 owner 时：
```typescript
if (
  isAgentSwarmsEnabled() &&
  status === 'in_progress' &&
  owner === undefined &&
  !existingTask.owner
) {
  updates.owner = getAgentName()
  updatedFields.push('owner')
}
```

这确保任务列表可以将 todo 项与队友匹配以显示活动状态。

### Owner 通知（Agent Swarms）

当所有权更改且启用 Agent Swarms 时：
```typescript
await writeToMailbox(
  updates.owner,
  {
    from: senderName,
    text: assignmentMessage,  // 带 type, taskId, subject, description, assignedBy, timestamp 的 JSON
    timestamp: new Date().toISOString(),
    color: senderColor,
  },
  taskListId,
)
```

这将任务分配消息发送到新 owner 的邮箱以进行异步通知。

### 依赖管理

**addBlocks**：添加在 此任务完成前无法开始的任务：
```typescript
const newBlocks = addBlocks.filter(id => !existingTask.blocks.includes(id))
for (const blockId of newBlocks) {
  await blockTask(taskListId, taskId, blockId)
}
```

**addBlockedBy**：添加必须在此任务之前完成的任务。关系以相反方向设置 — 更新阻塞者的 `blocks` 数组：
```typescript
for (const blockerId of newBlockedBy) {
  await blockTask(taskListId, blockerId, taskId)
}
```

### 验证提醒

当主线程 agent 在没有验证步骤的情况下关闭 3+ 个任务时：
```typescript
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
  !context.agentId &&
  updates.status === 'completed'
) {
  const allTasks = await listTasks(taskListId)
  const allDone = allTasks.every(t => t.status === 'completed')
  if (allDone && allTasks.length >= 3 && !allTasks.some(t => /verif/i.test(t.subject))) {
    verificationNudgeNeeded = true
  }
}
```

这在"循环退出时刻"触发 — 当最后一个任务关闭且任务循环自然结束时会显示提醒。提醒被附加到工具结果消息中。

### 结果格式化

`mapToolResultToToolResultBlockParam` 方法格式化结果：

**成功：**
```
Updated task #${taskId} ${updatedFields.join(', ')}
```

加上可选的附加项：
- **任务完成（Agent Swarms）**：`\n\nTask completed. Call TaskList now to find your next available task or see if your work unblocked others.`
- **验证提醒**：生成验证 agent 的提醒

**失败：**
```
${error || `Task #${taskId} not found`}
```

失败作为非错误工具结果返回（不会触发同级工具取消）。

## 错误处理

| 错误条件 | 处理 |
|----------------|----------|
| 任务未找到 | 返回 `{ success: false, error: 'Task not found' }`（非错误） |
| 完成钩子阻塞错误 | 返回 `{ success: false, error: combined messages }`（非错误） |
| 删除失败 | 返回 `{ success: false, error: 'Failed to delete task' }` |
| Todo V2 禁用 | 工具不可用（`isEnabled()` 返回 false） |

## 配置

### 环境变量

| 变量 | 目的 |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | 简单模式 — 可能影响工具可用性 |

### 功能标志

| 标志 | 目的 |
|------|---------|
| Todo V2 (`isTodoV2Enabled()`) | 控制工具可用性 |
| Agent Swarms (`isAgentSwarmsEnabled()`) | 启用自动 owner 分配和邮箱通知 |
| VERIFICATION_AGENT | 启用验证提醒检测 |
| `tengu_hive_evidence` | 验证提醒的附加控制 |

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `utils/tasks.ts` | `blockTask`、`deleteTask`、`getTask`、`getTaskListId`、`isTodoV2Enabled`、`listTasks`、`updateTask`、`TaskStatusSchema` |
| `utils/hooks.ts` | `executeTaskCompletedHooks`、`getTaskCompletedHookMessage` |
| `utils/teammate.ts` | `getAgentId`、`getAgentName`、`getTeammateColor`、`getTeamName` |
| `utils/teammateMailbox.ts` | `writeToMailbox` 用于 owner 通知 |
| `utils/agentSwarmsEnabled.ts` | Agent Swarms 功能检测 |
| `utils/lazySchema.ts` | 延迟模式评估 |
| `services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` |
| `tools/AgentTool/constants.ts` | `VERIFICATION_AGENT_TYPE` 常量 |

### 外部

| 包 | 目的 |
|---------|---------|
| `zod/v4` | 输入/输出模式验证 |
| `bun:bundle` | 功能标志 API（`feature()`） |

## 数据流

```
Model 请求
    |
    v
TaskUpdateTool 输入 { taskId, subject?, description?, status?, ... }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 预检查                                                           │
│ - 自动展开任务列表视图                                            │
│ - getTask() → 验证任务存在                                       │
│ - 如果未找到：返回 { success: false }                           │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 字段更新（基于差异）                                              │
│ - subject, description, activeForm, owner                       │
│ - metadata（合并，null 删除）                                    │
│ - 在 in_progress 时自动设置 owner（Agent Swarms）               │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 状态处理                                                         │
│                                                                  │
│  'deleted' → deleteTask() + 提前返回                             │
│  'completed' → executeTaskCompletedHooks()                      │
│              → 钩子错误：返回 { success: false }                │
│  其他 → 如果不同则应用                                            │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 持久化 + 通知                                                     │
│ - 如果有更改则 updateTask()                                       │
│ - writeToMailbox() 给新 owner（Agent Swarms）                   │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 依赖                                                             │
│ - addBlocks：每个新块调用 blockTask()                             │
│ - addBlockedBy：每个新阻塞者调用 blockTask()                     │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 验证检查                                                          │
│ - 所有任务完成 + 3+ 个任务 + 无验证步骤                           │
│ - 如果条件满足则设置 verificationNudgeNeeded                     │
└─────────────────────────────────────────────────────────────────┘
    |
    v
输出 { success, taskId, updatedFields, statusChange?, verificationNudgeNeeded? }
    |
    v
Model 响应："Updated task #ID fields..." [+ 提醒]
```

## 与任务系统的集成

TaskUpdateTool 是 Todo V2 任务生命周期的主要状态转换机制：

1. **状态转换**：驱动任务通过 `pending` → `in_progress` → `completed` 工作流
2. **任务删除**：特殊的 `deleted` 状态从任务列表中永久删除任务
3. **依赖图**：通过 `blockTask()` 管理 `blocks` 和 `blockedBy` 关系，同时更新关系的两端
4. **队友协调**：在 Agent Swarms 模式下，自动分配所有权并通过邮箱通知被分配者
5. **钩子集成**：执行 `TaskCreated` 和 `TaskCompleted` 钩子以实现可扩展性（通知、日志、验证）
6. **质量保证**：验证提醒在关闭多任务工作而没有验证时提醒模型生成验证 agent
7. **防止过期**：提示建议通过 TaskGet 在更新前读取任务的最新状态
8. **幂等更新**：仅写入实际更改的字段，防止不必要的磁盘 I/O 并保持审计轨迹清晰

该工具跨多个子系统协调：任务文件持久化、队友邮箱消息传递、钩子执行和依赖管理 — 使其成为 Task 工具族中最复杂的工具。
