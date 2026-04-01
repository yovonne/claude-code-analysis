# Remote Managed Settings Service

## Purpose
Manages fetching, caching, and validation of remotely managed settings for enterprise customers. Uses checksum-based validation to minimize network traffic and provides graceful degradation on failures. Settings are pushed from the server and applied as a policy layer in the settings merge pipeline.

## Location
`restored-src/src/services/remoteManagedSettings/`

### Key Files
| File | Description |
|------|-------------|
| `index.ts` | Main service — fetch, cache, poll, security check integration |
| `syncCache.ts` | Eligibility check (auth-touching) and cache reset wrapper |
| `syncCacheState.ts` | Leaf state module — session cache, eligibility mirror, sync file loading |
| `securityCheck.tsx` | Blocking dialog for dangerous settings changes |
| `types.ts` | Zod schema and types for the remote settings API response |

## Key Exports

### Functions (index.ts)
- `loadRemoteManagedSettings()`: Load settings during CLI initialization, start background polling
- `refreshRemoteManagedSettings()`: Refresh settings asynchronously (for auth state changes)
- `isEligibleForRemoteManagedSettings()`: Public API for eligibility check
- `waitForRemoteManagedSettingsToLoad()`: Wait for initial loading to complete
- `clearRemoteManagedSettingsCache()`: Clear all caches (session, persistent, stop polling)
- `startBackgroundPolling()`: Start hourly background polling
- `stopBackgroundPolling()`: Stop background polling
- `initializeRemoteManagedSettingsLoadingPromise()`: Initialize loading promise early
- `computeChecksumFromSettings(settings)`: Compute SHA-256 checksum from settings (exported for testing)

### Functions (syncCache.ts)
- `isRemoteManagedSettingsEligible()`: Check eligibility (cached, auth-touching)
- `resetSyncCache()`: Clear eligibility cache and leaf state

### Functions (syncCacheState.ts)
- `getRemoteManagedSettingsSyncFromCache()`: Synchronous cache read (session → file)
- `setSessionCache(value)`: Update session cache
- `setEligibility(v)`: Mirror eligibility result in leaf module
- `resetSyncCache()`: Clear session cache and eligibility
- `getSettingsPath()`: Path to remote-settings.json cache file

### Functions (securityCheck.tsx)
- `checkManagedSettingsSecurity(cachedSettings, newSettings)`: Show blocking dialog if dangerous settings changed
- `handleSecurityCheckResult(result)`: Exit if rejected, continue otherwise

### Types (types.ts)
- `RemoteManagedSettingsResponse`: API response (uuid, checksum, settings)
- `RemoteManagedSettingsFetchResult`: Fetch result with success, settings (null = 304), checksum, error, skipRetry

### Constants
- `SETTINGS_TIMEOUT_MS`: 10 seconds
- `DEFAULT_MAX_RETRIES`: 5
- `POLLING_INTERVAL_MS`: 1 hour
- `LOADING_PROMISE_TIMEOUT_MS`: 30 seconds

## Dependencies

### Internal Dependencies
- `auth` — API key and OAuth token retrieval
- `model/providers` — API provider detection
- `settings/types` — Settings schema validation
- `settings/changeDetector` — Hot-reload notification
- `syncCacheState` — Leaf state module (avoids circular dependency)
- `securityCheck` — Dangerous settings dialog

### External Dependencies
- `axios` — HTTP client
- `crypto` — SHA-256 checksum computation
- `fs/promises` — Async file I/O
- `react` — Security dialog rendering
- `ink` — Terminal UI rendering

## Implementation Details

### Eligibility
- **Console users (API key)**: All eligible if actual key is available
- **OAuth users (Claude.ai)**: Enterprise/C4E and Team subscribers
- **Externally-injected tokens** (CCD, CCR, Agent SDK, CI): Eligible — API returns empty for ineligible orgs
- **3rd party providers**: Not eligible
- **Custom base URL users**: Not eligible
- **Cowork (local-agent entrypoint)**: Not eligible — server-managed settings don't apply

