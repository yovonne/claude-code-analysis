# Session Lifecycle

## Purpose

Traces the complete lifecycle of a Claude Code session — from process startup through initialization, conversation turns, context management (compaction), and session termination.

## Location

Primary sources:
- `restored-src/src/main.tsx` — process entry, initialization, command routing
- `restored-src/src/state/AppState.tsx` / `AppStateStore.ts` — session state management
- `restored-src/src/QueryEngine.ts` — per-conversation turn management
- `restored-src/src/history.ts` — command history persistence
- `restored-src/src/assistant/sessionHistory.ts` — remote session history fetching
- `restored-src/src/services/compact/compact.ts` — conversation compaction
- `restored-src/src/services/compact/autoCompact.ts` — automatic compaction triggers
- `restored-src/src/services/compact/prompt.ts` — compaction prompt construction
- `restored-src/src/services/compact/postCompactCleanup.ts` — post-compaction cleanup

---

## Session Lifecycle States

```
Process Start
    │
    ▼
┌─────────────────────────────────────────────┐
│ 1. BOOTSTRAP                                │
│   - MDM/raw read prefetch                   │
│   - Keychain prefetch                       │
│   - Settings load (user, project, policy)   │
│   - Migration check                         │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 2. INITIALIZATION                           │
│   - init() — auth, telemetry, config        │
│   - Permission context setup                │
│   - Tool registry build                     │
│   - MCP server connections                  │
│   - Plugin loading                          │
│   - Skill discovery                         │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 3. SESSION START                            │
│   - Session ID generation                   │
│   - Transcript file creation                │
│   - SessionStart hooks                      │
│   - AppState store creation                 │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 4. CONVERSATION TURNS (loop)                │
│   - User input → QueryEngine.submitMessage  │
│   - API call → streaming response           │
│   - Tool execution (as needed)              │
│   - Transcript persistence                  │
│   - Auto-compact check                      │
│   - Result yielded                          │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 5. CONTEXT MANAGEMENT                       │
│   - Token usage tracking                    │
│   - Auto-compact trigger                    │
│   - Manual /compact command                 │
│   - Session memory compaction               │
│   - Partial compact (message selector)      │
│   - Snip compaction (HISTORY_SNIP feature)  │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 6. SESSION TERMINATION                      │
│   - User exit / interrupt                   │
│   - Graceful shutdown                       │
│   - Final transcript flush                  │
│   - History persistence                     │
│   - Cleanup registry execution              │
└─────────────────────────────────────────────┘
```

---

## Phase 1: Bootstrap

### Early Startup (< 100ms)

```
main.tsx module evaluation
  │
  ├─► profileCheckpoint('main_tsx_entry')
  │
  ├─► startMdmRawRead()        — parallel MDM subprocess (macOS)
  ├─► startKeychainPrefetch()  — parallel keychain reads (macOS)
  │
  └─► Import all modules (~135ms of parallel loading)
```

### Pre-Command Setup

```
main() function
  │
  ├─► Security: NoDefaultCurrentDirectoryInExePath = '1' (Windows)
  ├─► Signal handlers: SIGINT, exit
  ├─► URL/deep link handling (cc://, lodestone://)
  ├─► Special mode detection:
  │    ├─► `claude assistant [sessionId]` — assistant mode
  │    ├─► `claude ssh <host> [dir]` — SSH remote mode
  │    └─► `claude open <url>` — direct connect mode
  │
  ├─► Interactive vs non-interactive detection
  │    ├─► -p/--print flag → non-interactive (SDK mode)
  │    ├─► --init-only flag → non-interactive
  │    ├─► --sdk-url flag → non-interactive
  │    └─► !process.stdout.isTTY → non-interactive
  │
  ├─► Client type determination
  │    ├─► cli, sdk-cli, sdk-typescript, sdk-python
  │    ├─► github-action, claude-vscode, claude-desktop
  │    └─► remote, local-agent
  │
  └─► Eager settings load (--settings flag)
```

---

## Phase 2: Initialization

### Commander Hook (preAction)

