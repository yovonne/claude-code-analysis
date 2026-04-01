# TaskCreateTool

## 用途

TaskCreateTool 在任务列表（Todo V2 系统）中创建新的结构化任务。它允许模型将复杂工作分解为可跟踪、有序的任务，具有标题、描述、可选的用于微调器的主动形式以及任意元数据。任务默认以 `pending` 状态创建，无负责人。当 Agent Swarms 启用时，任务可以随后分配给队友。

## 位置

- `restored-src/src/tools/TaskCreateTool/TaskCreateTool.ts` — 主要工具定义（约 139 行）
- `restored-src/src/tools/TaskCreateTool/constants.ts` — 工具名称常量（2 行）
- `restored-src/src/tools/TaskCreateTool/prompt.ts` — 工具提示和描述（57 行）

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `TaskCreateTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `TASK_CREATE_TOOL_NAME` | 常量：`'TaskCreate'` |
| `DESCRIPTION` | 常量：`'Create a new task in the task list'` |
| `getPrompt()` | 生成使用提示，针对 Agent Swarms 上下文进行调整 |
| `Output` | Zod 推断的输出类型：`{ task: { id: string, subject: string } }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  subject: string,        // 必需：任务的简短标题（祈使形式）
  description: string,    // 必需：需要完成的工作
  activeForm?: string,    // 可选：用于微调器的现在进行时形式（例如 "Running tests"）
  metadata?: Record<string, unknown>  // 可选：任意键值元数据
}
```

### 输出 Schema

```typescript
{
  task: {
    id: string,           // 生成的任务 ID
    subject: string,      // 任务标题
  }
}
```

## 工具定义属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'TaskCreate'` | 内部工具标识符 |
| `searchHint` | `'create a task in the task list'` | 工具搜索提示 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `userFacingName()` | `'TaskCreate'` | UI 中的显示名称 |
| `shouldDefer` | `true` | 工具执行可以延迟 |
| `isEnabled()` | `isTodoV2Enabled()` | 仅在 Todo V2 功能启用时活动 |
| `isConcurrencySafe()` | `true` | 可安全并发运行 |
| `toAutoClassifierInput()` | `input.subject` | 使用 subject 进行自动模式分类 |
| `renderToolUseMessage()` | `null` | 无自定义工具使用消息渲染 |

## 任务创建流程

```
1. 接收输入
   - 模型返回带有 { subject, description, activeForm?, metadata? } 的 TaskCreate tool_use

2. 任务创建
   - getTaskListId() 解析当前任务列表
   - createTask() 创建具有以下内容的任务文件：
     - subject, description, activeForm
     - status: 'pending'
     - owner: undefined
     - blocks: []
     - blockedBy: []
     - metadata（如果提供）
   - 返回生成的 taskId

3. 创建后钩子
   - executeTaskCreatedHooks() 异步运行
   - 生成器产生结果；收集阻塞错误
   - 如果发生任何阻塞错误：
     - 通过 deleteTask() 删除任务
     - 抛出错误

4. UI 更新
   - 在应用状态中自动展开任务列表视图
   - 如果尚未展开则设置 expandedView 为 'tasks'

5. 返回结果
   - { task: { id: taskId, subject } }
```

## 实现细节

### 任务创建

工具委托给 `utils/tasks.ts` 中的 `createTask()`，它将任务文件写入任务列表目录。所有任务以以下状态开始：
- 状态：`pending`
- 负责人：`undefined`（未分配）
- 无依赖（`blocks: []`、`blockedBy: []`）

### 创建后钩子

任务创建后，`executeTaskCreatedHooks()` 运行一个执行副作用的生成器（例如通知、日志记录）。生成器产生可能包含 `blockingError` 标志的结果。如果遇到任何阻塞错误：
1. 收集所有阻塞错误消息
2. 删除新创建的任务（回滚）
3. 抛出带有组合消息的 `Error`

这确保任务创建是原子的——如果钩子失败，不会留下孤立任务。

### UI 自动展开

创建任务时，工具通过更新应用状态自动展开任务列表视图：
```typescript
context.setAppState(prev => {
  if (prev.expandedView === 'tasks') return prev
  return { ...prev, expandedView: 'tasks' as const }
})
```

这为用户提供即时的视觉反馈。

### 结果格式化

`mapToolResultToToolResultBlockParam` 方法将结果格式化为人类可读的字符串：
```
Task #${task.id} created successfully: ${task.subject}
```

### Agent Swarms 集成

当 `isAgentSwarmsEnabled()` 返回 true 时，提示包括额外指导：
- 描述应包含足够详细的信息供另一个代理理解
- 新任务创建时没有负责人——使用 `TaskUpdate` 分配它们
- 有关分配任务给队友的提示

## 错误处理

| 错误条件 | 处理 |
|----------------|----------|
| 创建后钩子阻塞错误 | 任务被删除，抛出带有组合消息的错误 |
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
| Agent Swarms（`isAgentSwarmsEnabled()`） | 在提示中启用队友分配功能 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/tasks.ts` | `createTask`、`deleteTask`、`getTaskListId`、`isTodoV2Enabled` |
| `utils/hooks.ts` | `executeTaskCreatedHooks`、`getTaskCreatedHookMessage` |
| `utils/teammate.ts` | `getAgentName`、`getTeamName` |
| `utils/lazySchema.ts` | 延迟 schema 评估 |
| `utils/agentSwarmsEnabled.ts` | Agent Swarms 功能检测 |

### 外部

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入/输出 schema 验证 |
| `bun:bundle` | 功能标志 API（通过钩子） |

## 数据流

```
Model Request
    |
    v
TaskCreateTool Input { subject, description, activeForm?, metadata? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ TASK CREATION                                                    │
│ - getTaskListId() → resolve task list                            │
│ - createTask() → write task file with pending status             │
│ - Returns taskId                                                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ POST-CREATION HOOKS                                              │
│ - executeTaskCreatedHooks() generator                            │
│ - Collect blocking errors                                        │
│ - On error: deleteTask() + throw                                │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ UI UPDATE                                                        │
│ - Auto-expand task list view                                     │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { task: { id, subject } }
    |
    v
Model Response: "Task #${id} created successfully: ${subject}"
```

## 与任务系统的集成

TaskCreateTool 是 Todo V2 任务生命周期的入口点：

1. **任务创建**：在任务列表目录中创建具有 `pending` 状态的任务文件
2. **任务生命周期**：创建的任务通过 TaskUpdateTool 流经：`pending` → `in_progress` → `completed`
3. **依赖**：任务可以在创建后通过 TaskUpdateTool 设置 `blocks` 和 `blockedBy` 关系
4. **分配**：任务可以通过 TaskUpdateTool 分配给负责人（代理/队友）
5. **可见性**：创建的任务出现在 TaskListTool 输出中，可以通过 TaskGetTool 单独检索

该工具创建的任务作为文件持久化在磁盘上的任务列表目录结构中，支持会话持久化和恢复功能。
