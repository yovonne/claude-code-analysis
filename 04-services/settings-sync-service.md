# Settings Sync Service

## Purpose
Syncs user settings and memory files (settings.json, CLAUDE.md) across Claude Code environments via a backend API. Supports bidirectional sync: upload from interactive CLI, download for CCR (Claude Code Remote) mode.

## Location
`restored-src/src/services/settingsSync/`

### Key Files
| File | Description |
|------|-------------|
| `index.ts` | Main service implementation — upload/download logic, API calls, file application |
| `types.ts` | Zod schemas and TypeScript types for the sync API contract |

## Key Exports

### Functions (index.ts)
- `uploadUserSettingsInBackground()`: Upload local settings to remote (interactive CLI only, fire-and-forget)
- `downloadUserSettings()`: Download settings from remote for CCR mode (cached promise, joinable)
- `redownloadUserSettings()`: Force fresh download, bypassing cached startup promise (for /reload-plugins)
- `_resetDownloadPromiseForTesting()`: Test-only helper to clear cached download promise

### Types (types.ts)
- `UserSyncData`: Full response from GET /api/claude_code/user_settings (userId, version, lastModified, checksum, content)
- `UserSyncContentSchema`: Flat key-value storage (entries: Record<string, string>)
- `SettingsSyncFetchResult`: Result from fetching user settings (success, data, isEmpty, error, skipRetry)
- `SettingsSyncUploadResult`: Result from uploading user settings (success, checksum, lastModified, error)

### Constants
- `SYNC_KEYS`: Key mappings for sync entries
  - `USER_SETTINGS`: '~/.claude/settings.json'
  - `USER_MEMORY`: '~/.claude/CLAUDE.md'
  - `projectSettings(projectId)`: 'projects/{projectId}/.claude/settings.local.json'
  - `projectMemory(projectId)`: 'projects/{projectId}/CLAUDE.local.md'
- `SETTINGS_SYNC_TIMEOUT_MS`: 10 seconds
- `DEFAULT_MAX_RETRIES`: 3
- `MAX_FILE_SIZE_BYTES`: 500 KB per file

## Dependencies

### Internal Dependencies
- `auth` — OAuth token management and refresh
- `git` — Repository remote hash for project identification
- `settings/settings` — Reading/writing local settings files
- `settings/settingsCache` — Cache invalidation after sync
- `claudemd` — Memory file cache management
- `api/withRetry` — Retry delay calculation
- `analytics` — Telemetry events for sync operations
- `growthbook` — Feature flag evaluation

### External Dependencies
- `axios` — HTTP client for API calls
- `lodash-es/pickBy` — Diffing local vs remote entries
- `zod/v4` — Schema validation
- `fs/promises` — File I/O

## Implementation Details

### Upload Flow (Interactive CLI)
1. Check eligibility: feature flag `UPLOAD_USER_SETTINGS`, growthbook `tengu_enable_settings_sync_push`, interactive mode, OAuth
2. Fetch current remote settings
3. Build entries from local files (user settings, user memory, project settings, project memory)
4. Diff local vs remote — only upload changed entries
5. PUT changed entries to API
6. Log telemetry events

### Download Flow (CCR Mode)
1. Check eligibility: feature flag `DOWNLOAD_USER_SETTINGS`, growthbook `tengu_strap_foyer`, OAuth
2. Fetch remote settings with retry (up to DEFAULT_MAX_RETRIES)
3. Apply remote entries to local files:
   - Write settings.json → resetSettingsCache()
   - Write CLAUDE.md → clearMemoryFileCaches()
4. Uses `markInternalWrite()` to prevent spurious change detection
5. Cached promise: first call starts fetch, subsequent calls join it

### File Size Limits
- Client-side: 500 KB per file (matches backend limit)
- Files exceeding limit or empty/whitespace-only are skipped

### Project Identification
- Uses git remote hash as project ID
- Project-specific files only synced when project ID is available

## Data Flow

### Upload
```
Interactive CLI startup
    ↓
uploadUserSettingsInBackground() (fire-and-forget)
    ↓
fetchUserSettings() — get current remote state
    ↓
buildEntriesFromLocalFiles() — read local files
    ↓
pickBy diff — find changed entries
    ↓
uploadUserSettings() — PUT changed entries
    ↓
Telemetry: tengu_settings_sync_upload_success/failed
```

### Download
```
CCR startup (print.ts runHeadless)
    ↓
downloadUserSettings() — cached promise, joinable
    ↓
fetchUserSettings() — with retries
    ↓
applyRemoteEntriesToLocal() — write files, invalidate caches
    ↓
markInternalWrite() — suppress change detection
    ↓
Telemetry: tengu_settings_sync_download_success/empty/error
```

## Integration Points
- **Main CLI**: `uploadUserSettingsInBackground()` called from main.tsx preAction
- **CCR Mode**: `downloadUserSettings()` fired at top of print.ts, awaited in plugin installation
- **Reload Plugins**: `redownloadUserSettings()` called by /reload-plugins for mid-session changes
- **Settings Cache**: `resetSettingsCache()` and `clearMemoryFileCaches()` after file writes
- **Change Detection**: `markInternalWrite()` prevents sync writes from triggering hot-reload loops

## Configuration
- Feature flags: `UPLOAD_USER_SETTINGS`, `DOWNLOAD_USER_SETTINGS`
- Growthbook flags: `tengu_enable_settings_sync_push`, `tengu_strap_foyer`
- OAuth requirement: first-party OAuth with `user:inference` scope
- Timeout: 10 seconds per request
- Retries: 3 with exponential backoff

## Error Handling
- **Fail-open**: Both upload and download log errors but don't block startup
- **Retry logic**: Transient errors (network, timeout) retry up to 3 times; auth errors skip retry
- **Cache sharing**: Download uses cached promise so multiple callers share one fetch
- **File write failures**: Individual file write failures logged but don't abort the entire sync

## Testing
- `_resetDownloadPromiseForTesting()` clears cached promise between tests
- Tests can mock feature flags and OAuth state

## Related Modules
- [Remote Managed Settings](./remote-managed-settings.md) — server-managed settings (different system)
- [Team Memory Sync](./team-memory-sync-service.md) — team memory file sync

## Notes
- Backend API contract: anthropic/anthropic#218817
- Upload is incremental — only changed entries are sent
- Download overwrites local files (server wins)
- Project-specific sync requires git remote hash for identification
- The download promise is cached so the fire-and-forget call and the awaited call share one fetch
