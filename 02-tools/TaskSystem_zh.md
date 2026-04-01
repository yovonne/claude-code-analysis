# 任务系统 (tasks/)

## 目的

`tasks/` 目录包含任务状态类型定义、任务生命周期管理以及跨所有任务相关工具的共享操作。它定义了所有具体任务状态类型的联合体，提供 `stopTask()` 函数用于终止运行中的任务，并包含针对本地 shell 任务、本地 agent 任务、远程 agent 任务、进程内队友任务、工作流任务、Monitor MCP 任务和 dream 任务的特定实现。

## 位置

- `restored-src/src/tasks/types.ts` — 任务状态类型定义（47 行）
- `restored-src/src/tasks/stopTask.ts` — 共享任务终止逻辑（101 行）
- `restored-src/src/tasks/LocalShellTask/` — 后台 shell 任务实现
- `restored-src/src/tasks/LocalAgentTask/` — 本地子 agent 任务实现
- `restored-src/src/tasks/RemoteAgentTask/` — 远程 agent 会话实现
- `restored-src/src/tasks/InProcessTeammateTask/` — 队友任务实现
- `restored-src/src/tasks/LocalWorkflowTask/` — 工作流任务实现
- `restored-src/src/tasks/MonitorMcpTask/` — Monitor MCP 任务实现
- `restored-src/src/tasks/DreamTask/` — Dream 任务实现
- `restored-src/src/tasks/pillLabel.ts` — 任务标签格式化

## 关键导出

### 来自 types.ts

| 导出 | 描述 |
|--------|-------------|
| `TaskState` | 所有具体任务状态类型的联合 |
| `BackgroundTaskState` | 出现在后台任务指示器中的任务类型 |
| `isBackgroundTask(task)` | 类型守卫：检查任务是否应显示在后台指示器中 |

### 来自 stopTask.ts

| 导出 | 描述 |
|--------|-------------|
| `StopTaskError` | 带类型代码的错误类：`not_found`、`not_running`、`unsupported_type` |
| `stopTask(taskId, context)` | 查找、验证、终止并通知任务终止 |

## 任务状态类型

### TaskState 联合类型

```typescript
type TaskState =
  | LocalShellTaskState        // 后台 shell 命令
  | LocalAgentTaskState        // 本地子 agent 执行
  | RemoteAgentTaskState       // 远程 agent 会话
  | InProcessTeammateTaskState // 进程内队友任务
  | LocalWorkflowTaskState     // 工作流执行
  | MonitorMcpTaskState        // Monitor MCP 任务
  | DreamTaskState             // Dream 任务
```

### BackgroundTaskState

可以出现在后台任务指示器中的任务：

```typescript
type BackgroundTaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

### 后台任务检测

当满足以下条件时，任务被视为后台任务：

```typescript
function isBackgroundTask(task: TaskState): task is BackgroundTaskState
```

条件：
1. 任务状态为 `'running'` 或 `'pending'`
2. 任务未明确设置为前台（`isBackgrounded !== false`）

## stopTask() 实现

### 目的

用于停止运行中后台任务的共享逻辑。被以下两者使用：
- `TaskStopTool`（LLM 调用）
- SDK `stop_task` 控制请求

### 函数签名

```typescript
async function stopTask(
  taskId: string,
  context: {
    getAppState: () => AppState,
    setAppState: (f: (prev: AppState) => AppState) => void,
  }
): Promise<{
  taskId: string,
  taskType: string,
  command: string | undefined,
}>
```

### 执行流程

```
1. 任务查找
   - 从 appState.tasks[taskId] 获取任务
   - 如果未找到：抛出 StopTaskError(code: 'not_found')

2. 状态验证
   - 检查 task.status === 'running'
   - 如果未运行：抛出 StopTaskError(code: 'not_running')

3. 任务实现解析
   - getTaskByType(task.type) → 任务实现
   - 如果不支持的类型：抛出 StopTaskError(code: 'unsupported_type')

4. 任务终止
   - taskImpl.kill(taskId, setAppState)
   - 委托给特定于任务类型的终止逻辑

5. 通知抑制（仅限 Shell 任务）
   - 将本地 shell 任务标记为已通知（抑制"exit code 137"噪音）
   - Agent 任务不抑制 — 它们发送有用的 AbortError 载荷

