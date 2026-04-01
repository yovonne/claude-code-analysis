# Permission Flow

## Purpose

Traces how tool execution permissions are evaluated, from the initial tool call through rule matching, classifier checks, user prompts, and final allow/deny decisions.

## Location

Primary sources:
- `restored-src/src/types/permissions.ts` — permission type definitions
- `restored-src/src/Tool.ts` — `ToolPermissionContext`, `checkPermissions`, `PermissionResult`
- `restored-src/src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx` — permission prompt example
- `restored-src/src/utils/permissions/permissionSetup.ts` — initialization
- `restored-src/src/utils/permissions/permissions.ts` — core permission evaluation
- `restored-src/src/utils/permissions/denialTracking.ts` — denial counting
- `restored-src/src/hooks/useCanUseTool.ts` — permission decision entry point

---

## Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Standard permission checking with rules |
| `bypassPermissions` | Skip all permission prompts (dangerous) |
| `acceptEdits` | Auto-approve file edits, prompt for destructive ops |
| `dontAsk` | Auto-approve all non-destructive operations |
| `plan` | Plan mode — no tool execution, only planning |
| `auto` | AI classifier evaluates each command for safety |
| `bubble` | Permissions handled via external bubble (remote/bridge) |

---

## Permission Rule System

### Rule Sources (in priority order)

1. `cliArg` — command-line flags (`--allowedTool`)
2. `session` — session-scoped rules (from plan mode exit)
3. `flagSettings` — `--settings` flag overrides
4. `policySettings` — enterprise policy restrictions
5. `userSettings` — `~/.claude/settings.json`
6. `projectSettings` — `.claude/settings.json` in project root
7. `localSettings` — `.claude/settings.local.json`
8. `command` — per-command built-in permissions

### Rule Structure

```typescript
type PermissionRule = {
  source: PermissionRuleSource    // where the rule came from
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string              // e.g., "Bash", "FileWrite"
    ruleContent?: string          // e.g., "ls *" (pattern match)
  }
}
```

### Rule Categories in ToolPermissionContext

```
ToolPermissionContext {
  mode: PermissionMode
  alwaysAllowRules: ToolPermissionRulesBySource   // auto-approved
  alwaysDenyRules: ToolPermissionRulesBySource    // always blocked
  alwaysAskRules: ToolPermissionRulesBySource     // always prompt
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource  // removed in auto mode
}
```

---

## Complete Permission Evaluation Flow

### Phase 1: Tool Call Initiation

```
API returns tool_use block
  │
  ▼
canUseTool(tool, input, context, assistantMessage, toolUseID)
  │
  ├─► 1. validateInput(tool, input)
  │     │
  │     ├─► tool.validateInput(input, context)
  │     │    └─► Returns { result: true } or { result: false, message, errorCode }
  │     │
  │     └─► If validation fails → deny with error message
  │
  └─► 2. checkPermissions(tool, input, context)
```

### Phase 2: Permission Decision Chain

```
checkPermissions(tool, input, context)
  │
  ├─► A. Tool-specific checkPermissions()
  │     │
  │     ├─► Each tool can override checkPermissions()
  │     ├─► Default: { behavior: 'allow', updatedInput }
  │     ├─► AskUserQuestionTool: { behavior: 'ask', message: 'Answer questions?' }
  │     └─► BashTool: complex command analysis
  │
  ├─► B. Hook evaluation (PreToolUse hooks)
  │     │
  │     ├─► Execute PreToolUse hooks in order
  │     ├─► Hooks can: allow, deny, ask, or modify input
  │     └─► First decisive hook wins
  │
  ├─► C. Permission rule matching
  │     │
  │     ├─► Check alwaysDenyRules (tool name + pattern)
  │     ├─► Check alwaysAllowRules (tool name + pattern)
  │     ├─► Check alwaysAskRules (tool name + pattern)
  │     └─► Apply mode-based defaults
  │
  ├─► D. Mode-based evaluation
  │     │
  │     ├─► bypassPermissions → allow all
  │     ├─► dontAsk → allow non-destructive
  │     ├─► acceptEdits → allow file edits, prompt for Bash/Write
  │     ├─► plan → deny all tool execution
  │     └─► auto → classifier evaluation (see below)
  │
  ├─► E. Safety checks
  │     │
  │     ├─► Working directory validation
  │     ├─► Sandbox command exclusions
  │     ├─► Windows path bypass detection
  │     └─► Cross-machine bridge message validation
  │
  └─► F. Final decision
        │
        ├─► 'allow' → execute tool
        ├─► 'deny' → return error to API
        └─► 'ask' → show permission dialog
```

