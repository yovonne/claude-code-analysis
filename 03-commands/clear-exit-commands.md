# Clear and Exit Commands

## Purpose

Documents the CLI commands that terminate or reset session state: clearing conversation history while keeping the session alive, and exiting the REPL entirely.

---

## /clear (Conversation Reset)

### Purpose
Clears the current conversation history and resets session state while keeping the REPL running. Provides a fresh context without terminating the session.

### Location
`restored-src/src/commands/clear/index.ts`
`restored-src/src/commands/clear/clear.ts`
`restored-src/src/commands/clear/conversation.ts`
`restored-src/src/commands/clear/caches.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `clear` |
| Aliases | `reset`, `new` |
| Type | `local` |
| Description | Clear conversation history and free up context |
| Supports Non-Interactive | `false` (creates a new session instead) |

### Key Exports

#### From `index.ts`
- `clear` — Command metadata with lazy-load configuration

#### From `clear.ts`
- `call`: `LocalCommandCall` — Entry point that delegates to `clearConversation`

#### From `conversation.ts`
- `clearConversation(options)`: Main clearing logic — resets messages, caches, session ID, and triggers lifecycle hooks

#### From `caches.ts`
- `clearSessionCaches(preservedAgentIds)`: Comprehensive session cache clearing without affecting messages or session ID

### Dependencies

#### Internal
- `../../bootstrap/state.js` — `getLastMainRequestId`, `getOriginalCwd`, `getSessionId`, `regenerateSessionId`, `clearInvokedSkills`, `setLastEmittedDate`
- `../../services/analytics/index.js` — `logEvent` for cache eviction hints
- `../../utils/sessionStorage.js` — `clearSessionMetadata`, `getAgentTranscriptPath`, `resetSessionFilePointer`, `saveWorktreeState`
- `../../utils/hooks.js` — `executeSessionEndHooks`, `getSessionEndHookTimeoutMs`
- `../../utils/sessionStart.js` — `processSessionStartHooks`
- `../../utils/worktree.js` — `getCurrentWorktreeSession`
- `../../utils/Shell.js` — `setCwd`
- `../../utils/plans.js` — `clearAllPlanSlugs`
- `../../utils/commitAttribution.js` — `createEmptyAttributionState`
- `../../utils/task/diskOutput.js` — `evictTaskOutput`, `initTaskOutputAsSymlink`
- `../../tasks/LocalAgentTask/LocalAgentTask.js` — `isLocalAgentTask`
- `../../tasks/InProcessTeammateTask/types.js` — `isInProcessTeammateTask`
- `../../tasks/LocalShellTask/guards.js` — `isLocalShellTask`
- `../../commands.js` — `clearCommandsCache`
- `../../context.js` — `getGitStatus`, `getSystemContext`, `getUserContext`, `setSystemPromptInjection`
- `../../hooks/fileSuggestions.js` — `clearFileSuggestionCaches`
- `../../hooks/useSwarmPermissionPoller.js` — `clearAllPendingCallbacks`
- `../../services/api/dumpPrompts.js` — `clearAllDumpState`
- `../../services/api/promptCacheBreakDetection.js` — `resetPromptCacheBreakDetection`
- `../../services/api/sessionIngress.js` — `clearAllSessions`
- `../../services/compact/postCompactCleanup.js` — `runPostCompactCleanup`
- `../../services/lsp/LSPDiagnosticRegistry.js` — `resetAllLSPDiagnosticState`
- `../../services/MagicDocs/magicDocs.js` — `clearTrackedMagicDocs`
- `../../skills/loadSkillsDir.js` — `clearDynamicSkills`
- `../../utils/attachments.js` — `resetSentSkillNames`
- `../../utils/bash/commands.js` — `clearCommandPrefixCaches`
- `../../utils/claudemd.js` — `resetGetMemoryFilesCache`
- `../../utils/detectRepository.js` — `clearRepositoryCaches`
- `../../utils/git/gitFilesystem.js` — `clearResolveGitDirCache`
- `../../utils/imageStore.js` — `clearStoredImagePaths`
- `../../utils/sessionEnvVars.js` — `clearSessionEnvVars`

#### External
- `bun:bundle` — `feature` flag system
- `crypto` — `randomUUID`, `UUID` type

### Implementation Details

#### Clearing Sequence
The `clearConversation` function executes these steps in order:

**1. Session End Hooks**
- Executes `SessionEnd` hooks with type `'clear'`
- Bounded by `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` (default 1.5s)
- Uses `AbortSignal.timeout` for enforcement

**2. Cache Eviction Hint**
- Logs `tengu_cache_eviction_hint` analytics event with the last request ID
- Signals to inference that this conversation's cache can be evicted

**3. Task Preservation**
- Identifies tasks to preserve: backgrounded tasks and main-session tasks
- Tasks with `isBackgrounded === false` are killed
- Preserved tasks retain their per-agent state (invoked skills, pending callbacks, dump state)

**4. Message Reset**
- `setMessages(() => [])` — Clears all messages

**5. Context Unblock**
- Clears the context-blocked flag for PROACTIVE/KAIROS features
- Allows proactive ticks to resume after clear

**6. Conversation ID Rotation**
- Generates a new UUID via `randomUUID()`
- Forces logo re-render in the UI

**7. Cache Clearing**
- Calls `clearSessionCaches(preservedAgentIds)` — see cache clearing section below
- Resets working directory to original cwd
- Clears file state cache, discovered skills, and loaded memory paths

**8. App State Cleanup**
- Partitions tasks: kills foreground tasks, preserves background tasks
- Kills running shell tasks (process kill + cleanup)
- Aborts running tasks via abort controllers
- Resets attribution state to empty
- Clears `standaloneAgentContext` (name/color from `/rename`, `/color`)
- Clears file history (snapshots, tracked files, sequence)
- Resets MCP state (clients, tools, commands, resources) while preserving `pluginReconnectKey`

**9. Metadata and Session ID Reset**
- Clears plan slug cache
- Clears cached session metadata (title, tag, agent name/color)
- Regenerates session ID with `setCurrentAsParent: true` for analytics lineage
- Updates `CLAUDE_CODE_SESSION_ID` environment variable (for `USER_TYPE === 'ant'`)
- Resets session file pointer

**10. Task Output Symlink Repair**
- Re-points symlinks for preserved running local agent tasks
- Ensures TaskOutput reads from the new session directory, not the frozen pre-clear snapshot
- Main-session tasks use shared per-agent paths, so no special handling needed

**11. State Re-Persistence**
- Saves coordinator mode state if enabled
- Saves worktree session state if applicable

**12. Session Start Hooks**
- Executes `SessionStart` hooks with type `'clear'`
- Updates messages with hook results

#### Cache Clearing (`clearSessionCaches`)
This function is a comprehensive cache reset that clears:

| Category | Caches Cleared |
|----------|---------------|
| Context | User context, system context, git status, session start date |
| Files | File suggestion caches |
| Commands | Command registry, dynamic skills |
| Prompts | Prompt cache break detection, system prompt injection, dump state, memory files cache |
| Images | Stored image paths |
| Session | All session ingress, session environment variables, Tungsten usage tracking |
| Swarm | Pending permission callbacks |
| Attribution | File content cache, pending bash states (feature-flagged) |
| Repository | Repository detection caches, git dir resolution cache |
| Bash | Command prefix caches (Haiku-extracted) |
| Skills | Invoked skills cache, dynamic skills, sent skill names |
| LSP | Diagnostic tracking state |
| MagicDocs | Tracked magic docs |
| WebFetch | URL cache (up to 50MB) |
| ToolSearch | Description cache (~500KB for 50 MCP tools) |
| AgentTool | Agent definitions cache |
| SkillTool | Prompt cache |

The `preservedAgentIds` parameter allows selective preservation of per-agent state for background tasks. When non-empty:
- AgentId-keyed state (invoked skills) is selectively cleared
- RequestId-keyed state (pending callbacks, dump state, cache-break tracking) is left intact

#### Edge Cases
- Non-interactive mode: `supportsNonInteractive` is `false`, so `/clear` creates a new session instead
- Running foreground tasks are killed (shell processes terminated, abort controllers triggered)
- Background tasks survive the clear and continue functioning
- Post-compaction cleanup is intentionally run even though messages are wiped (resets skill listing, memory files)
- The `pluginReconnectKey` is preserved so `/clear` doesn't cause a no-op MCP reconnection

### Data Flow
```
User runs /clear
  → executeSessionEndHooks('clear')
  → logEvent('tengu_cache_eviction_hint')
  → Identify preserved tasks (backgrounded + main-session)
  → setMessages([])
  → Clear context-blocked flag
  → setConversationId(randomUUID())
  → clearSessionCaches(preservedAgentIds)
  → setCwd(originalCwd)
  → Clear file state, skills, memory paths
  → Kill foreground tasks, preserve background tasks
  → Reset attribution, standaloneAgentContext, fileHistory, MCP
  → clearAllPlanSlugs()
  → clearSessionMetadata()
  → regenerateSessionId(setCurrentAsParent: true)
  → resetSessionFilePointer()
  → Re-point task output symlinks
  → Save mode/worktree state
  → processSessionStartHooks('clear')
  → Update messages with hook results
