# Team Memory Sync Service

## Purpose
Syncs team memory files between the local filesystem and a server API. Team memory is scoped per-repo (identified by git remote) and shared across all authenticated organization members. Supports pull (server wins), push (local wins on conflict), and bidirectional sync with optimistic locking.

## Location
`restored-src/src/services/teamMemorySync/`

### Key Files
| File | Description |
|------|-------------|
| `index.ts` | Main sync service — pull, push, batch, conflict resolution, local file ops |
| `types.ts` | Zod schemas and types for the team memory sync API |
| `watcher.ts` | File system watcher with debounced push and suppression logic |
| `secretScanner.ts` | Client-side secret scanner using gitleaks regex patterns |
| `teamMemSecretGuard.ts` | Pre-write guard for FileWriteTool/FileEditTool |

## Key Exports

### Functions (index.ts)
- `pullTeamMemory(state, options)`: Pull from server, write to local directory, returns files written
- `pushTeamMemory(state)`: Push local files to server with delta upload and conflict resolution
- `syncTeamMemory(state)`: Bidirectional sync — pull then push
- `isTeamMemorySyncAvailable()`: Check if OAuth is available for sync
- `createSyncState()`: Create fresh SyncState object for isolation
- `hashContent(content)`: Compute sha256 checksum for content comparison
- `batchDeltaByBytes(delta)`: Split delta into PUT-sized batches under gateway body limit

### Functions (watcher.ts)
- `startTeamMemoryWatcher()`: Start watching team memory directory, initial pull, then fs.watch
- `stopTeamMemoryWatcher()`: Stop watcher, flush pending changes
- `notifyTeamMemoryWrite()`: Explicit push trigger from PostToolUse hooks
- `isPermanentFailure(result)`: Determine if push failure should suppress retries
- `_resetWatcherStateForTesting(opts)`: Test-only state reset
- `_startFileWatcherForTesting(dir)`: Test-only real fs.watch starter

### Functions (secretScanner.ts)
- `scanForSecrets(content)`: Scan content for credentials, return matches (ruleId, label)
- `redactSecrets(content)`: Replace matched secrets with [REDACTED] in-place
- `getSecretLabel(ruleId)`: Convert gitleaks rule ID to human-readable label

### Functions (teamMemSecretGuard.ts)
- `checkTeamMemSecrets(filePath, content)`: Pre-write guard — returns error message if secrets detected

### Types (types.ts)
- `TeamMemoryData`: Full API response (organizationId, repo, version, lastModified, checksum, content)
- `TeamMemoryContentSchema`: entries + optional entryChecksums (per-key SHA-256)
- `TeamMemorySyncFetchResult`: Pull result with notModified, isEmpty, checksum support
- `TeamMemorySyncPushResult`: Push result with filesUploaded, conflict, skippedSecrets
- `TeamMemoryHashesResult`: Lightweight metadata-only probe result (?view=hashes)
- `SkippedSecretFile`: File skipped due to detected secret (path, ruleId, label)
- `TeamMemoryTooManyEntriesSchema`: Structured 413 error with max_entries, received_entries

### Types (index.ts)
- `SyncState`: Mutable state — lastKnownChecksum, serverChecksums (Map), serverMaxEntries

### Constants
- `TEAM_MEMORY_SYNC_TIMEOUT_MS`: 30 seconds
- `MAX_FILE_SIZE_BYTES`: 250 KB per entry
- `MAX_PUT_BODY_BYTES`: 200 KB (gateway body size limit)
- `MAX_RETRIES`: 3 for fetch
- `MAX_CONFLICT_RETRIES`: 2 for 412 conflict resolution
- `DEBOUNCE_MS`: 2000ms for watcher push debounce

## Dependencies

### Internal Dependencies
- `auth` — OAuth token management
- `git` — GitHub repo slug identification
- `teamMemPaths` — Team memory directory path validation
- `analytics` — Telemetry events for push/pull operations
- `api/withRetry` — Retry delay calculation

### External Dependencies
- `axios` — HTTP client
- `crypto` — SHA-256 hashing
- `fs/promises` — File I/O
- `path` — Path manipulation
- `zod/v4` — Schema validation

## Implementation Details

### Pull (Server Wins)
1. Fetch from API with ETag caching (If-None-Match → 304 Not Modified)
2. On 404: server has no data, clear stale serverChecksums
3. On 200: write remote entries to local directory
4. Skip entries whose on-disk content already matches (preserves mtime)
5. Validate paths against directory boundary (prevent traversal)
6. Parallel file writes for performance

