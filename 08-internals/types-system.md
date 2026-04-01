# Types System

## Purpose

TypeScript type definitions that form the foundation of the application's type safety. Includes branded IDs, command types, hook types, log/message types, permission types, plugin types, and text input types. The `generated/` subdirectory contains protobuf-generated types.

## Location

`restored-src/src/types/`

## Key Files

### `ids.ts` — Branded ID Types

Prevents accidentally mixing session IDs and agent IDs at compile time using TypeScript's branded types pattern.

**Types:**
- `SessionId`: `string & { readonly __brand: 'SessionId' }`
- `AgentId`: `string & { readonly __brand: 'AgentId' }`

**Functions:**
- `asSessionId(id)`: Unsafe cast from string to SessionId
- `asAgentId(id)`: Unsafe cast from string to AgentId
- `toAgentId(s)`: Validates against `AGENT_ID_PATTERN` (`/^a(?:.+-)?[0-9a-f]{16}$/`) — returns `AgentId | null`

### `command.ts` — Command Types

Defines the command system's type hierarchy.

**Command types:**
- `PromptCommand`: Sends a prompt to the model (skill-like)
- `LocalCommand`: Lazy-loaded native command implementation
- `LocalJSXCommand`: Lazy-loaded JSX-rendering command (interactive dialogs)
- `Command`: Union of all three with `CommandBase`

**Command base properties:**
- `availability`: Auth/provider environments where command is visible (`'claude-ai' | 'console'`)
- `isEnabled`: Dynamic enablement check (feature flags, env checks)
- `isHidden`: Hide from typeahead/help
- `isSensitive`: Args redacted from conversation history
- `immediate`: Execute without waiting for stop point
- `kind`: `'workflow'` for workflow-backed commands

**Context types:**
- `LocalJSXCommandContext`: Extended ToolUseContext with `canUseTool`, `setMessages`, `onChangeAPIKey`, `resume`, etc.
- `LocalJSXCommandOnDone`: Callback when command completes (with display, shouldQuery, metaMessages options)
- `ResumeEntrypoint`: How a session was resumed (`cli_flag`, `slash_command_picker`, etc.)

### `hooks.ts` — Hook Types

Types for the hook system, including callback hooks, permission hooks, and hook results.

**Key types:**
- `HookCallback`: SDK callback hook with `(input, toolUseID, abort, hookIndex, context) => Promise<HookJSONOutput>`
- `HookCallbackMatcher`: `{ matcher?, hooks[], pluginName? }` — matches events to callbacks
- `HookCallbackContext`: `{ getAppState, updateAttributionState }` — state access for callbacks
- `HookProgress`: Progress indicator for running hooks
- `HookBlockingError`: Blocking error from hook execution
- `PermissionRequestResult`: Allow/deny decision from PermissionRequest hook
- `HookResult`: Aggregated result from a single hook execution
- `AggregatedHookResult`: Combined result from multiple hooks

**Hook response schemas:**
- `promptRequestSchema` / `PromptRequest`: Prompt elicitation protocol
- `syncHookResponseSchema`: Sync hook response with event-specific `hookSpecificOutput`
- `hookJSONOutputSchema`: Union of async and sync hook responses

**Hook events covered by `hookSpecificOutput`:**
`PreToolUse`, `UserPromptSubmit`, `SessionStart`, `Setup`, `SubagentStart`, `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Notification`, `PermissionRequest`, `Elicitation`, `ElicitationResult`, `CwdChanged`, `FileChanged`, `WorktreeCreate`

### `logs.ts` — Log and Message Types

Defines the transcript/log entry types for session persistence and resume.

**Core types:**
- `SerializedMessage`: Message with session metadata (cwd, userType, sessionId, timestamp, version)
- `LogOption`: Full log entry with summary, tags, PR links, file history, attribution, and context collapse data
- `TranscriptMessage`: Serialized message with parent UUIDs and sidechain info

