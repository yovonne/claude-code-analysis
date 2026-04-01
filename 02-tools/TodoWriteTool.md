# TodoWriteTool

## Purpose

TodoWriteTool manages a structured task checklist for the current coding session. It allows Claude to create, update, and track progress on multi-step tasks, demonstrating thoroughness and keeping the user informed of overall progress. The tool stores todos in AppState keyed by session or agent ID, auto-clears completed lists, and includes a verification nudge mechanism that reminds the model to spawn a verification agent when closing out 3+ tasks without one.

## Location

- `restored-src/src/tools/TodoWriteTool/TodoWriteTool.ts` — Main tool definition and state management (116 lines)
- `restored-src/src/tools/TodoWriteTool/prompt.ts` — Comprehensive usage guidelines with examples (185 lines)
- `restored-src/src/tools/TodoWriteTool/constants.ts` — Tool name constant (2 lines)

## Key Exports

### From TodoWriteTool.ts

| Export | Description |
|--------|-------------|
| `TodoWriteTool` | The complete tool definition built via `buildTool()` |
| `Output` | Zod-inferred output type: `{ oldTodos, newTodos, verificationNudgeNeeded? }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  todos: TodoItem[],  // Required: The complete updated todo list
}

// TodoItem structure (from TodoListSchema):
{
  content: string,       // Task description (imperative form)
  activeForm: string,    // Present continuous form (e.g., "Running tests")
  status: 'pending' | 'in_progress' | 'completed',
}
```

### Output Schema

```typescript
{
  oldTodos: TodoItem[],              // The todo list before the update
  newTodos: TodoItem[],              // The todo list after the update
  verificationNudgeNeeded?: boolean, // True when verification reminder should be shown
}
```

## Task Management

### Execution Flow

```
1. INPUT RECEIVED
   - Model returns TodoWrite tool_use with { todos: [...] }

2. PERMISSION CHECK
   - Auto-allow (no permission check needed for todo operations)

3. STATE UPDATE
   a. Get current app state
   b. Determine storage key: agentId ?? sessionId
   c. Store old todos for comparison
   d. Check if all todos are completed
   e. If all done → clear list (newTodos = [])
   f. Otherwise → store provided todos
   g. Update AppState.todos[todoKey]

4. VERIFICATION NUDGE CHECK
   Conditions (all must be true):
   - VERIFICATION_AGENT feature flag enabled
   - tengu_hive_evidence feature flag enabled
   - No agentId (main-thread agent only)
   - All todos are completed
   - 3+ items in the list
   - None contain "verif" (case-insensitive) in content
   → If all true: set verificationNudgeNeeded = true

5. RESULT RETURNED
   - oldTodos, newTodos, verificationNudgeNeeded
```

### Task States

| State | Meaning |
|-------|---------|
| `pending` | Task not yet started |
| `in_progress` | Currently working on (limit: ONE at a time) |
| `completed` | Task finished successfully |

### Task Description Requirements

Each task must have two forms:

| Field | Form | Example |
|-------|------|---------|
| `content` | Imperative | "Run tests", "Build the project" |
| `activeForm` | Present continuous | "Running tests", "Building the project" |

### Auto-Clear on Completion

When all todos reach `completed` status, the list is automatically cleared (set to `[]`). This prevents stale completed task lists from persisting in the session state.

## Verification Nudge

When the model completes 3+ tasks without including a verification step, the tool result includes a reminder:

```
NOTE: You just closed out 3+ tasks and none of them was a verification step.
Before writing your final summary, spawn the verification agent
(subagent_type="<VERIFICATION_AGENT_TYPE>"). You cannot self-assign PARTIAL
by listing caveats in your summary — only the verifier issues a verdict.
```

This fires at the exact moment when the last task closes and the loop would exit, catching the pattern where the model skips verification.

### Nudge Conditions

All must be true:

| Condition | Check |
|-----------|-------|
| Feature flag | `VERIFICATION_AGENT` enabled |
| Feature flag | `tengu_hive_evidence` enabled |
| Main agent only | `context.agentId` is falsy |
| All complete | Every todo has `status === 'completed'` |
| List size | `todos.length >= 3` |
| No verification | No todo content matches `/verif/i` |

