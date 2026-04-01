# TaskGetTool

## Purpose

TaskGetTool retrieves a single task by its ID from the task list (Todo V2 system). It returns full task details including subject, description, status, and dependency information (blocks/blockedBy). This is the read-only lookup tool for individual tasks, complementing TaskListTool which provides summary views of all tasks.

## Location

- `restored-src/src/tools/TaskGetTool/TaskGetTool.ts` — Main tool definition (~129 lines)
- `restored-src/src/tools/TaskGetTool/constants.ts` — Tool name constant (2 lines)
- `restored-src/src/tools/TaskGetTool/prompt.ts` — Tool prompt and description (25 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `TaskGetTool` | The complete tool definition built via `buildTool()` |
| `TASK_GET_TOOL_NAME` | Constant: `'TaskGet'` |
| `DESCRIPTION` | Constant: `'Get a task by ID from the task list'` |
| `PROMPT` | Usage instructions for the model |
| `Output` | Zod-inferred output type: `{ task: Task \| null }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  taskId: string,    // Required: the ID of the task to retrieve
}
```

### Output Schema

```typescript
{
  task: {
    id: string,           // Task identifier
    subject: string,      // Task title
    description: string,  // Detailed requirements/context
    status: TaskStatus,   // 'pending' | 'in_progress' | 'completed'
    blocks: string[],     // Task IDs that this task blocks
    blockedBy: string[],  // Task IDs that block this task
  } | null                // null if task not found
}
```

## Tool Definition Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'TaskGet'` | Internal tool identifier |
| `searchHint` | `'retrieve a task by ID'` | Tool search hint |
| `maxResultSizeChars` | `100_000` | Maximum result size |
| `userFacingName()` | `'TaskGet'` | Display name in UI |
| `shouldDefer` | `true` | Tool execution can be deferred |
| `isEnabled()` | `isTodoV2Enabled()` | Only active when Todo V2 is enabled |
| `isConcurrencySafe()` | `true` | Safe to run concurrently |
| `isReadOnly()` | `true` | Read-only operation (no side effects) |
| `toAutoClassifierInput()` | `input.taskId` | Uses taskId for auto-mode classification |
| `renderToolUseMessage()` | `null` | No custom tool use message rendering |

## Task Retrieval Flow

```
1. INPUT RECEIVED
   - Model returns TaskGet tool_use with { taskId }

2. TASK LOOKUP
   - getTaskListId() resolves the current task list
   - getTask(taskListId, taskId) reads the task file
   - Returns task object or null if not found

3. RESULT FORMATTING
   - If task not found: { task: null }
   - If found: { task: { id, subject, description, status, blocks, blockedBy } }
```

## Implementation Details

### Task Lookup

The tool delegates to `getTask()` from `utils/tasks.ts`, which reads the task file from the task list directory. The task file contains all task fields persisted as JSON.

### Null Handling

If the task ID does not correspond to an existing task, the tool returns `{ task: null }` rather than throwing an error. This is a benign condition — it allows the model to handle missing tasks gracefully (e.g., task was already deleted or the ID was incorrect).

### Result Formatting

The `mapToolResultToToolResultBlockParam` method formats the result as human-readable text:

**Task found:**
```
Task #${task.id}: ${task.subject}
Status: ${task.status}
Description: ${task.description}
Blocked by: #${id1}, #${id2}     // Only if blockedBy is non-empty
Blocks: #${id1}, #${id2}         // Only if blocks is non-empty
```

**Task not found:**
```
Task not found
```

### Read-Only Classification

The tool implements `isReadOnly() => true`, marking it as a read-only operation. This means:
- It does not modify any state
- It can be safely called without side effects
- It may be auto-approved in certain permission modes

## Error Handling

| Error Condition | Handling |
|----------------|----------|
| Task ID not found | Returns `{ task: null }` — not an error |
| Todo V2 disabled | Tool is not available (`isEnabled()` returns false) |

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | Simple mode — may affect tool availability |

### Feature Flags

| Flag | Purpose |
|------|---------|
| Todo V2 (`isTodoV2Enabled()`) | Gates tool availability |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/tasks.ts` | `getTask`, `getTaskListId`, `isTodoV2Enabled`, `TaskStatusSchema` |
| `utils/lazySchema.ts` | Lazy schema evaluation |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |

## Data Flow

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
│ - getTask(taskListId, taskId) → read task file                   │
│ - Returns task object or null                                    │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                                │
│ - Task found: format with subject, status, description, deps     │
│ - Task not found: "Task not found"                               │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { task: { id, subject, description, status, blocks, blockedBy } | null }
    |
    v
Model Response: formatted task details or "Task not found"
```

## Integration with Task System

TaskGetTool provides individual task access within the Todo V2 ecosystem:

1. **Complement to TaskListTool**: While TaskListTool shows summaries of all tasks, TaskGetTool retrieves full details for a specific task
2. **Pre-Update Verification**: The prompt advises reading a task's latest state via TaskGet before updating it (staleness prevention)
3. **Dependency Awareness**: Returns `blocks` and `blockedBy` arrays so the model can understand task dependency chains
4. **Status Checking**: Allows the model to verify a task's current status before attempting work or updates
5. **Assignment Context**: Returns the full description field which may contain detailed requirements for assigned work

The tool reads task files directly from disk, ensuring it always returns the most current state regardless of in-memory caching.
