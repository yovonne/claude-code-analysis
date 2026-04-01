# Hooks Utils

## Purpose

The hooks system provides a comprehensive lifecycle event mechanism that allows external commands, prompts, HTTP endpoints, and TypeScript functions to intercept and react to Claude Code events. This module family covers hook registration, execution, configuration management, SSRF protection, async hook lifecycle, and event broadcasting.

## Location

Core hook execution and management:
- `restored-src/src/utils/hooks/sessionHooks.ts` — Session-scoped hook registration and retrieval
- `restored-src/src/utils/hooks/hooksConfigManager.ts` — Hook event metadata, grouping, and priority sorting
- `restored-src/src/utils/hooks/hooksSettings.ts` — Hook source filtering (managed-only, plugin-only, disable-all policies)
- `restored-src/src/utils/hooks/hooksConfigSnapshot.ts` — Snapshot-based config caching for mid-session hook display
- `restored-src/src/utils/hooks/execAgentHook.ts` — Agent-based hook execution (multi-turn LLM query)
- `restored-src/src/utils/hooks/execHttpHook.ts` — HTTP hook execution with SSRF protection
- `restored-src/src/utils/hooks/execPromptHook.ts` — Prompt-based hook execution (single-turn LLM query)
- `restored-src/src/utils/hooks/hookHelpers.ts` — Shared utilities: argument substitution, structured output tool, schema
- `restored-src/src/utils/hooks/ssrfGuard.ts` — DNS-level SSRF guard for HTTP hooks

Async hook lifecycle:
- `restored-src/src/utils/hooks/AsyncHookRegistry.ts` — Registry for async (background) shell command hooks
- `restored-src/src/utils/hooks/hookEvents.ts` — Event broadcasting system for hook started/progress/response events
- `restored-src/src/utils/hooks/postSamplingHooks.ts` — Post-sampling hook registry (internal, not exposed in settings)

Registration and watching:
- `restored-src/src/utils/hooks/registerFrontmatterHooks.ts` — Register hooks from agent/skill frontmatter
- `restored-src/src/utils/hooks/registerSkillHooks.ts` — Register hooks from skill frontmatter (with once: true support)
- `restored-src/src/utils/hooks/fileChangedWatcher.ts` — Chokidar-based file watcher for CwdChanged/FileChanged hooks
- `restored-src/src/utils/hooks/skillImprovement.ts` — Automatic skill definition improvement via post-sampling LLM analysis

## Key Exports

### Session Hooks (`sessionHooks.ts`)

#### Types
- `FunctionHook`: In-memory TypeScript callback hook with embedded function. Session-scoped only, cannot be persisted.
- `FunctionHookCallback`: `(messages, signal?) => boolean | Promise<boolean>` — returns true if check passes
- `SessionHooksState`: `Map<string, SessionStore>` — uses Map (not Record) for O(1) mutation under high concurrency

#### Functions
- `addSessionHook(setAppState, sessionId, event, matcher, hook, onHookSuccess?, skillRoot?)`: Add a command/prompt/agent/http hook to the session
- `addFunctionHook(setAppState, sessionId, event, matcher, callback, errorMessage, options?)`: Add a function hook; returns hook ID for removal
- `removeFunctionHook(setAppState, sessionId, event, hookId)`: Remove a function hook by ID
- `removeSessionHook(setAppState, sessionId, event, hook)`: Remove a specific hook from the session
- `getSessionHooks(appState, sessionId, event?)`: Get all session hooks for an event (excluding function hooks)
- `getSessionFunctionHooks(appState, sessionId, event?)`: Get all session function hooks separately
- `getSessionHookCallback(appState, sessionId, event, matcher, hook)`: Get full hook entry including callbacks
- `clearSessionHooks(setAppState, sessionId)`: Clear all session hooks for a session

### Hook Execution

#### Agent Hooks (`execAgentHook.ts`)
- `execAgentHook(hook, hookName, hookEvent, jsonInput, signal, toolUseContext, toolUseID, messages, agentName?)`: Executes an agent-based hook using multi-turn LLM query with StructuredOutput tool enforcement. Supports up to 50 turns, uses small fast model by default.