## Storage

Todos are stored in `AppState.todos` as a map:

```typescript
AppState.todos = {
  [sessionId]: [...],      // Main session todos
  [agentId]: [...],        // Agent-specific todos (if spawned)
}
```

The storage key is `context.agentId ?? getSessionId()`, ensuring:
- Main session uses the session ID
- Spawned agents have their own independent todo lists

## When to Use

The tool prompt provides detailed guidance on when to use the todo list:

### Use When

1. Complex multi-step tasks (3+ distinct steps)
2. Non-trivial tasks requiring careful planning
3. User explicitly requests todo list
4. User provides multiple tasks (numbered or comma-separated)
5. After receiving new instructions
6. When starting work on a task (mark as `in_progress` BEFORE beginning)
7. After completing a task (mark as `completed`, add follow-ups)

### Skip When

1. Single, straightforward task
2. Trivial task with no organizational benefit
3. Task completable in less than 3 trivial steps
4. Purely conversational or informational request

## Task Management Rules

| Rule | Description |
|------|-------------|
| One at a time | Exactly ONE task must be `in_progress` at any time |
| Immediate completion | Mark tasks complete IMMEDIATELY after finishing |
| No batching | Don't batch completions |
| Remove irrelevant | Delete tasks no longer relevant from the list entirely |
| Full accomplishment | ONLY mark completed when FULLY accomplished |
| Blockers | Keep as `in_progress` if blocked; create new task for resolution |
| Specific items | Create actionable, specific items with clear names |

## Permission Handling

| Operation | Permission Decision |
|-----------|-------------------|
| All operations | Auto-allow (no permission check required) |

The tool is considered safe because it only modifies in-memory state and has no side effects on the filesystem or system.

## Tool Availability

| Condition | Effect |
|-----------|--------|
| `isTodoV2Enabled()` returns true | Tool is **disabled** (V2 replaces it) |
| `isTodoV2Enabled()` returns false | Tool is **enabled** |

This tool represents the V1 todo system. When V2 is enabled, this tool is disabled and the new system takes over.

## Result Message

The tool result always includes a standard message:

```
Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable
```

When the verification nudge is triggered, it appends the verification reminder to this message.

## Configuration

### Feature Flags

| Flag | Purpose |
|------|---------|
| `VERIFICATION_AGENT` | Enables verification agent feature |
| `tengu_hive_evidence` | Enables verification nudge |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `USER_TYPE` | `'ant'` for internal users (enables extra features) |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `bootstrap/state.js` | Session ID access |
| `services/analytics/growthbook.js` | Feature flag access |
| `utils/tasks.js` | Todo V2 enablement check |
| `utils/todo/types.js` | TodoListSchema definition |
| `tools/AgentTool/constants.js` | Verification agent type constant |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `bun:bundle` | Feature flag API |

## Data Flow

```
Model Request
    |
    v
TodoWriteTool Input { todos: [{ content, activeForm, status }] }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  - Auto-allow (no permission check needed)                      │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ STATE UPDATE                                                    │
│                                                                 │
│  1. Get AppState                                                │
│  2. Determine key: agentId ?? sessionId                         │
│  3. Save oldTodos                                               │
│  4. Check if all completed                                      │
│  5. If all done → clear list                                    │
│  6. Otherwise → store new todos                                 │
│  7. Update AppState.todos[key]                                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VERIFICATION NUDGE CHECK                                        │
│                                                                 │
│  All conditions:                                                │
│  - VERIFICATION_AGENT flag on                                   │
│  - tengu_hive_evidence flag on                                  │
│  - Main agent (no agentId)                                      │
│  - All todos completed                                          │
│  - 3+ items                                                     │
│  - No "verif" in any content                                    │
│                                                                 │
│  → If all true: append verification reminder to result          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { oldTodos, newTodos, verificationNudgeNeeded? }
    |
    v
Model Response (tool_result with success message + optional nudge)
```
