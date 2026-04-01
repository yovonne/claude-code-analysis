# TaskOutputTool

## Purpose

TaskOutputTool retrieves output/logs from running or completed background tasks (shells, agents, remote sessions). It supports both blocking (wait for completion) and non-blocking (check current state) modes with configurable timeout. The tool is marked as deprecated in favor of using the Read tool directly on task output file paths, but remains available for backward compatibility.

## Location

- `restored-src/src/tools/TaskOutputTool/TaskOutputTool.tsx` — Main tool definition and UI (~584 lines)
- `restored-src/src/tools/TaskOutputTool/constants.ts` — Tool name constant (2 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `TaskOutputTool` | The complete tool definition built via `buildTool()` |
| `TASK_OUTPUT_TOOL_NAME` | Constant: `'TaskOutput'` |
| `Progress` | Re-exported `TaskOutputProgress` type for streaming updates |
| `TaskOutput` | Internal type: unified output covering all task types |
| `TaskOutputToolOutput` | Output type: `{ retrieval_status, task }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  task_id: string,     // Required: the task ID to get output from
  block?: boolean,     // Optional: whether to wait for completion (default: true)
  timeout?: number,    // Optional: max wait time in ms (0-600000, default: 30000)
}
```

### Output Schema

```typescript
{
  retrieval_status: 'success' | 'timeout' | 'not_ready',
  task: {
    task_id: string,
    task_type: TaskType,        // 'local_bash' | 'local_agent' | 'remote_agent' | ...
    status: string,
    description: string,
    output: string,
    exitCode?: number | null,   // For local_bash tasks
    error?: string,
    prompt?: string,            // For agent tasks
    result?: string,            // For agent tasks (clean result)
  } | null,
}
```

## Tool Definition Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'TaskOutput'` | Internal tool identifier |
| `searchHint` | `'read output/logs from a background task'` | Tool search hint |
| `maxResultSizeChars` | `100_000` | Maximum result size |
| `userFacingName()` | `'Task Output'` | Display name in UI |
| `shouldDefer` | `true` | Tool execution can be deferred |
| `aliases` | `['AgentOutputTool', 'BashOutputTool']` | Backward-compatible renamed tool aliases |
| `description()` | `'[Deprecated] — prefer Read on the task output file path'` | Marked as deprecated |
| `isConcurrencySafe()` | `true` | Read-only, safe to run concurrently |
| `isEnabled()` | `"external" !== 'ant'` | Disabled for external users |
| `isReadOnly()` | `true` | Read-only operation |
| `toAutoClassifierInput()` | `input.task_id` | Uses task_id for auto-mode classification |

## Task Output Retrieval Flow

```
1. INPUT VALIDATION
   - validateInput(): checks task_id is provided and task exists
   - Returns error code 1 if task_id missing
   - Returns error code 2 if task not found

2. NON-BLOCKING MODE (block: false)
   - If task is not running/pending: mark as notified, return current output
   - If task is still running: return retrieval_status 'not_ready'

3. BLOCKING MODE (block: true, default)
   - Show progress: "Waiting for task..."
   - Poll for task completion (100ms intervals)
   - Check abort controller for cancellation
   - On timeout: return retrieval_status 'timeout'
   - On completion: mark as notified, return output

4. OUTPUT EXTRACTION (per task type)
   - local_bash: stdout + stderr from shell command output
   - local_agent: clean result from in-memory result (preferred over disk)
   - remote_agent: command and output from remote session
```

## Implementation Details

### Input Validation

The `validateInput` method performs pre-execution checks:
- Task ID is required (error code 1)
- Task must exist in app state (error code 2)

### Non-Blocking Mode

When `block: false`:
- Returns current task state immediately
- If task is complete (not running/pending): marks task as `notified: true` and returns `retrieval_status: 'success'`
- If task is still running: returns `retrieval_status: 'not_ready'`

### Blocking Mode

When `block: true` (default):
- Shows progress message: "Waiting for task (esc to give additional instructions)"
- Polls app state every 100ms via `waitForTaskCompletion()`
- Respects `timeout` parameter (default 30s, max 600s)
- Checks `abortController.signal` for cancellation (throws `AbortError`)
- On timeout: returns `retrieval_status: 'timeout'` with current task state if available
- On completion: marks task as `notified: true`, returns `retrieval_status: 'success'`

### Task Type-Specific Output Extraction

The `getTaskOutputData()` function handles different task types:

**local_bash (shell tasks):**
- Gets stdout and stderr from `shellCommand.taskOutput`
- Falls back to disk output if taskOutput is unavailable
- Includes `exitCode` from task result

**local_agent (subagent tasks):**
- Prefers clean result from in-memory `result.content` over disk output
- Disk output is a symlink to the full session transcript (every message, tool use)
- In-memory result contains only the final assistant text content blocks
- Includes `prompt`, `result`, and `error` fields

**remote_agent (remote sessions):**
- Returns `command` as the prompt field
- Includes basic output fields

### Output Formatting

The `mapToolResultToToolResultBlockParam` method formats output as XML-like tags:
```xml
<retrieval_status>success</retrieval_status>

<task_id>...</task_id>
<task_type>...</task_type>
<status>...</status>
<exit_code>0</exit_code>
<output>
...
</output>
<error>...</error>
```

### Deprecation Notice

The tool is deprecated. The prompt instructs the model to:
- Use the Read tool on the task's output file path instead
- Background tasks return their output file path in the tool result
- Task completion sends a `<task-notification>` with the same path
- Read the file directly rather than using this tool

### UI Components

The tool includes rich UI rendering:
- `renderToolUseTag`: Shows task_id in dim color
- `renderToolUseProgressMessage`: Shows task description and waiting state
- `renderToolResultMessage`: Uses `TaskOutputResultDisplay` component with type-specific rendering
- `renderToolUseRejectedMessage`: Fallback rejection message
- `renderToolUseErrorMessage`: Fallback error message

The `TaskOutputResultDisplay` component renders differently per task type:
- **local_bash**: Uses `BashToolResultMessage` component
- **local_agent**: Shows prompt, result, and error with expand shortcut hint
- **remote_agent**: Shows description, status, and optional output
- **Other**: Shows description, status, and truncated output (first 500 chars)

## Error Handling

| Error Condition | Handling |
|----------------|----------|
| Missing task_id | validateInput returns error code 1 |
| Task not found | validateInput returns error code 2; call() throws Error |
| Task timeout | Returns `{ retrieval_status: 'timeout', task: null }` |
| Task still running (non-blocking) | Returns `{ retrieval_status: 'not_ready', task: current }` |
| Abort/cancellation | Throws `AbortError` |
| Tool disabled | Not available for external users |

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `USER_TYPE` | `'ant'` for internal users (enables tool) |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/task/diskOutput.ts` | `getTaskOutput` — reads task output from disk |
| `utils/task/framework.ts` | `updateTaskState` — marks task as notified |
| `utils/task/outputFormatting.ts` | `formatTaskOutput` — formats output for display |
| `utils/semanticBoolean.ts` | Semantic boolean parsing for `block` parameter |
| `utils/sleep.ts` | Polling interval sleep |
| `utils/slowOperations.ts` | `jsonParse` for result parsing |
| `utils/stringUtils.ts` | `countCharInString` for line counting |
| `utils/errors.ts` | `AbortError` for cancellation |
| `utils/messages.ts` | `extractTextContent` for agent result extraction |
| `tasks/types.ts` | `TaskState` type definition |
| `tasks/LocalShellTask/guards.ts` | `LocalShellTaskState` type |
| `tasks/LocalAgentTask/LocalAgentTask.ts` | `LocalAgentTaskState` type |
| `tasks/RemoteAgentTask/RemoteAgentTask.ts` | `RemoteAgentTaskState` type |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input schema validation |
| `react` | UI rendering components |
| `react/compiler-runtime` | React compiler memoization |

## Data Flow

```
Model Request
    |
    v
TaskOutputTool Input { task_id, block?, timeout? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ INPUT VALIDATION                                                 │
│ - validateInput(): task_id required, task must exist             │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ MODE CHECK                                                       │
│                                                                  │
│  block: false → Non-blocking                                     │
│    - Task complete → success + current output                    │
│    - Task running  → not_ready + current output                  │
│                                                                  │
│  block: true → Blocking                                          │
│    - Show progress: "Waiting for task..."                        │
│    - Poll every 100ms                                            │
│    - Check abort signal                                          │
│    - On timeout → timeout status                                 │
│    - On complete → success + output                              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ OUTPUT EXTRACTION (getTaskOutputData)                            │
│                                                                  │
│  local_bash:   stdout + stderr + exitCode                        │
│  local_agent:  clean result (in-memory preferred) + prompt       │
│  remote_agent: command + output                                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ NOTIFICATION                                                     │
│ - Mark task as notified (updateTaskState)                        │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { retrieval_status, task: { task_id, task_type, status, ... } }
    |
    v
Model Response: XML-formatted task output
```

## Integration with Task System

TaskOutputTool bridges the task execution and result retrieval layers:

1. **Background Task Monitoring**: Provides a way to check on tasks spawned by BashTool (background mode), AgentTool, or remote sessions
2. **Task Notification Suppression**: Marks tasks as `notified: true` after output is retrieved, preventing redundant completion notifications
3. **Type-Agnostic Output**: Works with all task types (`local_bash`, `local_agent`, `remote_agent`, etc.) through a unified interface
4. **Blocking/Non-Blocking Flexibility**: Supports both wait-for-completion and check-current-state patterns
5. **Deprecation Path**: The tool is being phased out in favor of direct file reads. Tasks now return their output file path in the tool result, and completion notifications include the same path, allowing the model to read output directly via the Read tool

The tool reads from both in-memory task state and disk output files, preferring clean in-memory results for agent tasks (to avoid raw transcript noise).