```

### Integration Points
- **Lifecycle hooks**: Both SessionEnd and SessionStart hooks fire around the clear
- **Task system**: Background tasks survive; foreground tasks are terminated
- **Analytics**: Cache eviction hints and session lineage tracking
- **Session storage**: Metadata, file pointers, and worktree state are reset
- **MCP system**: Clients are reset but reconnect key is preserved

---

## /exit (REPL Termination)

### Purpose
Exits the REPL session, performing graceful shutdown of the application.

### Location
`restored-src/src/commands/exit/index.ts`
`restored-src/src/commands/exit/exit.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `exit` |
| Aliases | `quit` |
| Type | `local-jsx` |
| Description | Exit the REPL |
| Immediate | `true` |

### Key Exports

#### Functions
- `call(onDone)`: Main entry point for the exit command

#### Constants
- `GOODBYE_MESSAGES`: Array of random goodbye messages — `['Goodbye!', 'See ya!', 'Bye!', 'Catch you later!']`

### Dependencies

#### Internal
- `../../components/ExitFlow.js` — `ExitFlow` component for worktree exit confirmation
- `../../utils/concurrentSessions.js` — `isBgSession` for background session detection
- `../../utils/gracefulShutdown.js` — `gracefulShutdown` for clean application termination
- `../../utils/worktree.js` — `getCurrentWorktreeSession` for worktree state detection