```
program.hook('preAction')
  │
  ├─► await ensureMdmSettingsLoaded()
  ├─► await ensureKeychainPrefetchCompleted()
  │
  ├─► init()                     — core initialization
  │    ├─► Load settings (user, project, local, policy)
  │    ├─► Initialize telemetry
  │    ├─► Check auth status
  │    └─► Apply environment variables
  │
  ├─► process.title = 'claude'
  ├─► initSinks()                — attach logging sinks
  ├─► runMigrations()            — apply config migrations
  │
  ├─► loadRemoteManagedSettings()  — enterprise settings (non-blocking)
  ├─► loadPolicyLimits()          — policy limits (non-blocking)
  │
  └─► uploadUserSettings()        — settings sync (non-blocking, feature-gated)
```

### Deferred Prefetches (after first render)

```
startDeferredPrefetches()
  │
  ├─► initUser()                 — user info
  ├─► getUserContext()           — user context
  ├─► prefetchSystemContextIfSafe() — git status (only if trusted)
  ├─► getRelevantTips()          — contextual tips
  ├─► countFilesRoundedRg()      — file count for context estimation
  ├─► initializeAnalyticsGates() — feature flags
  ├─► prefetchOfficialMcpUrls()  — MCP registry
  ├─► refreshModelCapabilities() — model capabilities
  ├─► settingsChangeDetector.initialize() — settings hot-reload
  └─► skillChangeDetector.initialize() — skill hot-reload
```

---

## Phase 3: Session Start

### QueryEngine Creation

```
new QueryEngine(config)
  │
  ├─► config: {
  │     cwd, tools, commands, mcpClients, agents,
  │     canUseTool, getAppState, setAppState,
  │     initialMessages, readFileCache, ...
  │   }
  │
  ├─► mutableMessages = initialMessages ?? []
  ├─► abortController = config.abortController ?? createAbortController()
  ├─► permissionDenials = []
  ├─► totalUsage = EMPTY_USAGE
  └─► discoveredSkillNames = new Set()
```

### Session State (AppState)

```
getDefaultAppState()
  │
  └─► Returns complete initial state:
       ├─► settings: SettingsJson
       ├─► toolPermissionContext: ToolPermissionContext
       ├─► mainLoopModel: ModelSetting
       ├─► mcp: { clients, tools, commands, resources }
       ├─► plugins: { enabled, disabled, errors }
       ├─► tasks: { [taskId]: TaskState }
       ├─► todos: { [agentId]: TodoList }
       ├─► notifications: { current, queue }
       ├─► speculation: SpeculationState
       ├─► activeOverlays: Set<string>
       └─► ... (60+ fields)
```

---

## Phase 4: Conversation Turns

### Turn Lifecycle

```
QueryEngine.submitMessage(prompt)
  │
  ├─► 1. Clear per-turn tracking
  │    └─► discoveredSkillNames.clear()
  │
  ├─► 2. Build system prompt
  │    ├─► fetchSystemPromptParts()
  │    │    ├─► Default system prompt (or custom)
  │    │    ├─► User context (OS, shell, git, etc.)
  │    │    ├─► System context (working directory, etc.)
  │    │    └─► Memory mechanics prompt (if enabled)
  │    └─► asSystemPrompt([...parts])
  │
  ├─► 3. Process user input
  │    ├─► processUserInput()
  │    │    ├─► Parse slash commands
  │    │    ├─► Resolve attachments
  │    │    └─► Build message array
  │    └─► Push to mutableMessages
  │
  ├─► 4. Record transcript (before API)
  │    └─► recordTranscript(messages)
  │
  ├─► 5. Yield system init message
  │    └─► buildSystemInitMessage(tools, mcp, model, mode, ...)
  │
  ├─► 6. Enter query() loop
  │    │
  │    ├─► normalizeMessagesForAPI()
  │    ├─► queryModelWithStreaming()
  │    │    ├─► Stream events (content blocks)
  │    │    ├─► Track usage
  │    │    └─► Handle errors/retries
  │    │
  │    ├─► For each tool_use:
  │    │    ├─► canUseTool() → permission check
  │    │    ├─► tool.call() → execute
  │    │    └─► Inject tool_result as user message
  │    │
  │    └─► Continue until stop_reason === 'end_turn'
  │
  ├─► 7. Check termination conditions
  │    ├─► maxTurns reached → error result
  │    ├─► maxBudgetUsd exceeded → error result
  │    └─► structured output retries exceeded → error result
  │
  └─► 8. Yield final result message
       └─► { type: 'result', subtype: 'success'|'error_*', ... }
```

