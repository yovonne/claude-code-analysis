# Task System (tasks/)

## Purpose

The `tasks/` directory contains the task state type definitions, task lifecycle management, and shared task operations used across all task-related tools. It defines the union of all concrete task state types, provides the `stopTask()` function for terminating running tasks, and contains task-type-specific implementations for local shell tasks, local agent tasks, remote agent tasks, in-process teammate tasks, workflow tasks, monitor MCP tasks, and dream tasks.

## Location

- `restored-src/src/tasks/types.ts` — Task state type definitions (47 lines)
- `restored-src/src/tasks/stopTask.ts` — Shared task termination logic (101 lines)
- `restored-src/src/tasks/LocalShellTask/` — Background shell task implementation
- `restored-src/src/tasks/LocalAgentTask/` — Local subagent task implementation
- `restored-src/src/tasks/RemoteAgentTask/` — Remote agent session implementation
- `restored-src/src/tasks/InProcessTeammateTask/` — Teammate task implementation
- `restored-src/src/tasks/LocalWorkflowTask/` — Workflow task implementation
- `restored-src/src/tasks/MonitorMcpTask/` — Monitor MCP task implementation
- `restored-src/src/tasks/DreamTask/` — Dream task implementation
- `restored-src/src/tasks/pillLabel.ts` — Task label formatting

## Key Exports

### From types.ts

| Export | Description |
|--------|-------------|
| `TaskState` | Union of all concrete task state types |
| `BackgroundTaskState` | Task types that appear in the background tasks indicator |
| `isBackgroundTask(task)` | Type guard: checks if task should show in background indicator |

### From stopTask.ts

| Export | Description |
|--------|-------------|
| `StopTaskError` | Error class with typed codes: `not_found`, `not_running`, `unsupported_type` |
| `stopTask(taskId, context)` | Look up, validate, kill, and notify task termination |

## Task State Types

### TaskState Union

```typescript
type TaskState =
  | LocalShellTaskState        // Background shell commands
  | LocalAgentTaskState        // Local subagent executions
  | RemoteAgentTaskState       // Remote agent sessions
  | InProcessTeammateTaskState // Teammate task assignments
  | LocalWorkflowTaskState     // Workflow executions
  | MonitorMcpTaskState        // Monitor MCP tasks
  | DreamTaskState             // Dream tasks
```

### BackgroundTaskState

Tasks that can appear in the background tasks indicator:

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

### Background Task Detection

A task is considered a background task when:

```typescript
function isBackgroundTask(task: TaskState): task is BackgroundTaskState
```

Conditions:
1. Task status is `'running'` or `'pending'`
2. Task is not explicitly foreground (`isBackgrounded !== false`)

## stopTask() Implementation

### Purpose

Shared logic for stopping a running background task. Used by both:
- `TaskStopTool` (LLM-invoked)
- SDK `stop_task` control request

### Function Signature

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

### Execution Flow

```
1. TASK LOOKUP
   - Get task from appState.tasks[taskId]
   - If not found: throw StopTaskError(code: 'not_found')

2. STATUS VALIDATION
   - Check task.status === 'running'
   - If not running: throw StopTaskError(code: 'not_running')

3. TASK IMPLEMENTATION RESOLUTION
   - getTaskByType(task.type) → task implementation
   - If unsupported type: throw StopTaskError(code: 'unsupported_type')

4. TASK KILLING
   - taskImpl.kill(taskId, setAppState)
   - Delegates to task-type-specific kill logic

5. NOTIFICATION SUPPRESSION (Shell Tasks Only)
   - Mark local shell tasks as notified (suppress "exit code 137" noise)
   - Agent tasks are NOT suppressed — they send useful AbortError payloads

6. SDK EVENT EMISSION
   - If notification was suppressed, emit TaskTerminated SDK event directly
   - Ensures SDK consumers still see the task close

7. RESULT RETURNED
   - { taskId, taskType, command }
   - command = task.command (shell) or task.description (other)
```