### Fetch Logic
1. Check eligibility — return null if not eligible
2. Load cached settings from file (syncCacheState)
3. Compute checksum from cached settings (sorted keys, SHA-256)
4. Fetch from API with If-None-Match header (ETag caching)
5. Handle responses:
   - 304: cached version still valid
   - 204/404: no settings exist, delete cache file
   - 200: validate with SettingsSchema, check security, save
6. On failure: use stale cache if available, otherwise fail open

### Security Check
- When new settings contain dangerous settings that differ from cached settings
- Shows blocking dialog in interactive mode
- User can accept (apply settings) or reject (keep cached, shutdown if first run)
- Skipped in non-interactive mode

### Background Polling
- Polls every hour via `setInterval` (unref'd to not block exit)
- Compares previous and new settings, triggers hot-reload if changed
- Calls `settingsChangeDetector.notifyChange('policySettings')`

### Cache Architecture (Circular Dependency Avoidance)
The module is split to break a circular dependency:
- `syncCacheState.ts`: Leaf module — no auth imports, only path/env/file/json
- `syncCache.ts`: Contains `isRemoteManagedSettingsEligible()` (the auth-touching part)
- `index.ts`: Main service, imports from both

Eligibility is a tri-state in the leaf: undefined (not determined), false (ineligible), true (proceed). Computed once and mirrored via `setEligibility()`.

### Checksum Computation
- Recursively sorts all keys to match Python's `json.dumps(sort_keys=True)`
- Uses compact separators (no spaces) to match Python's `separators=(",", ":")`
- Format: `sha256:<hex>`

## Data Flow

```
CLI Startup
    ↓
initializeRemoteManagedSettingsLoadingPromise()
    ↓
loadRemoteManagedSettings()
    ↓
getRemoteManagedSettingsSyncFromCache() — cache-first fast path
    ↓
fetchAndLoadRemoteManagedSettings()
    ↓
fetchWithRetry() — API call with ETag, up to 5 retries
    ↓
checkManagedSettingsSecurity() — dialog if dangerous changes
    ↓
saveSettings() → setSessionCache() → notifyChange('policySettings')
    ↓
startBackgroundPolling() — hourly refresh
```

## Integration Points
- **Settings Pipeline**: Cached settings merged as `policySettings` layer
- **CLI Initialization**: `loadRemoteManagedSettings()` called during startup
- **Auth Changes**: `refreshRemoteManagedSettings()` called on login/logout
- **Hot Reload**: `settingsChangeDetector.notifyChange('policySettings')` triggers re-merge
- **Security Dialog**: Blocking UI for dangerous settings changes

## Configuration
- API endpoint: `/api/claude_code/settings`
- Cache file: `~/.claude/remote-settings.json` (mode 0o600)
- Polling interval: 1 hour
- Fetch timeout: 10 seconds
- Max retries: 5 with exponential backoff

## Error Handling
- **Fail-open**: API failures continue without remote settings
- **Stale cache**: On fetch failure, uses cached settings if available
- **Auth errors**: Skip retry (401/403 won't succeed)
- **Empty settings**: 204/404 deletes cache file to prevent stale settings
- **Validation errors**: Invalid settings structure rejected, treated as fetch failure
- **Security rejection**: User can reject dangerous settings, keeping cached version

## Testing
- `_reset*ForTesting` helpers not exposed — tests use `clearRemoteManagedSettingsCache()`
- `computeChecksumFromSettings()` exported for testing server compatibility

## Related Modules
- [Policy Limits](./policy-limits-service.md) — similar patterns (ETag, polling, fail-open)
- [Settings Sync](./settings-sync-service.md) — user settings sync (different system)

## Notes
- Settings are validated with full `SettingsSchema` after initial permissive parse
- The loading promise has a 30-second timeout to prevent deadlocks
- Cache-first startup: cached settings applied immediately, fetch runs in background
- gh-23085: Settings cache reset on first eligibility determination prevents poisoned merged cache
- Cowork (local-agent) is explicitly excluded — MDM/file-based managed settings still apply