6. SDK 事件发送
   - 如果通知被抑制，直接发送 TaskTerminated SDK 事件
   - 确保 SDK 消费者仍能看到任务关闭

7. 返回结果
   - { taskId, taskType, command }
   - command = task.command (shell) 或 task.description (其他)
```

### StopTaskError

停止失败的类型化错误类：

```typescript
class StopTaskError extends Error {
  code: 'not_found' | 'not_running' | 'unsupported_type'
}
```

| 代码 | 条件 |
|------|-----------|
| `not_found` | 没有给定 ID 的任务存在 |
| `not_running` | 任务存在但状态不是 `'running'` |
| `unsupported_type` | 任务类型没有注册的实现 |

### 通知抑制逻辑

Shell 任务在被终止时会产生嘈杂的"exit code 137"（SIGKILL）通知。抑制逻辑：

1. 通过 `isLocalShellTask()` 守卫检查任务是否为 `LocalShellTask`
2. 将任务标记为 `notified: true` 以抑制 XML 通知
3. 直接发送 `TaskTerminated` SDK 事件，以便 SDK 消费者仍能收到关闭事件

Agent 任务故意不抑制 — 它们的 `AbortError` 捕获处理程序发送带有 `extractPartialResult(agentMessages)` 的通知，这包含有用的部分结果而非噪音。

## 任务类型实现

### LocalShellTask

后台 shell 命令执行：
- 管理 shell 进程生命周期
- 处理输出流式写入磁盘
- 支持前台/后台转换
- `killShellTasks.ts` — 批量 shell 任务终止
- `guards.ts` — 类型守卫（`isLocalShellTask`、`LocalShellTaskState`）

### LocalAgentTask

本地子 agent 执行：
- 管理子 agent 会话生命周期
- 将干净结果存储在内存中（与磁盘 transcript 分离）
- 处理中止/取消并提取部分结果

### RemoteAgentTask

远程 agent 会话管理：
- 连接到远程 agent 端点
- 管理会话状态和输出

### InProcessTeammateTask

队友任务执行：
- 在同一进程内运行队友任务
- 支持任务分配和完成工作流
- `types.ts` — 队友特定类型定义

### LocalWorkflowTask

工作流执行：
- 管理多步骤工作流状态
- 跟踪工作流进度和结果

### MonitorMcpTask

Monitor MCP（Model Context Protocol）任务：
- 与 MCP 监控功能集成
- 跟踪监控状态和警报

### DreamTask

Dream 任务实现：
- 用于 dream 模式操作的专用任务类型

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `state/AppState.ts` | `AppState` 类型定义 |
| `Task.ts` | `TaskStateBase` 类型 |
| `tasks.ts` | `getTaskByType()` — 任务实现注册表 |
| `utils/sdkEventQueue.ts` | `emitTaskTerminatedSdk()` — SDK 事件发送 |
| `LocalShellTask/guards.ts` | `isLocalShellTask()` 类型守卫 |

### 外部

| 包 | 目的 |
|---------|---------|
| TypeScript | 用于联合类型和类型守卫的类型系统 |

## 与任务工具的集成

tasks/ 目录为所有任务相关工具提供基础层：

1. **TaskCreateTool**：创建成为 app state 中 `TaskState` 条目的任务
2. **TaskGetTool**：从任务文件系统读取任务状态（持久化状态）
3. **TaskListTool**：列出所有任务，按元数据过滤并解析依赖
4. **TaskUpdateTool**：修改任务状态并触发钩子
5. **TaskStopTool**：使用 `stopTask()` 终止运行中的任务
6. **TaskOutputTool**：按类型读取运行中/已完成任务的输出

### 任务生命周期

```
创建 (TaskCreateTool)
    |
    v
Pending → In Progress (TaskUpdateTool: status='in_progress')
    |                           |
    v                           v
Running (后台任务)           Completed (TaskUpdateTool: status='completed')
    |                           |
    v                           v
Stopped (TaskStopTool)      Archived/Deleted (TaskUpdateTool: status='deleted')
```

### 状态持久化

任务存在于两层：
1. **磁盘持久化**：任务列表目录中的任务文件（Todo V2 系统）
2. **内存状态**：`appState.tasks` 记录用于运行时访问

修改任务状态的工具（TaskCreateTool、TaskUpdateTool）会更新两层。只读工具（TaskGetTool、TaskListTool）从磁盘读取以确保当前状态。