### StopTaskError

Typed error class for stop failures:

```typescript
class StopTaskError extends Error {
  code: 'not_found' | 'not_running' | 'unsupported_type'
}
```

| Code | Condition |
|------|-----------|
| `not_found` | No task exists with the given ID |
| `not_running` | Task exists but status is not `'running'` |
| `unsupported_type` | Task type has no registered implementation |

### Notification Suppression Logic

Shell tasks produce noisy "exit code 137" (SIGKILL) notifications when killed. The suppression logic:

1. Checks if task is a `LocalShellTask` via `isLocalShellTask()` guard
2. Marks the task as `notified: true` to suppress the XML notification
3. Emits a `TaskTerminated` SDK event directly so SDK consumers still receive the close event

Agent tasks are intentionally NOT suppressed — their `AbortError` catch handlers send notifications with `extractPartialResult(agentMessages)`, which carries useful partial results rather than noise.

## Task Type Implementations

### LocalShellTask

Background shell command execution:
- Manages shell process lifecycle
- Handles output streaming to disk
- Supports foreground/background transitions
- `killShellTasks.ts` — Bulk shell task termination
- `guards.ts` — Type guards (`isLocalShellTask`, `LocalShellTaskState`)

### LocalAgentTask

Local subagent execution:
- Manages subagent session lifecycle
- Stores clean result in memory (separate from disk transcript)
- Handles abort/cancellation with partial result extraction

### RemoteAgentTask

Remote agent session management:
- Connects to remote agent endpoints
- Manages session state and output

### InProcessTeammateTask

Teammate task execution:
- Runs teammate tasks within the same process
- Supports task assignment and completion workflows
- `types.ts` — Teammate-specific type definitions

### LocalWorkflowTask

Workflow execution:
- Manages multi-step workflow state
- Tracks workflow progress and results

### MonitorMcpTask

Monitor MCP (Model Context Protocol) task:
- Integrates with MCP monitoring capabilities
- Tracks monitoring state and alerts

### DreamTask

Dream task implementation:
- Specialized task type for dream mode operations

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `state/AppState.ts` | `AppState` type definition |
| `Task.ts` | `TaskStateBase` type |
| `tasks.ts` | `getTaskByType()` — task implementation registry |
| `utils/sdkEventQueue.ts` | `emitTaskTerminatedSdk()` — SDK event emission |
| `LocalShellTask/guards.ts` | `isLocalShellTask()` type guard |

### External

| Package | Purpose |
|---------|---------|
| TypeScript | Type system for union types and type guards |

## Integration with Task Tools

The tasks/ directory provides the foundation layer for all task-related tools:

1. **TaskCreateTool**: Creates tasks that become `TaskState` entries in app state
2. **TaskGetTool**: Reads task state from the task file system (persisted state)
3. **TaskListTool**: Lists all tasks, filtering by metadata and resolving dependencies
4. **TaskUpdateTool**: Modifies task state and triggers hooks
5. **TaskStopTool**: Uses `stopTask()` to terminate running tasks
6. **TaskOutputTool**: Reads output from running/completed tasks by type

### Task Lifecycle

```
Created (TaskCreateTool)
    |
    v
Pending → In Progress (TaskUpdateTool: status='in_progress')
    |                           |
    v                           v
Running (background tasks)  Completed (TaskUpdateTool: status='completed')
    |                           |
    v                           v
Stopped (TaskStopTool)      Archived/Deleted (TaskUpdateTool: status='deleted')
```

### State Persistence

Tasks exist in two layers:
1. **Disk persistence**: Task files in the task list directory (Todo V2 system)
2. **In-memory state**: `appState.tasks` record for runtime access

Tools that modify task state (TaskCreateTool, TaskUpdateTool) update both layers. Read-only tools (TaskGetTool, TaskListTool) read from disk to ensure current state.
