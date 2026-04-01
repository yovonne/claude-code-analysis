# TaskStopTool

## Purpose

TaskStopTool stops (kills) a running background task by its ID. It supports all background task types including shell tasks, agent tasks, and remote sessions. The tool also maintains backward compatibility with the deprecated `KillShell` tool name via an alias, ensuring existing transcripts and SDK users continue to work.

## Location

- `restored-src/src/tools/TaskStopTool/TaskStopTool.ts` — Main tool definition (~132 lines)
- `restored-src/src/tools/TaskStopTool/prompt.ts` — Tool name constant and description (9 lines)
- `restored-src/src/tools/TaskStopTool/UI.tsx` — UI rendering functions (41 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `TaskStopTool` | The complete tool definition built via `buildTool()` |
| `TASK_STOP_TOOL_NAME` | Constant: `'TaskStop'` (from prompt.ts) |
| `DESCRIPTION` | Tool description string (from prompt.ts) |
| `renderToolUseMessage` | Returns empty string (no tool use message) |
| `renderToolResultMessage` | Renders stopped task display with truncated command |
| `Output` | Zod-inferred output type: `{ message, task_id, task_type, command? }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  task_id?: string,   // Optional: the ID of the background task to stop
  shell_id?: string,  // Optional: deprecated, use task_id instead (KillShell compat)
}
```

At least one of `task_id` or `shell_id` must be provided.

### Output Schema

```typescript
{
  message: string,     // Status message about the operation
  task_id: string,     // The ID of the task that was stopped
  task_type: string,   // The type of the task that was stopped
  command?: string,    // The command or description of the stopped task
}
```

## Tool Definition Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'TaskStop'` | Internal tool identifier |
| `searchHint` | `'kill a running background task'` | Tool search hint |
| `aliases` | `['KillShell']` | Backward-compatible deprecated name |
| `maxResultSizeChars` | `100_000` | Maximum result size |
| `userFacingName()` | `''` for ant, `'Stop Task'` otherwise | Display name in UI |
| `shouldDefer` | `true` | Tool execution can be deferred |
| `isConcurrencySafe()` | `true` | Safe to run concurrently |
| `toAutoClassifierInput()` | `input.task_id ?? input.shell_id ?? ''` | Uses ID for auto-mode classification |

## Task Stop Flow

```
1. INPUT VALIDATION
   - validateInput(): checks task_id or shell_id is provided
   - Resolves id = task_id ?? shell_id
   - Returns error code 1 if ID missing
   - Returns error code 1 if task not found
   - Returns error code 3 if task is not running

2. TASK STOPPING
   - stopTask(id, { getAppState, setAppState }) is called
   - Delegates to the task's specific kill mechanism
   - Returns { taskId, taskType, command }

3. RESULT RETURNED
   - { message, task_id, task_type, command }
```

## Implementation Details

### ID Resolution

The tool accepts both `task_id` and `shell_id` parameters:
```typescript
const id = task_id ?? shell_id
```

`shell_id` is accepted for backward compatibility with the deprecated `KillShell` tool. New code should use `task_id`.

### Input Validation

The `validateInput` method performs three checks:
1. **Missing ID**: Either `task_id` or `shell_id` must be provided (error code 1)
2. **Task not found**: The ID must correspond to an existing task in app state (error code 1)
3. **Task not running**: The task must have status `'running'` (error code 3)

Attempting to stop a non-running task (pending, completed, etc.) is rejected.

### Task Stopping

The tool delegates to `stopTask()` from `tasks/stopTask.ts`:
```typescript
const result = await stopTask(id, {
  getAppState,
  setAppState,
})
```

The `stopTask` function handles the actual termination logic, which varies by task type:
- **Shell tasks**: Kills the underlying shell process
- **Agent tasks**: Aborts the agent execution
- **Remote tasks**: Terminates the remote session

### Result Formatting

The `mapToolResultToToolResultBlockParam` method returns JSON-stringified output:
```typescript
jsonStringify(output)
```

This produces a compact JSON representation of the result object.

### UI Rendering

**Tool Use Message:**
Returns an empty string — no special tool use message is displayed.

**Tool Result Message:**
Renders the stopped command with a "· stopped" suffix:
- Command is truncated to 2 lines max and 160 chars max in non-verbose mode
- Verbose mode shows the full command
- Truncated commands show `… · stopped`, non-truncated show ` · stopped`

Example output:
```
npm run build · stopped
```

## Error Handling

| Error Condition | Handling |
|----------------|----------|
| Missing task_id and shell_id | validateInput returns error code 1 |
| Task not found | validateInput returns error code 1 |
| Task not running | validateInput returns error code 3 |
| call() missing ID | Throws `Error('Missing required parameter: task_id')` |

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `USER_TYPE` | `'ant'` for internal users (affects userFacingName) |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `tasks/stopTask.ts` | `stopTask` — actual task termination logic |
| `utils/lazySchema.ts` | Lazy schema evaluation |
| `utils/slowOperations.ts` | `jsonStringify` for result formatting |
| `ink.js` | `Text` component for UI |
| `ink/stringWidth.js` | `stringWidth` for command truncation |
| `utils/format.js` | `truncateToWidthNoEllipsis` for command truncation |
| `components/MessageResponse.js` | `MessageResponse` component |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |

## Data Flow

```
Model Request
    |
    v
TaskStopTool Input { task_id?, shell_id? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ INPUT VALIDATION                                                 │
│ - Resolve id = task_id ?? shell_id                               │
│ - Check id is provided (error 1)                                 │
│ - Check task exists in app state (error 1)                       │
│ - Check task status is 'running' (error 3)                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ TASK STOPPING                                                    │
│ - stopTask(id, { getAppState, setAppState })                     │
│ - Delegates to task-type-specific kill mechanism                 │
│ - Returns { taskId, taskType, command }                          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                                │
│ - message: "Successfully stopped task: ${id} (${command})"       │
│ - JSON-stringified for model response                            │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { message, task_id, task_type, command? }
    |
    v
Model Response: JSON result
UI Display: "${truncated_command} · stopped"
```

## Integration with Task System

TaskStopTool provides task termination within the background task management system:

1. **Background Task Lifecycle**: Completes the lifecycle for background tasks by providing a way to cancel them before natural completion
2. **KillShell Compatibility**: Maintains the `KillShell` alias so existing session transcripts that reference the old tool name continue to work during replay
3. **Task State Management**: Uses `getAppState` and `setAppState` to read and update task state during termination
4. **Type-Agnostic Stopping**: Works with all background task types through the unified `stopTask()` function, which dispatches to the appropriate kill mechanism based on task type
5. **Running-Only Guard**: Only allows stopping tasks that are actively running — pending or completed tasks cannot be stopped

The tool delegates all actual termination logic to `stopTask()` in `tasks/stopTask.ts`, which handles the platform-specific and task-type-specific details of process termination.
