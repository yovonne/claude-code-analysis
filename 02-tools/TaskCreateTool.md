# TaskCreateTool

## Purpose

TaskCreateTool creates new structured tasks in the task list (Todo V2 system). It allows the model to break down complex work into trackable, ordered tasks with titles, descriptions, optional active forms for spinners, and arbitrary metadata. Tasks are created with `pending` status and no owner by default. When Agent Swarms are enabled, tasks can later be assigned to teammates.

## Location

- `restored-src/src/tools/TaskCreateTool/TaskCreateTool.ts` — Main tool definition (~139 lines)
- `restored-src/src/tools/TaskCreateTool/constants.ts` — Tool name constant (2 lines)
- `restored-src/src/tools/TaskCreateTool/prompt.ts` — Tool prompt and description (57 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `TaskCreateTool` | The complete tool definition built via `buildTool()` |
| `TASK_CREATE_TOOL_NAME` | Constant: `'TaskCreate'` |
| `DESCRIPTION` | Constant: `'Create a new task in the task list'` |
| `getPrompt()` | Generates the usage prompt, adapting for Agent Swarms context |
| `Output` | Zod-inferred output type: `{ task: { id: string, subject: string } }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  subject: string,        // Required: brief title for the task (imperative form)
  description: string,    // Required: what needs to be done
  activeForm?: string,    // Optional: present continuous form for spinner (e.g., "Running tests")
  metadata?: Record<string, unknown>  // Optional: arbitrary key-value metadata
}
```

### Output Schema

```typescript
{
  task: {
    id: string,           // Generated task ID
    subject: string,      // The task title
  }
}
```

## Tool Definition Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'TaskCreate'` | Internal tool identifier |
| `searchHint` | `'create a task in the task list'` | Tool search hint |
| `maxResultSizeChars` | `100_000` | Maximum result size |
| `userFacingName()` | `'TaskCreate'` | Display name in UI |
| `shouldDefer` | `true` | Tool execution can be deferred |
| `isEnabled()` | `isTodoV2Enabled()` | Only active when Todo V2 feature is enabled |
| `isConcurrencySafe()` | `true` | Safe to run concurrently |
| `toAutoClassifierInput()` | `input.subject` | Uses subject for auto-mode classification |
| `renderToolUseMessage()` | `null` | No custom tool use message rendering |

## Task Creation Flow

```
1. INPUT RECEIVED
   - Model returns TaskCreate tool_use with { subject, description, activeForm?, metadata? }

2. TASK CREATION
   - getTaskListId() resolves the current task list
   - createTask() creates the task file with:
     - subject, description, activeForm
     - status: 'pending'
     - owner: undefined
     - blocks: []
     - blockedBy: []
     - metadata (if provided)
   - Returns the generated taskId

3. POST-CREATION HOOKS
   - executeTaskCreatedHooks() runs asynchronously
   - Generator yields results; blocking errors are collected
   - If any blocking error occurs:
     - Task is deleted via deleteTask()
     - Error is thrown

4. UI UPDATE
   - Auto-expands the task list view in the app state
   - Sets expandedView to 'tasks' if not already expanded

5. RESULT RETURNED
   - { task: { id: taskId, subject } }
```

## Implementation Details

### Task Creation

The tool delegates to `createTask()` from `utils/tasks.ts`, which writes a task file to the task list directory. All tasks start with:
- Status: `pending`
- Owner: `undefined` (unassigned)
- No dependencies (`blocks: []`, `blockedBy: []`)

### Post-Creation Hooks

After task creation, `executeTaskCreatedHooks()` runs a generator that performs side effects (e.g., notifications, logging). The generator yields results that may contain `blockingError` flags. If any blocking error is encountered:
1. All blocking error messages are collected
2. The newly created task is deleted (rollback)
3. An `Error` is thrown with the combined messages

This ensures task creation is atomic — if hooks fail, no orphaned task remains.

### UI Auto-Expand

When a task is created, the tool automatically expands the task list view by updating app state:
```typescript
context.setAppState(prev => {
  if (prev.expandedView === 'tasks') return prev
  return { ...prev, expandedView: 'tasks' as const }
})
```

This provides immediate visual feedback to the user.

### Result Formatting

The `mapToolResultToToolResultBlockParam` method formats the result as a human-readable string:
```
Task #${task.id} created successfully: ${task.subject}
```

### Agent Swarms Integration

When `isAgentSwarmsEnabled()` returns true, the prompt includes additional guidance:
- Descriptions should include enough detail for another agent to understand
- New tasks are created without an owner — use `TaskUpdate` to assign them
- Tips about assigning tasks to teammates

## Error Handling

| Error Condition | Handling |
|----------------|----------|
| Post-creation hook blocking error | Task is deleted, error thrown with combined messages |
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
| Agent Swarms (`isAgentSwarmsEnabled()`) | Enables teammate assignment features in prompt |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/tasks.ts` | `createTask`, `deleteTask`, `getTaskListId`, `isTodoV2Enabled` |
| `utils/hooks.ts` | `executeTaskCreatedHooks`, `getTaskCreatedHookMessage` |
| `utils/teammate.ts` | `getAgentName`, `getTeamName` |
| `utils/lazySchema.ts` | Lazy schema evaluation |
| `utils/agentSwarmsEnabled.ts` | Agent Swarms feature detection |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `bun:bundle` | Feature flag API (via hooks) |

## Data Flow

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
│ - On error: deleteTask() + throw                                 │
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

## Integration with Task System

TaskCreateTool is the entry point for the Todo V2 task lifecycle:

1. **Task Creation**: Creates task files in the task list directory with `pending` status
2. **Task Lifecycle**: Created tasks flow through: `pending` → `in_progress` → `completed` (via TaskUpdateTool)
3. **Dependencies**: Tasks can have `blocks` and `blockedBy` relationships set via TaskUpdateTool after creation
4. **Assignment**: Tasks can be assigned to owners (agents/teammates) via TaskUpdateTool
5. **Visibility**: Created tasks appear in TaskListTool output and can be retrieved individually via TaskGetTool

Tasks created by this tool are persisted as files on disk in the task list directory structure, enabling session persistence and resume capability.
