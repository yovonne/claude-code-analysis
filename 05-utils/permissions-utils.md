# Permissions Utilities

**Directory:** `restored-src/src/utils/permissions/`

## Purpose

Implements the permission system for tool execution, including rule parsing, classification, auto-mode AI decisions, path validation, and denial tracking.

## Core Types

### PermissionRule
```typescript
{
  source: PermissionRuleSource
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue  // { toolName, ruleContent? }
}
```

### PermissionMode
- `default` - Standard permission prompts
- `acceptEdits` - Auto-accept file edits in working directory
- `plan` - Plan mode (no tool execution)
- `bypassPermissions` - Skip permission prompts
- `dontAsk` - Auto-deny everything
- `auto` - AI classifier decides (ant-only)

## permissions.ts

### Main Entry Point

- `hasPermissionsToUseTool(tool, input, context, assistantMessage, toolUseID)` - Primary permission check function
  - Step 1: Check deny rules, ask rules, tool implementation checks, safety checks
  - Step 2a: Check bypass permissions mode
  - Step 2b: Check always-allowed rules
  - Step 3: Convert passthrough to ask
  - Auto-mode integration: AI classifier for tool decisions
  - Headless agent support: PermissionRequest hooks for non-interactive contexts

### Rule Extraction

- `getAllowRules(context)` - Extracts all allow rules from all sources
- `getDenyRules(context)` - Extracts all deny rules
- `getAskRules(context)` - Extracts all ask rules
- `toolAlwaysAllowedRule(context, tool)` - Checks if entire tool is allowed
- `getDenyRuleForTool(context, tool)` - Checks if tool is denied
- `getRuleByContentsForTool(context, tool, behavior)` - Maps rule contents to rules

### Auto-Mode Integration

- Classifies tool actions using AI when in auto mode
- Fast-paths: acceptEdits mode check, safe-tool allowlist
- Denial tracking: falls back to prompting after 3 consecutive or 20 total denials
- Headless mode: auto-denies when prompts unavailable

### Permission Rule Editing

- `deletePermissionRule()` - Removes a rule from appropriate destination
- `applyPermissionRulesToPermissionContext()` - Applies rules to context

## permissionRuleParser.ts

### String to RuleValue Conversion

- `permissionRuleValueFromString(ruleString)` - Parses "Tool(content)" format
  - Handles escaped parentheses
  - Normalizes legacy tool names (Task to Agent, KillShell to TaskStop)
  - Empty or wildcard content treated as tool-wide rule

- `permissionRuleValueToString(ruleValue)` - Converts back to string format
  - Escapes parentheses in content

- `escapeRuleContent(content)` / `unescapeRuleContent(content)` - Escape utilities

## permissionsLoader.ts

### Loading Rules from Disk

- `loadAllPermissionRulesFromDisk()` - Loads rules from all enabled sources
  - Respects allowManagedPermissionRulesOnly flag
- `getPermissionRulesForSource(source)` - Loads rules from specific source
- `addPermissionRulesToSettings()` - Adds rules to settings file
- `deletePermissionRuleFromSettings()` - Removes rule from settings
- `shouldAllowManagedPermissionRulesOnly()` - Checks if only managed rules apply
- `shouldShowAlwaysAllowOptions()` - Inverse of above

## PermissionMode.ts

### Mode Configuration

- `permissionModeTitle(mode)` - Human-readable title
- `permissionModeShortTitle(mode)` - Short title for UI
- `permissionModeSymbol(mode)` - Symbol for display
- `permissionModeFromString(str)` - Parses string to mode enum
- `toExternalPermissionMode(mode)` - Converts to external-compatible mode
- `isExternalPermissionMode(mode)` - Type guard

## permissionExplainer.ts

### AI-Powered Explanations

- `generatePermissionExplanation()` - Uses Haiku to explain commands
  - Returns risk level (LOW/MEDIUM/HIGH), explanation, reasoning, risk
  - Uses forced tool choice for structured output
  - Extracts conversation context for reasoning
- `isPermissionExplainerEnabled()` - Checks global config flag

