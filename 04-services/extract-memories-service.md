# Extract Memories Service

## Purpose
Extracts durable memories from the current session transcript and writes them to the auto-memory directory. Runs at the end of each complete query loop (when the model produces a final response with no tool calls) via stop hooks. Uses the forked agent pattern to share the parent's prompt cache.

## Location
`restored-src/src/services/extractMemories/`

### Key Files
| File | Description |
|------|-------------|
| `extractMemories.ts` | Main service — extraction orchestration, tool permissions, closure state |
| `prompts.ts` | Prompt templates for the extraction agent (auto-only and combined variants) |

## Key Exports

### Functions (extractMemories.ts)
- `initExtractMemories()`: Initialize the extraction system with fresh closure state (call once at startup)
- `executeExtractMemories(context, appendSystemMessage)`: Run extraction at end of query loop
- `drainPendingExtraction(timeoutMs)`: Await all in-flight extractions with soft timeout
- `createAutoMemCanUseTool(memoryDir)`: Create canUseTool function for forked agent permissions

### Functions (prompts.ts)
- `buildExtractAutoOnlyPrompt(newMessageCount, existingMemories, skipIndex)`: Prompt for auto-only memory
- `buildExtractCombinedPrompt(newMessageCount, existingMemories, skipIndex)`: Prompt for auto + team memory

### Types
- `AppendSystemMessageFn`: Function type for appending system messages

## Dependencies

### Internal Dependencies
- `forkedAgent` — runForkedAgent for subagent execution
- `memdir/memoryScan` — scanMemoryFiles, formatMemoryManifest
- `memdir/paths` — getAutoMemPath, isAutoMemoryEnabled, isAutoMemPath
- `memdir/teamMemPaths` — Team memory paths (conditional, feature-gated)
- `memdir/memoryTypes` — Memory type definitions and examples
- `messages` — createUserMessage, createMemorySavedMessage
- `analytics/growthbook` — Feature flags (tengu_passport_quail, tengu_moth_copse, tengu_bramble_lintel)
- `analytics` — Telemetry events

### External Dependencies
- `bun:bundle` — feature() build flag
- `path` — Path manipulation

## Implementation Details

### Extraction Flow
1. Check eligibility: main agent only, feature gate, auto-memory enabled, not remote mode
2. Count new model-visible messages since last extraction cursor
3. Check mutual exclusion: skip if main agent already wrote to memory files
4. Scan existing memory files for manifest (pre-inject, saves agent turn)
5. Build prompt (auto-only or combined variant based on team memory status)
6. Run forked agent with restricted tools (max 5 turns)
7. Advance cursor, extract written file paths, log telemetry
8. Show inline "Saved N memories" message if files were written

### Closure-Scoped State
All mutable state lives inside `initExtractMemories()` closure:
- `inFlightExtractions`: Set of pending extraction promises
- `lastMemoryMessageUuid`: Cursor UUID for incremental processing
- `hasLoggedGateFailure`: One-shot flag to avoid repeated log spam
- `inProgress`: Prevents overlapping extraction runs
- `turnsSinceLastExtraction`: Throttle counter (configurable via tengu_bramble_lintel)
- `pendingContext`: Stashed context for trailing run after in-progress extraction

### Mutual Exclusion
When the main agent writes memories itself, the forked extraction is skipped and the cursor advances past the write. This makes the main agent and background agent mutually exclusive per turn, avoiding redundant work.

### Tool Permissions (createAutoMemCanUseTool)
- **Allowed**: FileRead, Grep, Glob (unrestricted, read-only)
- **Allowed**: Bash (read-only commands only — ls/find/grep/cat/stat/wc/head/tail)
- **Allowed**: FileEdit/FileWrite (only for paths within memory directory)
- **Allowed**: REPL (passes through to inner tool checks)
- **Denied**: Everything else (MCP, Agent, write-capable Bash, rm)

### Coalescing and Trailing Runs
When an extraction is already in progress:
- Incoming calls stash their context (latest overwrites previous)
- After current extraction completes, a trailing run processes the stashed context
- Trailing runs skip the throttle check (already-committed work)

### Drain Mechanism
- `drainPendingExtraction()` awaits all in-flight extractions with configurable timeout (default 60s)
- Called by print.ts after response is flushed but before gracefulShutdownSync
- Ensures forked agent completes before 5s shutdown failsafe

### Prompt Variants
- **Auto-only**: Four-type taxonomy, no scope guidance (single directory)
- **Combined**: Four-type taxonomy with per-type scope guidance (private vs team directories)
- **skipIndex**: When true, omits Step 2 (MEMORY.md index update) from instructions

## Data Flow

```
Query loop completes (no tool calls)
    ↓
handleStopHooks → executeExtractMemories()
    ↓
Check gates: main agent? feature? enabled? not remote?
    ↓
Count new messages since cursor
    ↓
Check hasMemoryWritesSince() — skip if main agent wrote
    ↓
Throttle check: turnsSinceLastExtraction >= threshold
    ↓
scanMemoryFiles() → formatMemoryManifest()
    ↓
buildExtractPrompt() — auto-only or combined
    ↓
runForkedAgent() — max 5 turns, restricted tools
    ↓
Advance cursor, extract written paths
    ↓
createMemorySavedMessage() → appendSystemMessage()
    ↓
Telemetry: tengu_extract_memories_extraction
```

## Integration Points
- **Stop Hooks**: Called fire-and-forget from handleStopHooks
- **Print Mode**: `drainPendingExtraction()` called before gracefulShutdownSync
- **Auto Dream**: Shares `createAutoMemCanUseTool` for tool permissions
- **Team Memory**: Conditional support via feature('TEAMMEM') flag

## Configuration
- Feature flags:
  - `tengu_passport_quail`: Master gate for extraction
  - `tengu_moth_copse`: Skip MEMORY.md index step
  - `tengu_bramble_lintel`: Turn throttle (default 1)
- Build flag: `TEAMMEM` for team memory support
- Max turns: 5 (hard cap to prevent verification rabbit-holes)
- Drain timeout: 60 seconds (default)

## Error Handling
- **Best-effort**: Extraction errors logged but don't notify user
- **Cursor preservation**: On error, cursor stays put so messages are reconsidered
- **Coalesced calls**: Stashed context overwritten by latest (only most recent matters)
- **Gate failures**: Logged once per session (hasLoggedGateFailure flag)

## Testing
- `initExtractMemories()` called in beforeEach for fresh closure state
- All mutable state is closure-scoped for test isolation
- Tests can verify tool denial, message counting, and cursor advancement

## Related Modules
- [Auto Dream](./auto-dream-service.md) — background memory consolidation
- [Team Memory Sync](./team-memory-sync-service.md) — team memory file sync

## Notes
- The main agent's system prompt always has full save instructions — extraction only runs when main agent didn't write
- Prompt cache sharing is preserved by giving the fork the same tool list (tools are part of cache key)
- Index file updates (MEMORY.md) are filtered from the "memories saved" count — user-visible memory is the topic file
- Team memory count tracked separately in telemetry when TEAMMEM is enabled
