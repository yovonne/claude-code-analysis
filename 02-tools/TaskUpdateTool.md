# TaskUpdateTool

## Purpose

TaskUpdateTool modifies existing tasks in the task list (Todo V2 system). It supports updating task status (including the special `deleted` action), subject, description, active form, owner, metadata, and dependency relationships (blocks/blockedBy). It also executes post-completion hooks when tasks are marked as completed, handles teammate assignment notifications via mailbox, and provides verification nudges when tasks are closed without verification steps.

## Location

- `restored-src/src/tools/TaskUpdateTool/TaskUpdateTool.ts` — Main tool definition (~407 lines)
- `restored-src/src/tools/TaskUpdateTool/constants.ts` — Tool name constant (2 lines)
- `restored-src/src/tools/TaskUpdateTool/prompt.ts` — Tool prompt and description (78 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `TaskUpdateTool` | The complete tool definition built via `buildTool()` |
| `TASK_UPDATE_TOOL_NAME` | Constant: `'TaskUpdate'` |
| `DESCRIPTION` | Constant: `'Update a task in the task list'` |
| `PROMPT` | Usage instructions with status workflow, field descriptions, and examples |
| `Output` | Zod-inferred output type: `{ success, taskId, updatedFields, error?, statusChange?, verificationNudgeNeeded? }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  taskId: string,           // Required: the ID of the task to update
  subject?: string,         // Optional: new task title
  description?: string,     // Optional: new task description
  activeForm?: string,      // Optional: present continuous form for spinner
  status?: TaskStatus | 'deleted',  // Optional: new status or 'deleted' action
  addBlocks?: string[],     // Optional: task IDs that this task blocks
  addBlockedBy?: string[],  // Optional: task IDs that block this task
  owner?: string,           // Optional: new task owner (agent name)
  metadata?: Record<string, unknown>,  // Optional: metadata to merge (null to delete key)
}
```

### Output Schema

```typescript
{
  success: boolean,                    // Whether the update succeeded
  taskId: string,                      // The task that was updated
  updatedFields: string[],             // List of fields that were actually changed
  error?: string,                      // Error message if success is false
  statusChange?: {                     // Only present when status was changed
    from: string,
    to: string,
  },
  verificationNudgeNeeded?: boolean,   // True when 3+ tasks closed without verification
}
```

## Tool Definition Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'TaskUpdate'` | Internal tool identifier |
| `searchHint` | `'update a task'` | Tool search hint |
| `maxResultSizeChars` | `100_000` | Maximum result size |
| `userFacingName()` | `'TaskUpdate'` | Display name in UI |
| `shouldDefer` | `true` | Tool execution can be deferred |
| `isEnabled()` | `isTodoV2Enabled()` | Only active when Todo V2 is enabled |
| `isConcurrencySafe()` | `true` | Safe to run concurrently |
| `toAutoClassifierInput()` | `input.taskId + input.status + input.subject` | Composite for auto-mode classification |
| `renderToolUseMessage()` | `null` | No custom tool use message rendering |

## Task Update Flow

```
1. INPUT RECEIVED
   - Model returns TaskUpdate tool_use with { taskId, ...updates }

2. UI AUTO-EXPAND
   - Auto-expands task list view in app state

3. TASK EXISTS CHECK
   - getTask(taskListId, taskId) retrieves current task
   - If not found: returns { success: false, error: 'Task not found' }

4. FIELD UPDATES (only if different from current value)
   - subject, description, activeForm, owner
   - metadata (merged, null values delete keys)
   - Auto-set owner when teammate marks in_progress (Agent Swarms)

5. STATUS HANDLING
   - 'deleted': deleteTask() and return early
   - 'completed': executeTaskCompletedHooks() before applying
   - Other: apply if different from current status

6. APPLY UPDATES
   - updateTask(taskListId, taskId, updates) if any changes

7. OWNER NOTIFICATION (Agent Swarms)
   - writeToMailbox() notifies new owner of assignment

8. DEPENDENCY UPDATES
   - addBlocks: blockTask() for each new block
   - addBlockedBy: blockTask() for each new blocker (reverse direction)

9. VERIFICATION NUDGE
   - Check if 3+ tasks completed without verification step
   - Set verificationNudgeNeeded flag if so

10. RESULT RETURNED
    - { success, taskId, updatedFields, statusChange?, verificationNudgeNeeded? }
```

## Implementation Details

### Field Update Logic

Each field is only updated if:
1. The value is provided (not `undefined`)
2. The value differs from the current task value

This prevents unnecessary writes and keeps `updatedFields` accurate:
```typescript
if (subject !== undefined && subject !== existingTask.subject) {
  updates.subject = subject
  updatedFields.push('subject')
}
```

### Metadata Merging

Metadata is merged with existing metadata, with `null` values deleting keys:
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

### Status Workflow

Status progresses through: `pending` → `in_progress` → `completed`

The special `deleted` status permanently removes the task from the task list.

#### Deletion

When `status === 'deleted'`:
```typescript
const deleted = await deleteTask(taskListId, taskId)
return { success: deleted, taskId, updatedFields: deleted ? ['deleted'] : [] }
```

Returns early — no other fields are updated.

#### Completion Hooks

When `status === 'completed'`, the tool runs `executeTaskCompletedHooks()` before applying the status change:
- Generator yields results that may contain `blockingError` flags
- If any blocking error occurs: returns `{ success: false, error: combined messages }` without applying the status change
- This ensures completion hooks can veto the status change if needed

### Auto-Owner Assignment (Agent Swarms)

When Agent Swarms is enabled and a task is marked `in_progress` without an explicit owner:
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

This ensures the task list can match todo items to teammates for showing activity status.

### Owner Notification (Agent Swarms)

When ownership changes and Agent Swarms is enabled:
```typescript
await writeToMailbox(
  updates.owner,
  {
    from: senderName,
    text: assignmentMessage,  // JSON with type, taskId, subject, description, assignedBy, timestamp
    timestamp: new Date().toISOString(),
    color: senderColor,
  },
  taskListId,
)
```

This sends a task assignment message to the new owner's mailbox for asynchronous notification.

### Dependency Management

**addBlocks**: Adds tasks that cannot start until this task completes:
```typescript
const newBlocks = addBlocks.filter(id => !existingTask.blocks.includes(id))
for (const blockId of newBlocks) {
  await blockTask(taskListId, taskId, blockId)
}
```

**addBlockedBy**: Adds tasks that must complete before this task can start. The relationship is set up in reverse — the blocker's `blocks` array is updated:
```typescript
for (const blockerId of newBlockedBy) {
  await blockTask(taskListId, blockerId, taskId)
}
```

### Verification Nudge

When the main-thread agent completes all tasks in a list of 3+ tasks without any verification step:
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

This fires at the "loop exit moment" — when the last task is closed and the task loop would naturally end. The reminder is appended to the tool result message.

### Result Formatting

The `mapToolResultToToolResultBlockParam` method formats results:

**Success:**
```
Updated task #${taskId} ${updatedFields.join(', ')}
```

Plus optional additions:
- **Task completion (Agent Swarms)**: `\n\nTask completed. Call TaskList now to find your next available task or see if your work unblocked others.`
- **Verification nudge**: Reminder to spawn the verification agent

**Failure:**
```
${error || `Task #${taskId} not found`}
```

Failures are returned as non-error tool results (not triggering sibling tool cancellation).

## Error Handling

| Error Condition | Handling |
|----------------|----------|
| Task not found | Returns `{ success: false, error: 'Task not found' }` (non-error) |
| Completion hook blocking error | Returns `{ success: false, error: combined messages }` (non-error) |
| Delete failure | Returns `{ success: false, error: 'Failed to delete task' }` |
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
| Agent Swarms (`isAgentSwarmsEnabled()`) | Enables auto-owner assignment and mailbox notifications |
| VERIFICATION_AGENT | Enables verification nudge detection |
| `tengu_hive_evidence` | Additional gating for verification nudge |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/tasks.ts` | `blockTask`, `deleteTask`, `getTask`, `getTaskListId`, `isTodoV2Enabled`, `listTasks`, `updateTask`, `TaskStatusSchema` |
| `utils/hooks.ts` | `executeTaskCompletedHooks`, `getTaskCompletedHookMessage` |
| `utils/teammate.ts` | `getAgentId`, `getAgentName`, `getTeammateColor`, `getTeamName` |
| `utils/teammateMailbox.ts` | `writeToMailbox` for owner notifications |
| `utils/agentSwarmsEnabled.ts` | Agent Swarms feature detection |
| `utils/lazySchema.ts` | Lazy schema evaluation |
| `services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` |
| `tools/AgentTool/constants.ts` | `VERIFICATION_AGENT_TYPE` constant |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `bun:bundle` | Feature flag API (`feature()`) |

## Data Flow

```
Model Request
    |
    v
TaskUpdateTool Input { taskId, subject?, description?, status?, ... }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PRE-CHECKS                                                       │
│ - Auto-expand task list view                                     │
│ - getTask() → verify task exists                                 │
│ - If not found: return { success: false }                        │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ FIELD UPDATES (diff-based)                                       │
│ - subject, description, activeForm, owner                        │
│ - metadata (merge, null deletes)                                 │
│ - Auto-set owner on in_progress (Agent Swarms)                   │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ STATUS HANDLING                                                  │
│                                                                  │
│  'deleted' → deleteTask() + return early                         │
│  'completed' → executeTaskCompletedHooks()                       │
│              → On hook error: return { success: false }          │
│  Other → apply if different                                      │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERSIST + NOTIFY                                                 │
│ - updateTask() if changes exist                                  │
│ - writeToMailbox() for new owner (Agent Swarms)                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ DEPENDENCIES                                                     │
│ - addBlocks: blockTask() for each new block                      │
│ - addBlockedBy: blockTask() for each new blocker                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VERIFICATION CHECK                                               │
│ - All tasks completed + 3+ tasks + no verification step          │
│ - Set verificationNudgeNeeded if conditions met                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { success, taskId, updatedFields, statusChange?, verificationNudgeNeeded? }
    |
    v
Model Response: "Updated task #ID fields..." [+ reminders]
```

## Integration with Task System

TaskUpdateTool is the primary state-transition mechanism for the Todo V2 task lifecycle:

1. **Status Transitions**: Drives tasks through `pending` → `in_progress` → `completed` workflow
2. **Task Deletion**: Special `deleted` status permanently removes tasks from the task list
3. **Dependency Graph**: Manages `blocks` and `blockedBy` relationships via `blockTask()`, which updates both sides of the relationship
4. **Teammate Coordination**: In Agent Swarms mode, automatically assigns ownership and notifies assignees via mailbox
5. **Hook Integration**: Executes `TaskCreated` and `TaskCompleted` hooks for extensibility (notifications, logging, validation)
6. **Quality Assurance**: Verification nudge reminds the model to spawn a verification agent when closing multi-task work without verification
7. **Staleness Prevention**: The prompt advises reading a task's latest state via TaskGet before updating it
8. **Idempotent Updates**: Only writes fields that have actually changed, preventing unnecessary disk I/O and keeping audit trails clean

The tool coordinates across multiple subsystems: task file persistence, teammate mailbox messaging, hook execution, and dependency management — making it the most complex tool in the Task tool family.