#### HTTP Hooks (`execHttpHook.ts`)
- `execHttpHook(hook, hookEvent, jsonInput, signal?)`: POSTs hook input JSON to configured URL with SSRF protection, env var interpolation in headers, sandbox proxy routing, and URL allowlist enforcement.

#### Prompt Hooks (`execPromptHook.ts`)
- `execPromptHook(hook, hookName, hookEvent, jsonInput, signal, toolUseContext, messages?, toolUseID?)`: Executes a prompt-based hook using single-turn LLM query with JSON schema output enforcement.

### SSRF Guard (`ssrfGuard.ts`)

#### Functions
- `isBlockedAddress(address)`: Returns true if the address is in a blocked range (private, link-local, CGNAT). Allows loopback (127.0.0.0/8, ::1) for local dev hooks.
- `ssrfGuardedLookup(hostname, options, callback)`: dns.lookup-compatible function that validates resolved IPs before connection. Used as axios `lookup` option.

#### Blocked Ranges
- IPv4: 0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16
- IPv6: ::, fc00::/7, fe80::/10, IPv4-mapped addresses in blocked ranges
- Allowed: 127.0.0.0/8, ::1, everything else

### Async Hook Registry (`AsyncHookRegistry.ts`)

#### Types
- `PendingAsyncHook`: Tracks a running background shell command hook with process ID, timeout, progress interval, and response state

#### Functions
- `registerPendingAsyncHook(...)`: Register a new async hook with timeout and progress tracking
- `getPendingAsyncHooks()`: Get all pending hooks that haven't sent their response yet
- `checkForAsyncHookResponses()`: Poll all pending hooks for completed responses; parses JSON from stdout
- `removeDeliveredAsyncHooks(processIds)`: Remove hooks whose responses have been delivered
- `finalizePendingAsyncHooks()`: Kill and finalize all remaining hooks (session cleanup)

### Hook Events (`hookEvents.ts`)

#### Types
- `HookStartedEvent`, `HookProgressEvent`, `HookResponseEvent`: Event shapes for hook lifecycle
- `HookExecutionEvent`: Union of all event types
- `HookEventHandler`: `(event: HookExecutionEvent) => void`

#### Functions
- `registerHookEventHandler(handler)`: Register handler for hook events; drains pending events immediately
- `emitHookStarted(hookId, hookName, hookEvent)`: Emit hook started event
- `emitHookProgress(data)`: Emit hook progress event with stdout/stderr
- `startHookProgressInterval(params)`: Start periodic progress polling; returns stop function
- `emitHookResponse(data)`: Emit hook response event with exit code and outcome
- `setAllHookEventsEnabled(enabled)`: Enable emission of all hook event types (beyond SessionStart/Setup)

### Hook Config Manager (`hooksConfigManager.ts`)

#### Types
- `HookEventMetadata`: Summary, description, and matcher metadata for each hook event
- `MatcherMetadata`: Field to match and allowed values

#### Functions
- `getHookEventMetadata(toolNames)`: Memoized metadata for all 26+ hook events (PreToolUse, PostToolUse, Stop, SubagentStop, etc.)
- `groupHooksByEventAndMatcher(appState, toolNames)`: Group all hooks (settings + registered + plugin) by event and matcher
- `getSortedMatchersForEvent(hooksByEventAndMatcher, event)`: Sort matchers by source priority
- `getHooksForMatcher(hooksByEventAndMatcher, event, matcher)`: Get hooks for a specific event+matcher
- `getMatcherMetadata(event, toolNames)`: Get metadata for a specific event's matcher field

### Hook Settings (`hooksSettings.ts`)

#### Types
- `HookSource`: `'userSettings' | 'projectSettings' | 'localSettings' | 'policySettings' | 'pluginHook' | 'sessionHook' | 'builtinHook'`
- `IndividualHookConfig`: Normalized hook config with event, command, matcher, and source

#### Functions
- `isHookEqual(a, b)`: Compare two hooks by command/prompt content (not timeout)
- `getHookDisplayText(hook)`: Get display text for a hook
- `getAllHooks(appState)`: Collect hooks from all allowed sources respecting policy restrictions
- `getHooksForEvent(appState, event)`: Filter hooks for a specific event
- `shouldAllowManagedHooksOnly()`: Check if only managed hooks should run
- `shouldDisableAllHooksIncludingManaged()`: Check if ALL hooks (including managed) are disabled
- `sortMatchersByPriority(matchers, hooksByEventAndMatcher, selectedEvent)`: Sort matchers by source priority (user > project > local > plugin)

