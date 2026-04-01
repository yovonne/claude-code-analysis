# EnterPlanModeTool

## Purpose

EnterPlanModeTool requests user permission to transition the session from the default/code mode into plan mode. In plan mode, the model focuses on exploring the codebase, understanding existing patterns, and designing an implementation approach before writing any code. The tool enforces a read-only exploration phase where file edits are restricted to the plan file only.

## Location

- `restored-src/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` — Main tool definition and call logic (127 lines)
- `restored-src/src/tools/EnterPlanModeTool/prompt.ts` — Tool prompt with usage guidelines (171 lines)
- `restored-src/src/tools/EnterPlanModeTool/UI.tsx` — Terminal UI rendering (33 lines)
- `restored-src/src/tools/EnterPlanModeTool/constants.ts` — Tool name constant (2 lines)

## Key Exports

| Export | Description |
|--------|-------------|
| `EnterPlanModeTool` | The complete tool definition built via `buildTool()` |
| `Output` | Output type: `{ message: string }` |
| `ENTER_PLAN_MODE_TOOL_NAME` | Constant string `'EnterPlanMode'` |

## Input/Output Schemas

### Input Schema

```typescript
{
  // No parameters needed — the tool is a simple mode transition request
}
```

The input schema is an empty strict object. No configuration is required because the tool's sole purpose is to request a mode transition.

### Output Schema

```typescript
{
  message: string,  // Confirmation that plan mode was entered
}
```

## Plan Mode Activation

### Activation Flow

```
1. MODEL CALLS EnterPlanMode
   - Tool is invoked with no parameters

2. CONTEXT VALIDATION
   - Throws if called from an agent context (agentId present)
   - Plan mode cannot be entered from sub-agents

3. MODE TRANSITION (handlePlanModeTransition)
   - handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')
   - Validates the transition is legal from the current mode

4. STATE UPDATE
   - context.setAppState() wraps prepareContextForPlanMode()
   - applyPermissionUpdate() sets mode to 'plan' with destination 'session'
   - prepareContextForPlanMode() runs classifier activation side effects
     when the user's defaultMode is 'auto'

5. RESULT RETURNED
   - message: "Entered plan mode. You should now focus on exploring..."
   - mapToolResultToToolResultBlockParam() appends workflow instructions
```

### State Changes

The tool modifies `toolPermissionContext` in app state:

| Field | Before | After |
|-------|--------|-------|
| `mode` | `'default'` / `'auto'` / etc. | `'plan'` |
| `prePlanMode` | undefined | (preserved, set by prepareContextForPlanMode) |
| Permission rules | Full access | Stripped to read-only (except plan file) |

### Classifier Side Effects

When `defaultMode` is `'auto'`, `prepareContextForPlanMode()` triggers the auto-mode classifier activation side effects. This ensures the classifier state is consistent with the mode transition. The full lifecycle is managed in `permissionSetup.ts`.

## Mode Gating

### Availability Conditions

The tool is disabled under these conditions:

```typescript
isEnabled() {
  if (
    (feature('KAIROS') || feature('KAIROS_CHANNELS')) &&
    getAllowedChannels().length > 0
  ) {
    return false
  }
  return true
}
```

When the `--channels` feature flag is active (Telegram/Discord integration), both EnterPlanMode and ExitPlanMode are disabled. This prevents plan mode from becoming a trap — the model could enter plan mode but the approval dialog would hang because the terminal UI is not available on messaging platforms.

### Tool Properties

| Property | Value | Rationale |
|----------|-------|-----------|
| `shouldDefer` | `true` | Requires user approval before executing |
| `isConcurrencySafe` | `true` | Mode transition is atomic |
| `isReadOnly` | `true` | Does not modify files |
| `userFacingName` | `''` (empty) | Internal tool name only |

## Plan Mode Workflow Instructions

### Result Block Content

After the tool succeeds, `mapToolResultToToolResultBlockParam()` returns instructions that guide the model's behavior in plan mode. The content differs based on whether the interview phase feature is enabled:

**Without interview phase** (standard mode):
```
Entered plan mode. You should now focus on exploring the codebase and designing an implementation approach.

In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.
```