### Push (Local Wins, Delta Upload)
1. Read local files, scan for secrets (skip files with detected secrets)
2. Hash each local entry once
3. Compute delta: only keys whose local hash differs from serverChecksums
4. Split delta into batches under MAX_PUT_BODY_BYTES
5. PUT batches with If-Match header (optimistic locking)
6. On 412 conflict: probe GET ?view=hashes, refresh serverChecksums, recompute delta, retry
7. Update serverChecksums after each successful batch

### Conflict Resolution
- Uses ETag-based optimistic locking (If-Match header)
- On 412: cheap probe (?view=hashes) to refresh serverChecksums without downloading bodies
- Recomputed delta naturally excludes server-origin content from concurrent pushes
- Local-wins-on-conflict: user's active edit is not silently discarded
- Max 2 conflict retries before failing

### File Watcher
- Uses `fs.watch({recursive: true})` — O(1) fds on macOS (FSEvents), O(subdirs) on Linux (inotify)
- Debounced push: 2s after last change
- Push suppression on permanent failures (4xx except 409/429, no_oauth, no_repo)
- Suppression cleared on file unlink (recovery action for too-many-entries)
- Initial pull runs before watcher starts (disk writes won't trigger schedulePush)

### Secret Scanning (PSR M22174)
- Curated subset of gitleaks rules with near-zero false-positive rates
- Covers: AWS, GCP, Azure, GitHub, GitLab, Slack, Stripe, OpenAI, Anthropic, and more
- Anthropic API key prefix assembled at runtime to avoid literal in bundle
- Files with secrets are SKIPPED (not uploaded), never logged with secret values
- Pre-write guard in FileWriteTool/FileEditTool prevents secrets from being written

### Batch Splitting
- Greedy bin-packing over sorted keys (deterministic batches)
- Single entry exceeding limit goes into its own solo batch
- Each batch is an independent PUT — if batch N fails, 1..N-1 are already committed

## Data Flow

```
File Edit → teamMemSecretGuard.ts (pre-write check)
                ↓
          scanForSecrets() — block if secrets found
                ↓
          Write to disk → fs.watch event → schedulePush()
                ↓
          Debounce (2s) → executePush()
                ↓
          readLocalTeamMemory() → scanForSecrets() → hashContent()
                ↓
          Compute delta → batchDeltaByBytes()
                ↓
          PUT batches with If-Match → 412? → probe ?view=hashes → retry
```

## Integration Points
- **PostToolUse Hooks**: `notifyTeamMemoryWrite()` called after memory file writes
- **FileWriteTool/FileEditTool**: `checkTeamMemSecrets()` validates content before write
- **Startup**: `startTeamMemoryWatcher()` initializes sync system
- **Shutdown**: `stopTeamMemoryWatcher()` flushes pending changes (best-effort, 2s budget)

## Configuration
- Feature flag: `TEAMMEM` build flag
- Eligibility: team memory enabled, OAuth available, GitHub remote present
- Server endpoint: `/api/claude_code/team_memory?repo={owner/repo}`
- Per-entry size cap: 250 KB
- Gateway body cap: ~256-512 KB (client stays under 200 KB)
- Server max entries: learned from structured 413 response (GB-tunable per-org)

## Error Handling
- **Permanent failures**: 4xx (except 409/429), no_oauth, no_repo — suppress retries until unlink
- **Transient failures**: 409 (conflict), 429 (rate limit), 5xx — retry with backoff
- **Conflict resolution**: 2 retries with cheap hash probe
- **Oversized files**: Skipped client-side (250 KB limit)
- **Secret detection**: Files skipped, user warned with rule labels (never secret values)
- **Fail-open**: Watcher errors logged but don't crash the session

## Testing
- `_resetWatcherStateForTesting()` resets module state, optionally seeds syncState
- `_startFileWatcherForTesting()` starts real fs.watch for fd-count regression tests
- Tests create fresh SyncState per test for isolation

## Related Modules
- [Settings Sync](./settings-sync-service.md) — user settings sync (different system)
- [Extract Memories](./extract-memories-service.md) — memory extraction agent

## Notes
- API contract: anthropic/anthropic#250711 + #283027 (entryChecksums) + #293258 (structured 413)
- File deletions do NOT propagate — deleting locally won't remove from server, next pull restores
- Watcher uses fs.watch directly (not chokidar) to avoid fd exhaustion with 500+ files
- Push is read-only on disk (delta+probe, no merge writes) — no event suppression needed
- Secret scanner uses lazy-compiled regex cache — compiles once on first scan