**Specialized message types (discriminated by `type`):**
- `SummaryMessage`: AI-generated session summary
- `CustomTitleMessage`: User-set custom title
- `AiTitleMessage`: AI-generated title (distinct from custom)
- `LastPromptMessage`: Last user prompt for `claude ps`
- `TaskSummaryMessage`: Periodic task summary (every min(5 steps, 2min))
- `TagMessage`: Session tag
- `AgentNameMessage` / `AgentColorMessage` / `AgentSettingMessage`: Agent metadata
- `PRLinkMessage`: GitHub PR link
- `ModeEntry`: Session mode (coordinator/normal)
- `WorktreeStateEntry`: Worktree session state for resume
- `ContentReplacementEntry`: Content block replacement records for prompt cache
- `FileHistorySnapshotMessage`: File state snapshots
- `AttributionSnapshotMessage`: Per-file character contribution tracking
- `SpeculationAcceptMessage`: Speculation acceptance record
- `ContextCollapseCommitEntry`: Persisted context collapse commit (`type: 'marble-origami-commit'`)
- `ContextCollapseSnapshotEntry`: Staged queue snapshot (`type: 'marble-origami-snapshot'`)

**Entry union:** All message types united as `Entry`

### `permissions.ts` — Permission Types

Pure permission type definitions (no runtime dependencies) extracted to break import cycles.

**Permission modes:**
- External: `'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'`
- Internal: External + `'auto' | 'bubble'`
- Runtime set: External + `'auto'` (conditional on `TRANSCRIPT_CLASSIFIER` flag)

**Permission behaviors:** `'allow' | 'deny' | 'ask'`

**Permission rules:**
- `PermissionRule`: `{ source, ruleBehavior, ruleValue }`
- `PermissionRuleSource`: Where rule originated (userSettings, projectSettings, localSettings, flagSettings, policySettings, cliArg, command, session)
- `PermissionRuleValue`: `{ toolName, ruleContent? }`

**Permission updates:**
- `PermissionUpdate`: Discriminated union — `addRules | replaceRules | removeRules | setMode | addDirectories | removeDirectories`
- `PermissionUpdateDestination`: Where to persist (userSettings, projectSettings, localSettings, session, cliArg)

**Permission decisions:**
- `PermissionAllowDecision`: `{ behavior: 'allow', updatedInput?, userModified?, decisionReason?, ... }`
- `PermissionAskDecision`: `{ behavior: 'ask', message, suggestions?, pendingClassifierCheck?, ... }`
- `PermissionDenyDecision`: `{ behavior: 'deny', message, decisionReason }`
- `PermissionDecision`: Union of allow/ask/deny
- `PermissionResult`: Decision + `passthrough` variant

**Decision reasons (discriminated union):**
`rule`, `mode`, `subcommandResults`, `permissionPromptTool`, `hook`, `asyncAgent`, `sandboxOverride`, `classifier`, `workingDir`, `safetyCheck`, `other`

**Classifier types:**
- `ClassifierResult`: `{ matches, matchedDescription?, confidence, reason }`
- `YoloClassifierResult`: Detailed 2-stage classifier result with thinking, usage, duration, request IDs, prompt lengths, and stage breakdown

**Tool permission context:**
- `ToolPermissionContext`: Immutable snapshot of mode, rules (alwaysAllow/alwaysDeny/alwaysAsk by source), working directories, and bypass flags

### `plugin.ts` — Plugin Types

Types for the plugin system including definitions, loaded plugins, and errors.

**Key types:**
- `BuiltinPluginDefinition`: Built-in plugin with name, description, skills, hooks, MCP servers
- `PluginRepository`: Git repository info (url, branch, commitSha)
- `LoadedPlugin`: Fully loaded plugin with all component paths and configurations
- `PluginComponent`: `'commands' | 'agents' | 'skills' | 'hooks' | 'output-styles'`