### Hook Config Snapshot (`hooksConfigSnapshot.ts`)

#### Functions
- `captureHooksConfigSnapshot()`: Capture initial hooks config from allowed sources
- `updateHooksConfigSnapshot()`: Refresh snapshot (resets settings cache first)
- `getHooksConfigFromSnapshot()`: Get current snapshot (lazy-captures if absent)
- `resetHooksConfigSnapshot()`: Clear snapshot (testing)

### File Changed Watcher (`fileChangedWatcher.ts`)

#### Functions
- `initializeFileChangedWatcher(cwd)`: Initialize chokidar watcher for CwdChanged/FileChanged hooks
- `updateWatchPaths(paths)`: Dynamically update watched paths from hook output
- `onCwdChangedForHooks(oldCwd, newCwd)`: Handle working directory changes; re-resolve paths and re-execute CwdChanged hooks
- `setEnvHookNotifier(cb)`: Set callback for env hook notifications
- `resetFileChangedWatcherForTesting()`: Reset watcher state

### Skill Improvement (`skillImprovement.ts`)

#### Types
- `SkillUpdate`: `{ section, change, reason }` — a suggested improvement to a skill definition

#### Functions
- `initSkillImprovement()`: Register the skill improvement post-sampling hook (gated by feature flags)
- `applySkillImprovement(skillName, updates)`: Apply suggested improvements to a skill file via LLM rewrite (fire-and-forget)

### Registration Helpers

- `registerFrontmatterHooks(setAppState, sessionId, hooks, sourceName, isAgent?)`: Register hooks from agent/skill frontmatter; converts Stop→SubagentStop for agents
- `registerSkillHooks(setAppState, sessionId, hooks, skillName, skillRoot?)`: Register skill hooks with `once: true` auto-removal support

### Hook Helpers (`hookHelpers.ts`)

- `hookResponseSchema`: Zod schema for hook responses (`{ ok: boolean, reason?: string }`)
- `addArgumentsToPrompt(prompt, jsonInput)`: Replace `$ARGUMENTS` placeholders in hook prompts
- `createStructuredOutputTool()`: Create a StructuredOutput tool for hook responses
- `registerStructuredOutputEnforcement(setAppState, sessionId)`: Register a Stop hook that enforces StructuredOutput tool usage

### Post-Sampling Hooks (`postSamplingHooks.ts`)

- `registerPostSamplingHook(hook)`: Register an internal post-sampling hook (not exposed in settings)
- `executePostSamplingHooks(messages, systemPrompt, userContext, systemContext, toolUseContext, querySource?)`: Execute all registered hooks
- `clearPostSamplingHooks()`: Clear all hooks (testing)

## Hook Event Types

The system supports 26+ event types including: PreToolUse, PostToolUse, PostToolUseFailure, PermissionDenied, Notification, UserPromptSubmit, SessionStart, SessionEnd, Stop, StopFailure, SubagentStart, SubagentStop, PreCompact, PostCompact, PermissionRequest, Setup, TeammateIdle, TaskCreated, TaskCompleted, Elicitation, ElicitationResult, ConfigChange, InstructionsLoaded, WorktreeCreate, WorktreeRemove, CwdChanged, FileChanged.

## Design Notes

- Session hooks use `Map` instead of `Record` for O(1) mutation under high concurrency (parallel agents registering hooks in one tick)
- HTTP hooks have comprehensive SSRF protection: URL allowlist, DNS-level IP validation (blocking private/link-local ranges), env var allowlist for header interpolation, and sandbox proxy routing
- Agent hooks run as multi-turn LLM queries with StructuredOutput enforcement and a 50-turn limit
- Async hooks (background shell commands) are tracked in a global registry with progress polling and timeout handling
- The SSRF guard allows loopback (127.0.0.0/8, ::1) intentionally — local dev policy servers are a primary HTTP hook use case
- Skill improvement runs every 5 user messages as a post-sampling hook, analyzing conversation for preferences that should be baked into skill definitions