---

## Phase 5: Context Management

### Token Tracking

```
Every API response:
  │
  ├─► message_start → reset currentMessageUsage
  ├─► message_delta → accumulate usage
  └─► message_stop → add to totalUsage
  │
  ▼
totalUsage = {
  input_tokens, output_tokens,
  cache_read_input_tokens, cache_creation_input_tokens
}
```

### Auto-Compact Trigger

```
After each turn:
  │
  ▼
autoCompactIfNeeded(messages, context, cacheSafeParams, querySource)
  │
  ├─► Check suppression conditions:
  │    ├─► DISABLE_COMPACT env var
  │    ├─► DISABLE_AUTO_COMPACT env var
  │    ├─► userConfig.autoCompactEnabled === false
  │    ├─► querySource === 'compact' (prevent recursion)
  │    ├─► querySource === 'session_memory' (prevent deadlock)
  │    ├─► REACTIVE_COMPACT feature flag
  │    └─► CONTEXT_COLLAPSE feature (suppressed when active)
  │
  ├─► Calculate token count
  │    └─► tokenCountWithEstimation(messages) - snipTokensFreed
  │
  ├─► Calculate threshold
  │    ├─► effectiveContextWindow = contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY
  │    ├─► autoCompactThreshold = effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS
  │    │    └─► AUTOCOMPACT_BUFFER_TOKENS = 13,000
  │    └─► Check: tokenCount >= autoCompactThreshold
  │
  └─► If threshold exceeded:
       ├─► Try session memory compaction first
       │    └─► trySessionMemoryCompaction()
       │
       └─► If no session memory compact:
            └─► compactConversation()
```

### Compaction Process

```
compactConversation(messages, context, cacheSafeParams, ...)
  │
  ├─► 1. Pre-compact hooks
  │    └─► executePreCompactHooks()
  │
  ├─► 2. Build compact prompt
  │    ├─► NO_TOOLS_PREAMBLE (CRITICAL: respond with text only)
  │    ├─► BASE_COMPACT_PROMPT (9-section summary structure)
  │    ├─► Custom instructions (if any)
  │    └─► NO_TOOLS_TRAILER
  │
  ├─► 3. Stream compact summary
  │    ├─► Try forked agent (cache sharing)
  │    │    └─► runForkedAgent({ maxTurns: 1, skipCacheWrite: true })
  │    └─► Fallback: regular streaming
  │         └─► queryModelWithStreaming()
  │
  ├─► 4. Process summary
  │    ├─► Strip <analysis> section
  │    ├─► Extract <summary> section
  │    └─► formatCompactSummary()
  │
  ├─► 5. Clear caches
  │    ├─► context.readFileState.clear()
  │    └─► context.loadedNestedMemoryPaths?.clear()
  │
  ├─► 6. Generate post-compact attachments
  │    ├─► File attachments (recently read files)
  │    ├─► Async agent attachments
  │    ├─► Plan attachment (if in plan mode)
  │    ├─► Skill attachment (if skills used)
  │    ├─► Deferred tools delta
  │    ├─► Agent listing delta
  │    └─► MCP instructions delta
  │
  ├─► 7. Session start hooks
  │    └─► processSessionStartHooks('compact')
  │
  ├─► 8. Create boundary marker
  │    └─► createCompactBoundaryMessage('auto'|'manual', tokenCount, ...)
  │
  ├─► 9. Post-compact hooks
  │    └─► executePostCompactHooks()
  │
  ├─► 10. Build post-compact messages
  │     └─► [boundaryMarker, ...summaryMessages, ...attachments, ...hookResults]
  │
  └─► 11. Post-compact cleanup
       └─► runPostCompactCleanup()
```

### Token Warning Thresholds