## pathValidation.ts

### Path Permission Checking

- `isPathAllowed(resolvedPath, context, operationType)` - Core path permission check
  - Step 1: Check deny rules
  - Step 2: Check internal editable paths
  - Step 2.5: Safety checks (dangerous files, Claude config)
  - Step 3: Check working directory plus acceptEdits mode
  - Step 3.5: Check internal readable paths
  - Step 3.7: Check sandbox write allowlist
  - Step 4: Check allow rules

- `validatePath(path, cwd, context, operationType)` - Full validation with tilde/glob handling
- `validateGlobPattern(cleanPath, cwd, context, operationType)` - Validates glob base directory
- `isDangerousRemovalPath(resolvedPath)` - Checks for dangerous rm targets (/, ~, drive roots)
- `expandTilde(path)` - Expands tilde to home directory
- `getGlobBaseDirectory(path)` - Extracts base directory from glob pattern
- `isPathInSandboxWriteAllowlist(resolvedPath)` - Checks sandbox write allowlist

## getNextPermissionMode.ts

### Mode Cycling

- `getNextPermissionMode(context)` - Determines next mode in Shift+Tab cycle
- `cyclePermissionMode(context)` - Computes next mode and prepares context

## dangerousPatterns.ts

### Dangerous Command Patterns

- `DANGEROUS_BASH_PATTERNS` - List of dangerous bash command prefixes
  - Cross-platform: python, node, npx, bash, ssh, etc.
  - Unix-specific: zsh, fish, eval, exec, sudo, etc.
  - Ant-only: fa run, coo, gh, curl, kubectl, aws, gcloud

## autoModeState.ts

### State Management

- `setAutoModeActive(active)` / `isAutoModeActive()` - Active state flag
- `setAutoModeFlagCli(passed)` / `getAutoModeFlagCli()` - CLI flag state
- `setAutoModeCircuitBroken(broken)` / `isAutoModeCircuitBroken()` - Circuit breaker

## denialTracking.ts

### Denial Limits

- `createDenialTrackingState()` - Initial state (0 consecutive, 0 total)
- `recordDenial(state)` - Increments both counters
- `recordSuccess(state)` - Resets consecutive counter
- `shouldFallbackToPrompting(state)` - Returns true if 3 consecutive or 20 total denials

## bypassPermissionsKillswitch.ts

### Runtime Killswitch

- `checkAndDisableBypassPermissionsIfNeeded()` - Disables bypass based on Statsig gate
- `checkAndDisableAutoModeIfNeeded()` - Disables auto-mode based on gate/settings
- React hooks for component integration

## filesystem.ts

### File Permission Implementation

- `checkWritePermissionForTool()` - Tool-level write permission check
- `checkReadPermissionForTool()` - Tool-level read permission check
- `pathInWorkingPath()` - Checks if path is within working directory
- `pathInAllowedWorkingPath()` - Checks with additional directories
- `matchingRuleForInput()` - Finds matching rule for a path input
- `checkEditableInternalPath()` - Checks internal editable paths
- `checkReadableInternalPath()` - Checks internal readable paths
- `checkPathSafetyForAutoEdit()` - Comprehensive safety validation

## Type Definition Files

### PermissionResult.ts / PermissionRule.ts / PermissionUpdate.ts / PermissionUpdateSchema.ts

- Schema definitions for permission results, rules, and updates
- `applyPermissionUpdate()` / `applyPermissionUpdates()` - Apply updates to context
- `persistPermissionUpdates()` - Persist updates to settings

## AI Classification (Feature-gated)

### classifierDecision.ts / classifierShared.ts / yoloClassifier.ts / bashClassifier.ts

- `classifyYoloAction()` - Classifies actions for auto-mode
- `classifyBashCommand()` - Classifies bash commands for permission rules
- `formatActionForClassifier()` - Formats action for classifier input

## Other Files

- `shadowedRuleDetection.ts` - Detects overlapping permission rules
- `permissionSetup.ts` - Mode management and transitions
- `PermissionPromptToolResultSchema.ts` - Prompt tool result schema