#### External
- `bun:bundle` — `feature` flag system
- `child_process` — `spawnSync` for tmux detach
- `lodash-es/sample.js` — Random selection from goodbye messages
- `react` — Component rendering

### Implementation Details

#### Core Logic — Three Exit Paths

**Path 1: Background Session (tmux detach)**
When running inside a `claude --bg` tmux session:
1. Calls `onDone()` with no message
2. Executes `tmux detach-client` via `spawnSync` (stdio ignored)
3. Returns `null` — the REPL keeps running; `claude attach` can reconnect
4. This path covers `/exit`, `/quit`, `Ctrl+C`, `Ctrl+D` — all funnel through `handleExit`

**Path 2: Worktree Session (confirmation flow)**
When in a worktree session (`getCurrentWorktreeSession() !== null`):
1. Renders the `ExitFlow` component with `showWorktree={true}`
2. `ExitFlow` provides a confirmation UI for exiting the worktree
3. On cancel: calls `onDone()` with no message
4. On confirm: proceeds to graceful shutdown

**Path 3: Normal Exit (graceful shutdown)**
For standard sessions:
1. Selects a random goodbye message from `GOODBYE_MESSAGES`
2. Calls `onDone(goodbyeMessage)` to display it
3. Awaits `gracefulShutdown(0, 'prompt_input_exit')` for clean termination
4. Returns `null`

#### Graceful Shutdown
The `gracefulShutdown` function (imported from `../../utils/gracefulShutdown.js`) handles:
- Session persistence and final state saving
- Resource cleanup
- Process termination

#### Edge Cases
- Background sessions detach instead of killing, allowing reconnection
- Worktree sessions require confirmation to prevent accidental data loss
- All exit triggers (`/exit`, `/quit`, `Ctrl+C`, `Ctrl+D`) route through the same handler
- The `immediate: true` flag means the command executes without waiting for pending operations

### Data Flow
```
User triggers exit (/exit, /quit, Ctrl+C, Ctrl+D)
  │
  ├─ BG session? → onDone() → tmux detach-client → REPL keeps running
  │
  ├─ Worktree? → Show ExitFlow → User confirms → gracefulShutdown
  │                        → User cancels → onDone() (no-op)
  │
  └─ Normal → Random goodbye → onDone(message) → gracefulShutdown(0, 'prompt_input_exit')
```

### Integration Points
- **Background sessions**: Integrates with tmux for detach/attach lifecycle
- **Worktree mode**: Uses ExitFlow component for safe worktree exit confirmation
- **Graceful shutdown**: Central shutdown utility handles all cleanup
- **REPL handler**: All exit signals funnel through REPL's `handleExit`

---

## Clear vs Exit Comparison

| Aspect | `/clear` | `/exit` |
|--------|----------|---------|
| **Session** | Continues with new session ID | Terminates entirely |
| **Messages** | Wiped completely | Persisted to log storage |
| **Tasks** | Background preserved, foreground killed | All terminated via shutdown |
| **Hooks** | SessionEnd + SessionStart fire | SessionEnd fires during shutdown |
| **REPL** | Stays running | Exits |
| **Aliases** | `reset`, `new` | `quit` |
| **Type** | `local` | `local-jsx` |
| **Immediate** | No | Yes |

---

## Related Modules
- [session-commands.md](./session-commands.md) — Session lifecycle management commands
- [session-lifecycle.md](../09-data-flow/session-lifecycle.md) — Session lifecycle data flow
- [gracefulShutdown](../../restored-src/src/utils/gracefulShutdown.js) — Application shutdown utility
- [sessionStorage](../../restored-src/src/utils/sessionStorage.js) — Session storage utilities