```
calculateTokenWarningState(tokenUsage, model)
  │
  ├─► warningThreshold = threshold - WARNING_THRESHOLD_BUFFER_TOKENS (20,000)
  ├─► errorThreshold = threshold - ERROR_THRESHOLD_BUFFER_TOKENS (20,000)
  ├─► blockingLimit = effectiveWindow - MANUAL_COMPACT_BUFFER_TOKENS (3,000)
  │
  └─► Returns:
       ├─► percentLeft
       ├─► isAboveWarningThreshold
       ├─► isAboveErrorThreshold
       ├─► isAboveAutoCompactThreshold
       └─► isAtBlockingLimit
```

---

## Phase 6: Session Termination

### Graceful Shutdown

```
User exits (Ctrl+C, /exit, or process signal)
  │
  ▼
gracefulShutdown()
  │
  ├─► Abort in-flight API calls
  ├─► Wait for pending transcript flush
  ├─► Flush command history
  │    └─► flushPromptHistory()
  │
  ├─► Execute cleanup registry
  │    └─► registerCleanup() callbacks
  │
  ├─► Close MCP connections
  ├─► Kill subprocesses
  └─► Reset terminal cursor
```

### History Persistence

```
addToHistory(command)
  │
  ├─► Skip if CLAUDE_CODE_SKIP_PROMPT_HISTORY
  ├─► Register cleanup on first use
  ├─► Store pasted content (inline or hash reference)
  ├─► Push to pendingEntries
  └─► flushPromptHistory() (async)
       │
       ├─► Acquire lock on history.jsonl
       ├─► Write JSON lines
       └─► Release lock

removeLastFromHistory()
  └─► Undo most recent entry (for Esc interrupt rewind)
```

### Remote Session History

```
fetchLatestEvents(ctx, limit)
  │
  ├─► GET /v1/sessions/{sessionId}/events
  │    └─► ?limit=100&anchor_to_latest=true
  │
  └─► Returns HistoryPage { events, firstId, hasMore }

fetchOlderEvents(ctx, beforeId, limit)
  └─► GET /v1/sessions/{sessionId}/events
       └─► ?limit=100&before_id={beforeId}
```

---

## Session Interruption and Recovery

### Interrupt Handling

```
User presses Ctrl+C or Esc
  │
  ▼
abortController.abort()
  │
  ├─► In-flight API call cancelled
  ├─► Running tool interrupted
  │    └─► interruptBehavior() determines:
  │         ├─► 'cancel' → discard result
  │         └─► 'block' → keep running
  │
  ├─► Transcript preserved up to interrupt point
  └─► Session can be resumed with --resume <sessionId>
```

### Session Resume

```
--resume <sessionId> or --continue
  │
  ├─► Load transcript from disk
  ├─► processResumedConversation()
  │    ├─► Rebuild message chain
  │    ├─► Restore file state cache
  │    └─► Re-inject system context
  │
  └─► Continue from last assistant message
```

---

## Key Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `DISABLE_COMPACT` | Disable all compaction |
| `DISABLE_AUTO_COMPACT` | Disable only auto-compact |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Override auto-compact threshold (%) |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Override context window size |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | Override blocking limit |
| `CLAUDE_CODE_EAGER_FLUSH` | Force flush transcript after each message |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | Skip command history recording |

### Token Buffers

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Reserved for compact output |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Buffer before auto-compact triggers |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | Warning threshold buffer |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | Error threshold buffer |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Minimum buffer for manual compact |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker for failed compacts |

---

## Integration Points

| Component | Role in Session Lifecycle |
|-----------|--------------------------|
| `main.tsx` | Process entry, initialization, mode detection |
| `AppState` | Central session state store |
| `QueryEngine` | Per-conversation turn management |
| `compact.ts` | Conversation compaction |
| `autoCompact.ts` | Automatic compaction triggers |
| `history.ts` | Command history persistence |
| `sessionHistory.ts` | Remote session event fetching |
| `cleanupRegistry` | Graceful shutdown coordination |

## Related Documentation

- [Message Flow](./message-flow.md)
- [Permission Flow](./permission-flow.md)
- [Tool Execution Flow](./tool-execution-flow.md)
- [Compaction Service](../04-services/compact-service.md)
- [State Management](../01-core-modules/state-management.md)
