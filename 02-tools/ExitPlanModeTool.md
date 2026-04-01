# ExitPlanModeTool

## Purpose

ExitPlanModeTool (ExitPlanModeV2Tool) presents a completed plan to the user for approval and transitions the session out of plan mode back to the previous mode. It handles plan file persistence, team-based approval workflows, and restores the full permission set that was stripped when entering plan mode. The tool supports both direct user approval and multi-agent team lead approval flows.

## Location

- `restored-src/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` — Main tool definition and call logic (494 lines)
- `restored-src/src/tools/ExitPlanModeTool/prompt.ts` — Tool prompt (30 lines)
- `restored-src/src/tools/ExitPlanModeTool/UI.tsx` — Terminal UI rendering (82 lines)
- `restored-src/src/tools/ExitPlanModeTool/constants.ts` — Tool name constants (3 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `ExitPlanModeV2Tool` | The complete tool definition built via `buildTool()` |
| `Output` | Output type: `{ plan, isAgent, filePath, hasTaskTool?, planWasEdited?, awaitingLeaderApproval?, requestId? }` |
| `AllowedPrompt` | Type for prompt-based permission requests: `{ tool: 'Bash', prompt: string }` |
| `_sdkInputSchema` | SDK-facing schema with injected `plan` and `planFilePath` fields |
| `EXIT_PLAN_MODE_TOOL_NAME` | Constant string `'ExitPlanMode'` |
| `EXIT_PLAN_MODE_V2_TOOL_NAME` | Constant string `'ExitPlanMode'` (same name, V2 implementation) |

## Input/Output Schemas

### Input Schema

```typescript
{
  allowedPrompts?: {
    tool: 'Bash',
    prompt: string  // Semantic description like "run tests", "install dependencies"
  }[]
}
```

The input accepts optional prompt-based permissions that the plan needs to implement. These describe categories of actions rather than specific commands, allowing the user to grant broad permission buckets for plan execution.

### SDK Input Schema (Extended)

```typescript
{
  allowedPrompts?: { tool: 'Bash', prompt: string }[]
  plan?: string          // Injected by normalizeToolInput from disk
  planFilePath?: string  // Injected by normalizeToolInput
}
```

The SDK sees additional fields that are injected by `normalizeToolInput` — the plan content is read from disk rather than passed as a parameter.

### Output Schema

```typescript
{
  plan: string | null,                // The plan that was presented
  isAgent: boolean,                   // Whether called from an agent context
  filePath?: string,                  // Path where the plan was saved
  hasTaskTool?: boolean,              // Whether Agent tool is available
  planWasEdited?: boolean,            // True if user edited the plan (CCR or Ctrl+G)
  awaitingLeaderApproval?: boolean,   // Teammate sent plan to team lead
  requestId?: string,                 // Unique ID for the plan approval request
}
```

## Mode Transition

### Exit Flow

```
1. MODEL CALLS ExitPlanMode
   - Optional: { allowedPrompts: [...] }

2. VALIDATION (validateInput)
   - For teammates: always passes (leader handles approval)
   - For non-teammates: verifies mode === 'plan'
   - Rejects with error code 1 if not in plan mode

3. PERMISSION CHECK (checkPermissions)
   - Teammates: 'allow' (bypass UI, mailbox handles approval)
   - Non-teammates: 'ask' (show confirmation dialog)

4. PLAN RETRIEVAL
   - Read plan from disk via getPlan(agentId)
   - If CCR web UI sent edited plan via input.plan, write to disk first
   - Persist file snapshot for remote contexts

5. TEAMMATE APPROVAL FLOW (if applicable)
   - If isTeammate() && isPlanModeRequired():
     a. Generate unique request ID
     b. Build plan_approval_request message
     c. Write to team-lead mailbox via writeToMailbox()
     d. Update task state to show "awaiting approval"
     e. Return with awaitingLeaderApproval: true

6. MODE RESTORATION (non-teammate or voluntary plan mode)
   - Retrieve prePlanMode from state
   - Check auto-mode gate status (circuit breaker defense)
   - Restore permissions (dangerous rules)
   - Update state: mode = prePlanMode, prePlanMode = undefined
   - Set hasExitedPlanMode flag
   - Set needsPlanModeExitAttachment flag

7. RESULT RETURNED
   - Plan content, file path, and metadata
   - mapToolResultToToolResultBlockParam() formats response
```

## Plan File Handling

### Plan Source Priority

1. **CCR Web UI Edit**: If the user edited the plan through the CCR web interface, the edited content arrives via `permissionResult.updatedInput` and is written to disk before processing
2. **Disk Fallback**: If no edited plan is provided, the plan is read from disk via `getPlan(agentId)`

### Disk Synchronization

When an edited plan is provided:
```typescript
if (inputPlan !== undefined && filePath) {
  await writeFile(filePath, inputPlan, 'utf-8')
  void persistFileSnapshotIfRemote()
}
```

This ensures that `VerifyPlanExecution` and `Read` tools see the user-edited version. The snapshot is re-saved because the earlier snapshot (captured in `normalizeToolInput`) contains the old plan.

### Plan File Validation

For teammates where `isPlanModeRequired()` is true, the tool validates that a plan file exists before proceeding:
```typescript
if (!plan) {
  throw new Error(`No plan file found at ${filePath}...`)
}
```

## Team-Based Approval Workflow

### Teammate Detection

The tool uses `isTeammate()` to determine if the current session is running as a team member agent. Teammates have a different approval flow than standalone sessions.

### Plan Approval Request

When a teammate with required plan mode exits planning:

```typescript
const approvalRequest = {
  type: 'plan_approval_request',
  from: agentName,
  timestamp: new Date().toISOString(),
  planFilePath: filePath,
  planContent: plan,
  requestId,
}

await writeToMailbox('team-lead', {
  from: agentName,
  text: jsonStringify(approvalRequest),
  timestamp: new Date().toISOString(),
}, teamName)
```

The request is sent to the team lead's mailbox. The teammate then waits for a response before proceeding with implementation.

### Task State Update

For in-process teammates, the task state is updated to show the approval status:
```typescript
setAwaitingPlanApproval(agentTaskId, context.setAppState, true)
```

This allows the team lead to see which tasks are awaiting plan approval.

### Approval Response Handling

When the team lead approves or rejects, the response arrives in the teammate's mailbox. The teammate processes the response and either proceeds with implementation or refines the plan.

## Mode Restoration

### Pre-Plan Mode Recovery

The tool restores the mode that was active before entering plan mode:

```typescript
let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
```

### Auto-Mode Gate Defense

A circuit breaker prevents bypassing auto-mode restrictions through plan mode:

```typescript
if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
  restoreMode = 'default'  // Fall back to default if auto mode is unavailable
}
```

Without this defense, ExitPlanMode could bypass the circuit breaker by calling `setAutoModeActive(true)` directly.

### Permission Restoration

When restoring to a non-auto mode, dangerous permissions that were stripped during plan mode entry are restored:

```typescript
if (!restoringToAuto && prev.toolPermissionContext.strippedDangerousRules) {
  baseContext = restoreDangerousPermissions(baseContext)
}
```

When restoring to auto mode, dangerous permissions remain stripped:
```typescript
if (restoringToAuto) {
  baseContext = stripDangerousPermissionsForAutoMode(baseContext)
}
```

### State Flags

| Flag | Set When | Purpose |
|------|----------|---------|
| `hasExitedPlanMode` | Any successful exit | Tracks that plan mode was exited this session |
| `needsPlanModeExitAttachment` | Any successful exit | Triggers plan mode exit attachment in response |
| `needsAutoModeExitAttachment` | Auto mode was used during plan but restoring to default | Signals auto-mode transition cleanup |

## Result Block Formatting

### Teammate Awaiting Approval

```
Your plan has been submitted to the team lead for approval.

Plan file: ${filePath}

**What happens next:**
1. Wait for the team lead to review your plan
2. You will receive a message in your inbox with approval/rejection
3. If approved, you can proceed with implementation
4. If rejected, refine your plan based on the feedback

**Important:** Do NOT proceed until you receive approval. Check your inbox for response.

Request ID: ${requestId}
```

### Agent Context

```
User has approved the plan. There is nothing else needed from you now. Please respond with "ok"
```

### Empty Plan

```
User has approved exiting plan mode. You can now proceed.
```

### Standard Plan Approval

```
User has approved your plan. You can now start coding. Start with updating your todo list if applicable

Your plan has been saved to: ${filePath}
You can refer back to it if needed during implementation.

## Approved Plan [edited by user]:
${plan}
```

### Team Parallelization Hint

When the Agent tool is available, a hint is appended:
```
If this plan can be broken down into multiple independent tasks, consider using the TeamCreate tool to create a team and parallelize the work.
```

## UI Rendering

### Tool Use Message

Returns `null` — no intermediate "using" message is displayed.

### Tool Result Message

Three variants based on plan state:

**Empty plan:**
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> Exited plan mode</Text>
  </Box>
</Box>
```

**Awaiting leader approval:**
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> Plan submitted for team lead approval</Text>
  </Box>
  <MessageResponse>
    <Box flexDirection="column">
      {filePath && <Text dimColor>Plan file: {displayPath}</Text>}
      <Text dimColor>Waiting for team lead to review and approve...</Text>
    </Box>
  </MessageResponse>
</Box>
```

**User approved plan:**
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> User approved Claude&apos;s plan</Text>
  </Box>
  <MessageResponse>
    <Box flexDirection="column">
      {filePath && <Text dimColor>Plan saved to: {displayPath} · /plan to edit</Text>}
      <Markdown>{plan}</Markdown>
    </Box>
  </MessageResponse>
</Box>
```

### Tool Use Rejected Message

```tsx
<Box flexDirection="column">
  <RejectedPlanMessage plan={planContent} />
</Box>
```

Renders the `RejectedPlanMessage` component with the plan content, showing the user's rejection feedback.

## Validation and Error Handling

### Input Validation

| Condition | Result | Error Code |
|-----------|--------|------------|
| Not in plan mode | Reject with message | 1 |
| Teammate context | Always pass (leader handles approval) | — |
| Cannot verify worktree state | Reject (fail-closed) | 3 |

### Permission Check

| Context | Behavior | Rationale |
|---------|----------|-----------|
| Teammate | `allow` — bypass UI | Mailbox handles approval asynchronously |
| Non-teammate | `ask` — show confirmation | User must explicitly approve exiting plan mode |

### Mode Gating

Same gating as EnterPlanMode — disabled when `--channels` feature flag is active with configured channels. This prevents the model from entering a mode it cannot exit on messaging platforms.

## Prompt Guidance

### When to Use

- After writing a plan to the plan file and ready for user approval
- Only for implementation planning tasks (not research/exploration)

### Before Using

- Plan must be complete and unambiguous
- Use AskUserQuestion first if there are unresolved approach questions
- Do NOT use AskUserQuestion to ask "Is this plan okay?" — that's what ExitPlanMode does

### Examples

**Good:** "Help me implement yank mode for vim" — use ExitPlanMode after planning implementation steps

**Bad:** "Search for and understand the implementation of vim mode" — do not use, this is research not planning

## Integration Points

### `bootstrap/state.js`

- `getAllowedChannels()` — Channel list for gating
- `hasExitedPlanModeInSession()` — Session-level exit tracking
- `setHasExitedPlanMode(bool)` — Mark plan mode as exited
- `setNeedsPlanModeExitAttachment(bool)` — Trigger exit attachment
- `setNeedsAutoModeExitAttachment(bool)` — Signal auto-mode cleanup

### `utils/plans.js`

- `getPlan(agentId)` — Read plan content from disk
- `getPlanFilePath(agentId)` — Resolve plan file path
- `persistFileSnapshotIfRemote()` — Sync plan to remote storage

### `utils/teammate.js`

- `isTeammate()` — Check if running as team member
- `isPlanModeRequired()` — Check if plan mode is mandatory for this agent
- `getAgentName()` / `getTeamName()` — Agent/team identification

### `utils/teammateMailbox.js`

- `writeToMailbox(recipient, message, teamName)` — Send approval request to team lead

### `utils/inProcessTeammateHelpers.js`

- `findInProcessTeammateTaskId(agentName, state)` — Find task ID for state updates
- `setAwaitingPlanApproval(taskId, setState, bool)` — Update task approval status

### `utils/agentId.js`

- `formatAgentId(name, team)` — Format agent identifier
- `generateRequestId(type, agentId)` — Generate unique request ID

### `utils/permissions/autoModeState.js` (conditional)

- `isAutoModeActive()` — Check current auto-mode state
- `setAutoModeActive(bool)` — Set auto-mode state
- `isAutoModeGateEnabled()` — Check if auto-mode gate is active
- `getAutoModeUnavailableReason()` — Get reason auto-mode is unavailable
- `getAutoModeUnavailableNotification(reason)` — Get user notification text
- `stripDangerousPermissionsForAutoMode(context)` — Strip permissions for auto mode
- `restoreDangerousPermissions(context)` — Restore previously stripped permissions

### `utils/permissions/permissionSetup.js` (conditional)

- Module loaded dynamically when `TRANSCRIPT_CLASSIFIER` feature is enabled

## Data Flow

```
Model has completed planning
    |
    v
ExitPlanMode tool call { allowedPrompts? }
    |
    v
┌─────────────────────────────────────────┐
│ VALIDATION                              │
│ - Teammate: pass through                │
│ - Non-teammate: verify mode === 'plan'  │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ PERMISSION CHECK                        │
│ - Teammate: allow (bypass UI)           │
│ - Non-teammate: ask (show dialog)       │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ PLAN RETRIEVAL                          │
│ - Read from disk                        │
│ - Write edited plan if provided         │
│ - Persist snapshot for remote           │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ TEAMMATE FLOW?                          │
│                                         │
│ Yes:                                    │
│   - Build approval request              │
│   - Send to team lead mailbox           │
│   - Update task state                   │
│   - Return with awaitingLeaderApproval  │
│                                         │
│ No:                                     │
│   - Proceed to mode restoration         │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ MODE RESTORATION                        │
│                                         │
│ - Retrieve prePlanMode                  │
│ - Check auto-mode gate (circuit breaker)│
│ - Restore/strip permissions             │
│ - Update state: mode = prePlanMode      │
│ - Set exit flags                        │
└─────────────────────────────────────────┘
    |
    v
Result block with approved plan content
    |
    v
Model begins implementation phase
```