**With interview phase** (`isPlanModeInterviewPhaseEnabled()`):
```
Entered plan mode. You should now focus on exploring the codebase and designing an implementation approach.

DO NOT write or edit any files except the plan file. Detailed workflow instructions will follow.
```

The interview phase variant defers detailed instructions to a separate `plan_mode` attachment in the system messages.

## Prompt Guidance

### When to Use (External User Variant)

The prompt provides seven conditions where EnterPlanMode should be preferred:

1. **New Feature Implementation** — Adding meaningful functionality
2. **Multiple Valid Approaches** — Several different solutions possible
3. **Code Modifications** — Changes affecting existing behavior
4. **Architectural Decisions** — Choosing between patterns/technologies
5. **Multi-File Changes** — Tasks touching more than 2-3 files
6. **Unclear Requirements** — Need exploration before understanding scope
7. **User Preferences Matter** — Implementation could go multiple ways

### When NOT to Use

- Single-line or few-line fixes (typos, obvious bugs, small tweaks)
- Adding a single function with clear requirements
- Tasks with very specific, detailed instructions from the user
- Pure research/exploration tasks (use Agent tool instead)

### Ant Variant

When `USER_TYPE === 'ant'`, a more conservative prompt is used that requires genuine architectural ambiguity before suggesting plan mode. The Ant variant:
- Narrows the criteria to only significant ambiguity
- Treats "can we work on X" as a signal to start, not plan
- Prefers using AskUserQuestion over full planning for minor questions

### What Happens Section

The `WHAT_HAPPENS_SECTION` describing plan mode behavior is conditionally included:
- Omitted when `isPlanModeInterviewPhaseEnabled()` is true (instructions come via attachment)
- Included otherwise as part of the tool prompt

## UI Rendering

### Tool Use Message

`renderToolUseMessage()` returns `null` — no intermediate "using" message is displayed.

### Tool Result Message

```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> Entered plan mode</Text>
  </Box>
  <Box paddingLeft={2}>
    <Text dimColor>
      Claude is now exploring and designing an implementation approach.
    </Text>
  </Box>
</Box>
```

Displays a colored indicator dot (plan mode color) with the status text and a dimmed description.

### Tool Use Rejected Message

```tsx
<Box flexDirection="row" marginTop={1}>
  <Text color={getModeColor('default')}>{BLACK_CIRCLE}</Text>
  <Text> User declined to enter plan mode</Text>
</Box>
```

Shows in the default mode color when the user rejects the plan mode request.

## Integration Points

### `bootstrap/state.js`

- `handlePlanModeTransition(from, to)` — Validates and logs mode transitions
- `getAllowedChannels()` — Returns configured channel list for gating

### `utils/permissions/permissionSetup.js`

- `prepareContextForPlanMode(context)` — Prepares permission context for plan mode, strips dangerous rules, preserves pre-plan mode state

### `utils/permissions/PermissionUpdate.js`

- `applyPermissionUpdate(context, update)` — Applies the `{ type: 'setMode', mode: 'plan', destination: 'session' }` update atomically

### `utils/planModeV2.js`

- `isPlanModeInterviewPhaseEnabled()` — Feature flag check for interview phase behavior

### `utils/plans.js`

- Plan file path resolution and management (used by ExitPlanMode, referenced in workflow instructions)

## Data Flow

```
Model decides task needs planning
    |
    v
EnterPlanMode tool call (no params)
    |
    v
┌─────────────────────────────────────────┐
│ VALIDATION                              │
│ - Check: not in agent context           │
│ - Check: tool is enabled (no channels)  │
└─────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────┐
│ MODE TRANSITION                         │
│                                         │
│ handlePlanModeTransition(current, plan) │
│                                         │
│ prepareContextForPlanMode():            │
│   - Strip dangerous permissions         │
│   - Preserve prePlanMode                │
│   - Run classifier side effects         │
│                                         │
│ applyPermissionUpdate():                │
│   - Set mode to 'plan'                  │
│   - Destination: 'session'              │
└─────────────────────────────────────────┘
    |
    v
State updated: mode = 'plan'
    |
    v
Result block with workflow instructions:
  - Without interview phase: 6-step guide
  - With interview phase: brief + deferred
    |
    v
Model enters read-only exploration phase
```