### Phase 3: Auto Mode Classifier

```
Auto mode enabled?
  │
  ▼
Build classifier transcript
  │
  ├─► System prompt (classifier instructions)
  ├─► Tool call descriptions (toAutoClassifierInput)
  ├─► Recent user prompts
  └─► Conversation context
  │
  ▼
Call classifier API (fast stage)
  │
  ├─► If fast stage confident → return decision
  └─► If uncertain → thinking stage (deeper analysis)
  │
  ▼
ClassifierResult {
  matches: boolean
  matchedDescription?: string
  confidence: 'high' | 'medium' | 'low'
  reason: string
}
  │
  ▼
YoloClassifierResult {
  shouldBlock: boolean
  reason: string
  thinking?: string
  unavailable?: boolean
  transcriptTooLong?: boolean
  model: string
  usage?: ClassifierUsage
  stage?: 'fast' | 'thinking'
}
  │
  ├─► shouldBlock === false → allow
  ├─► shouldBlock === true → deny
  └─► unavailable → fall back to normal prompting
```

### Phase 4: User Permission Dialog

```
Decision === 'ask'
  │
  ▼
Build permission prompt
  │
  ├─► tool.userFacingName(input)
  ├─► tool.getActivityDescription(input)
  ├─► Risk assessment (LOW/MEDIUM/HIGH)
  ├─► Permission explanation
  └─► Suggested updates (save as allow/deny rule)
  │
  ▼
Show permission dialog (REPL UI)
  │
  ├─► User selects: Allow / Allow & Remember / Deny / Deny & Remember
  │
  ├─► Allow → execute tool
  ├─► Allow & Remember → add to alwaysAllowRules
  ├─► Deny → return error to API
  └─► Deny & Remember → add to alwaysDenyRules
  │
  ▼
Track denial count (for classifier fallback)
  │
  └─► If denials exceed threshold → force prompt next time
```

---

## Permission Decision Types

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?; decisionReason?; toolUseID? }
  | { behavior: 'deny'; message: string; decisionReason: PermissionDecisionReason }
  | { behavior: 'ask'; message: string; suggestions?: PermissionUpdate[]; blockedPath? }
  | { behavior: 'passthrough'; message: string; pendingClassifierCheck? }

type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'workingDir'; reason: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'other'; reason: string }
```

---

## AskUserQuestionTool Permission Flow (Example)

```
Model calls AskUserQuestion tool
  │
  ▼
validateInput()
  │
  ├─► Check HTML preview validity (if previewFormat === 'html')
  ├─► Validate no <html>, <body>, <!DOCTYPE>, <script>, <style>
  └─► Return { result: true/false, message, errorCode }
  │
  ▼
checkPermissions()
  │
  └─► Always returns { behavior: 'ask', message: 'Answer questions?' }
      (This tool always requires user interaction)
  │
  ▼
Permission dialog shows:
  ├─► Question headers (chips)
  ├─► Options with labels and descriptions
  ├─► Optional preview content (markdown or HTML)
  └─► "Other" option for custom input
  │
  ▼
User responds → answers recorded
  │
  └─► ToolResult: { questions, answers, annotations? }
```

---

## Denial Tracking

```
DenialTrackingState {
  count: number          // consecutive denials
  threshold: number      // when to force prompt
  lastToolName: string   // last denied tool
}
```

- Each denial increments the counter
- When threshold exceeded → force permission prompt even in auto mode
- Reset on successful tool execution
- Used by classifier modes (YOLO, headless) as a safety fallback

---

## Integration Points

| Component | Role in Permission Flow |
|-----------|------------------------|
| `Tool.checkPermissions()` | Tool-specific permission logic |
| `PermissionMode` | Global permission behavior selector |
| `ToolPermissionContext` | Carries rules and mode through execution |
| `PreToolUse hooks` | External permission interceptors |
| `Classifier` | AI-powered safety evaluation (auto mode) |
| `DenialTrackingState` | Fallback safety mechanism |
| `AskUserQuestionTool` | Multi-choice user permission prompt |

## Related Documentation

- [Message Flow](./message-flow.md)
- [Tool Execution Flow](./tool-execution-flow.md)
- [Session Lifecycle](./session-lifecycle.md)
- [Permissions Utils](../05-utils/permissions-utils.md)
- [AskUserQuestionTool](../02-tools/AskUserQuestionTool.md)
