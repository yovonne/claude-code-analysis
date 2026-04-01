# TaskListTool

## Purpose

TaskListTool lists all tasks in the task list (Todo V2 system) with summary information. It provides a compact overview of each task including ID, subject, status, owner, and unresolved dependencies. Internal/metadata tasks are automatically filtered out. Resolved (completed) dependencies are also filtered from the `blockedBy` lists to keep the output focused on actionable blockers.

## Location

- `restored-src/src/tools/TaskListTool/TaskListTool.ts` — Main tool definition (~117 lines)
- `restored-src/src/tools/TaskListTool/constants.ts` — Tool name constant (2 lines)
- `restored-src/src/tools/TaskListTool/prompt.ts` — Tool prompt and description (50 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `TaskListTool` | The complete tool definition built via `buildTool()` |
| `TASK_LIST_TOOL_NAME` | Constant: `'TaskList'` |
| `DESCRIPTION` | Constant: `'List all tasks in the task list'` |
| `getPrompt()` | Generates the usage prompt, adapting for Agent Swarms context |
| `Output` | Zod-inferred output type: `{ tasks: TaskSummary[] }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  // No parameters — lists all tasks
}
```

### Output Schema

```typescript
{
  tasks: {
    id: string,           // Task identifier
    subject: string,      // Brief task description
    status: TaskStatus,   // 'pending' | 'in_progress' | 'completed'
    owner?: string,       // Agent ID if assigned, absent if unassigned
    blockedBy: string[],  // Open task IDs that must complete first (resolved deps filtered)
  }[]
}
```

## Tool Definition Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'TaskList'` | Internal tool identifier |
| `searchHint` | `'list all tasks'` | Tool search hint |
| `maxResultSizeChars` | `100_000` | Maximum result size |
| `userFacingName()` | `'TaskList'` | Display name in UI |
| `shouldDefer` | `true` | Tool execution can be deferred |
| `isEnabled()` | `isTodoV2Enabled()` | Only active when Todo V2 is enabled |
| `isConcurrencySafe()` | `true` | Safe to run concurrently |
| `isReadOnly()` | `true` | Read-only operation (no side effects) |
| `renderToolUseMessage()` | `null` | No custom tool use message rendering |

## Task Listing Flow

```
1. INPUT RECEIVED
   - Model returns TaskList tool_use with {} (no parameters)

2. TASK LISTING
   - getTaskListId() resolves the current task list
   - listTasks(taskListId) reads all task files
   - Internal tasks filtered out (metadata?._internal === true)

3. DEPENDENCY FILTERING
   - Build set of completed task IDs
   - Filter out resolved dependencies from blockedBy arrays
   - Only unresolved (open) blockers are shown

4. RESULT MAPPING
   - Each task mapped to summary: { id, subject, status, owner, blockedBy }
```

## Implementation Details

### Task Listing

The tool delegates to `listTasks()` from `utils/tasks.ts`, which reads all task files from the task list directory and returns them as an array.

### Internal Task Filtering

Tasks with `metadata._internal` set to `true` are filtered out:
```typescript
const allTasks = (await listTasks(taskListId)).filter(
  t => !t.metadata?._internal,
)
```

This hides system-internal tasks from the model's view, keeping the task list clean and focused on user-facing work items.

### Resolved Dependency Filtering

Completed tasks are removed from `blockedBy` lists:
```typescript
const resolvedTaskIds = new Set(
  allTasks.filter(t => t.status === 'completed').map(t => t.id),
)

const tasks = allTasks.map(task => ({
  ...task,
  blockedBy: task.blockedBy.filter(id => !resolvedTaskIds.has(id)),
}))
```

This ensures the model only sees active blockers. If a dependency has been completed, it no longer appears in the blockedBy list, making it clear which tasks are actually blocked.

### Result Formatting

The `mapToolResultToToolResultBlockParam` method formats the result as human-readable text:

**Tasks found:**
```
#${task.id} [${task.status}] ${task.subject}${owner}${blocked}
```

Where:
- `owner` = ` (${task.owner})` if assigned, empty otherwise
- `blocked` = ` [blocked by #${id1}, #${id2}]` if blockedBy is non-empty

**No tasks:**
```
No tasks found
```

### Read-Only Classification

The tool implements `isReadOnly() => true`, marking it as a read-only operation with no side effects.

### Agent Swarms Integration

When `isAgentSwarmsEnabled()` returns true, the prompt includes teammate-specific workflow guidance:
- TaskList is used to find available work after completing a task
- Look for tasks with status `pending`, no owner, and empty `blockedBy`
- Prefer tasks in ID order (lowest first) for sequential context
- Claim tasks using TaskUpdate (set `owner`) or wait for leader assignment

## Error Handling

| Error Condition | Handling |
|----------------|----------|
| No tasks in list | Returns empty array, formatted as "No tasks found" |
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
| Agent Swarms (`isAgentSwarmsEnabled()`) | Enables teammate workflow guidance in prompt |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/tasks.ts` | `listTasks`, `getTaskListId`, `isTodoV2Enabled`, `TaskStatusSchema` |
| `utils/lazySchema.ts` | Lazy schema evaluation |
| `utils/agentSwarmsEnabled.ts` | Agent Swarms feature detection |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |

## Data Flow

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

## Integration with Task System

TaskListTool provides the overview layer of the Todo V2 task system:

1. **Task Discovery**: Primary way to discover available tasks, their status, and dependencies
2. **Progress Tracking**: Shows overall project progress through task statuses
3. **Dependency Visibility**: Displays unresolved blockers so the model can prioritize unblocking work
4. **Teammate Coordination**: In Agent Swarms mode, teammates use TaskList to find unassigned, unblocked tasks to claim
5. **ID Order Preference**: The prompt advises working on tasks in ID order (lowest first) since earlier tasks often set up context for later ones
6. **Post-Completion Navigation**: After completing a task, the model calls TaskList to find newly unblocked work or the next available task

The tool filters resolved dependencies to prevent clutter — completed tasks no longer appear as blockers, keeping the output focused on actionable information.