**PluginError:** Discriminated union of 25+ error types covering:
- Loading: `path-not-found`, `git-auth-failed`, `git-timeout`, `network-error`
- Manifest: `manifest-parse-error`, `manifest-validation-error`
- Marketplace: `plugin-not-found`, `marketplace-not-found`, `marketplace-load-failed`, `marketplace-blocked-by-policy`
- MCP: `mcp-config-invalid`, `mcp-server-suppressed-duplicate`, `mcpb-download-failed`, `mcpb-extract-failed`, `mcpb-invalid-manifest`
- LSP: `lsp-config-invalid`, `lsp-server-start-failed`, `lsp-server-crashed`, `lsp-request-timeout`, `lsp-request-failed`
- Hooks: `hook-load-failed`
- Components: `component-load-failed`
- Dependencies: `dependency-unsatisfied`
- Cache: `plugin-cache-miss`
- Generic: `generic-error`

**Helper:**
- `getPluginErrorMessage(error)`: Returns human-readable error message for any PluginError type

### `textInputTypes.ts` — Text Input Types

Types for the terminal text input system including vim mode and command queue.

**Key types:**
- `BaseTextInputProps`: 30+ props for text input (value, onChange, onSubmit, cursor, highlights, ghost text, paste handling, etc.)
- `VimTextInputProps`: Extends base with vim mode support
- `VimMode`: `'INSERT' | 'NORMAL'`
- `TextInputState` / `VimInputState`: Input state with cursor position, viewport, paste state
- `PromptInputMode`: `'bash' | 'prompt' | 'orphaned-permission' | 'task-notification'`
- `QueuePriority`: `'now' | 'next' | 'later'` — interrupt levels for the command queue
- `QueuedCommand`: Command with value, mode, priority, pasted contents, origin, workload, agentId
- `OrphanedPermission`: Permission result without an active tool context

**Helpers:**
- `isValidImagePaste(c)`: Type guard for non-empty image pastes
- `getImagePasteIds(pastedContents)`: Extracts image paste IDs from a command

### `generated/` — Generated Types

Protobuf-generated types organized by namespace:

- `events_mono/claude_code/` — Claude Code event types
- `events_mono/common/` — Common event types
- `events_mono/growthbook/` — GrowthBook feature flag types
- `events_mono/protobuf/` — Protobuf message types
- `google/` — Google protobuf well-known types

## Dependencies

### Internal Dependencies

- `Tool.js` — ToolUseContext reference
- `state/AppState.js` — AppState reference
- `utils/permissions/*` — Permission rule/behavior schemas
- `entrypoints/agentSdkTypes.js` — Hook events, permission types
- `ink.js` — Key type

### External Dependencies

- `@anthropic-ai/sdk` — ContentBlockParam type
- `crypto` — UUID type
- `type-fest` — `IsEqual` for compile-time assertions
- `bun:bundle` — Feature flag checks

## Implementation Details

### Branded Types

`SessionId` and `AgentId` use TypeScript's nominal typing via intersection with a branded object. This prevents passing a raw string where a SessionId is expected without an explicit cast.

### Discriminated Unions

Most complex types use discriminated unions with a `type` or `behavior` field for exhaustive type checking:
- `PermissionDecision`: discriminated by `behavior`
- `PermissionUpdate`: discriminated by `type`
- `PermissionDecisionReason`: discriminated by `type`
- `PluginError`: discriminated by `type`
- `Entry` (log messages): discriminated by `type`

### Compile-Time Assertions

`types/hooks.ts` uses `type-fest.IsEqual` to assert that SDK types match Zod schema types at compile time:
```typescript
type _assertSDKTypesMatch = Assert<IsEqual<SchemaHookJSONOutput, HookJSONOutput>>
```

## Related Modules

- [Schemas](./schemas.md) — Zod schemas validate against these types
- [Bootstrap](./bootstrap.md) — Uses SessionId and AgentId types
- [Hooks System](./hooks-system.md) — Consumes hook and permission types
