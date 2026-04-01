# Auto Dream Service

## Purpose
Background memory consolidation. Automatically fires the /dream prompt as a forked subagent when enough time has passed and enough sessions have accumulated. Consolidates recent session learnings into durable, well-organized memory files.

## Location
`restored-src/src/services/autoDream/`

### Key Files
| File | Description |
|------|-------------|
| `autoDream.ts` | Main orchestrator — gate checks, forked agent execution, task management |
| `config.ts` | Leaf config module — enabled state from settings + GrowthBook |
| `consolidationLock.ts` | Lock file management — mtime-based scheduling, PID ownership |
| `consolidationPrompt.ts` | Prompt template builder for the consolidation agent |

## Key Exports

### Functions (autoDream.ts)
- `initAutoDream()`: Initialize the auto-dream runner (call once at startup or per-test)
- `executeAutoDream(context, appendSystemMessage)`: Entry point from stopHooks — no-op until init

### Functions (config.ts)
- `isAutoDreamEnabled()`: Check if auto-dream is enabled (user setting overrides GrowthBook default)

### Functions (consolidationLock.ts)
- `readLastConsolidatedAt()`: Get mtime of lock file (= lastConsolidatedAt), 0 if absent
- `tryAcquireConsolidationLock()`: Acquire lock — write PID, return prior mtime or null if blocked
- `rollbackConsolidationLock(priorMtime)`: Rewind mtime after failed fork
- `recordConsolidation()`: Stamp lock from manual /dream (optimistic, best-effort)
- `listSessionsTouchedSince(sinceMs)`: Get session IDs with mtime after sinceMs

### Functions (consolidationPrompt.ts)
- `buildConsolidationPrompt(memoryRoot, transcriptDir, extra)`: Build the full consolidation prompt

### Types
- `AutoDreamConfig`: `{ minHours: number, minSessions: number }`

### Constants
- `DEFAULTS`: `{ minHours: 24, minSessions: 5 }`
- `SESSION_SCAN_INTERVAL_MS`: 10 minutes (scan throttle)

## Dependencies

### Internal Dependencies
- `forkedAgent` — runForkedAgent for subagent execution
- `DreamTask` — task registration, progress tracking, abort handling
- `extractMemories` — createAutoMemCanUseTool (shared tool permissions)
- `memdir/paths` — getAutoMemPath, isAutoMemoryEnabled
- `bootstrap/state` — getKairosActive, getIsRemoteMode, getSessionId, getOriginalCwd
- `analytics/growthbook` — tengu_onyx_plover feature flag
- `analytics` — telemetry events

### External Dependencies
- `fs/promises` — Lock file I/O
- `path` — Path manipulation

## Implementation Details

### Gate Order (Cheapest First)
1. **Feature gate**: Not KAIROS mode, not remote mode, auto-memory enabled, auto-dream enabled
2. **Time gate**: Hours since lastConsolidatedAt >= minHours (one stat call)
3. **Scan throttle**: At least 10 minutes since last session scan
4. **Session gate**: Transcript count with mtime > lastConsolidatedAt >= minSessions
5. **Lock**: No other process mid-consolidation (PID-based lock)

### Lock File Design
- Lock file: `.consolidate-lock` inside the memory directory
- Body: holder's PID
- mtime: lastConsolidatedAt (used for time gate)
- Stale threshold: 1 hour (HOLDER_STALE_MS)
- Reclaim: dead PID or unparseable body → reclaim
- Race handling: two reclaimers both write → last wins PID, loser bails on re-read
- Rollback: on fork failure, rewind mtime to pre-acquire value

### Forked Agent Execution
1. Register DreamTask with session count and abort controller
2. Build consolidation prompt with memory root, transcript dir, and tool constraints
3. Run forked agent with:
   - Read-only tool constraints (Bash restricted to ls/find/grep/cat/stat/wc/head/tail)
   - Memory directory write permissions only
   - Skip transcript (no recording)
   - Progress watcher for UI updates
4. On success: complete task, show inline completion summary
5. On failure: fail task, rollback lock, log event

### Progress Watcher
- Extracts text blocks and tool_use counts from assistant messages
- Collects file paths from Edit/Write tool calls
- Updates DreamTask state via addDreamTurn

### Configuration
- `minHours` and `minSessions` from GrowthBook flag `tengu_onyx_plover`
- User setting `autoDreamEnabled` overrides GrowthBook when explicitly set
- Defensive per-field validation (stale GB cache can return wrong types)

## Data Flow

```
Post-sampling hook
    ↓
executeAutoDream()
    ↓
Gate checks: feature → time → throttle → sessions → lock
    ↓
Register DreamTask
    ↓
buildConsolidationPrompt()
    ↓
runForkedAgent() — read-only tools, memory-dir writes only
    ↓
Progress watcher → addDreamTurn() → UI updates
    ↓
Success: completeDreamTask() + inline summary
Failure: failDreamTask() + rollbackConsolidationLock()
```

### Consolidation Prompt Phases
1. **Orient**: ls memory directory, read MEMORY.md, skim existing files
2. **Gather**: Scan daily logs, check for drifted memories, grep transcripts narrowly
3. **Consolidate**: Write/update memory files, merge signal, fix contradictions
4. **Prune and index**: Update MEMORY.md (under 200 lines, ~25KB), remove stale entries

## Integration Points
- **Post-sampling Hooks**: `executeAutoDream()` called from backgroundHousekeeping
- **Manual /dream**: Uses same lock mechanism via `recordConsolidation()`
- **DreamTask UI**: Progress visible in background tasks dialog
- **Extract Memories**: Shares `createAutoMemCanUseTool` for tool permissions

## Configuration
- GrowthBook flag: `tengu_onyx_plover` (enabled, minHours, minSessions)
- User setting: `autoDreamEnabled` in settings.json
- Defaults: 24 hours, 5 sessions
- Scan throttle: 10 minutes between session scans
- Lock stale threshold: 1 hour

## Error Handling
- **Gate failures**: Silent return — no logging for normal skip paths
- **Lock acquisition failure**: Silent return — another process is consolidating
- **Fork failure**: Rollback lock, fail task, log error event
- **User abort**: DreamTask.kill handles abort, lock rollback, no double-rollback
- **Forced mode**: Bypasses enabled/time/session gates (test override, not exposed)

## Testing
- `initAutoDream()` called in beforeEach for fresh closure-scoped state
- State is closure-scoped (not module-level) for test isolation
- `isForced()` is a build-time test override (always returns false in production)

## Related Modules
- [Extract Memories](./extract-memories-service.md) — memory extraction agent
- [Analytics](./analytics-service.md) — telemetry events

## Notes
- Extracted from dream.ts so auto-dream ships independently of KAIROS feature flags
- Lock file mtime IS lastConsolidatedAt — clever dual-purpose design
- Session scan uses mtime (sessions TOUCHED), not birthtime (0 on ext4)
- Forked agent shares parent's prompt cache for efficiency
- Manual /dream uses optimistic lock stamping (no post-skill completion hook)
