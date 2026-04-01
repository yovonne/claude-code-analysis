# Policy Limits Service

## Purpose
Fetches organization-level policy restrictions from the API and uses them to disable CLI features. Follows the same patterns as remote managed settings: fail-open, ETag caching, background polling, and retry logic.

## Location
`restored-src/src/services/policyLimits/`

### Key Files
| File | Description |
|------|-------------|
| `index.ts` | Main service — fetch, cache, poll, and policy evaluation |
| `types.ts` | Zod schema and types for the policy limits API response |

## Key Exports

### Functions (index.ts)
- `loadPolicyLimits()`: Load policy limits during CLI initialization, start background polling
- `refreshPolicyLimits()`: Refresh policy limits asynchronously (for auth state changes)
- `isPolicyAllowed(policy)`: Check if a specific policy is allowed (synchronous)
- `isPolicyLimitsEligible()`: Check if current user is eligible for policy limits
- `waitForPolicyLimitsToLoad()`: Wait for initial loading to complete
- `clearPolicyLimitsCache()`: Clear all caches (session, persistent, stop polling)
- `startBackgroundPolling()`: Start hourly background polling
- `stopBackgroundPolling()`: Stop background polling
- `initializePolicyLimitsLoadingPromise()`: Initialize loading promise early (called from init.ts)
- `_resetPolicyLimitsForTesting()`: Test-only sync reset of module state

### Types (types.ts)
- `PolicyLimitsResponse`: API response with `restrictions` record (policy key → { allowed: boolean })
- `PolicyLimitsFetchResult`: Fetch result with success, restrictions (null = 304 cache valid), etag, error, skipRetry

### Constants
- `CACHE_FILENAME`: 'policy-limits.json'
- `FETCH_TIMEOUT_MS`: 10 seconds
- `DEFAULT_MAX_RETRIES`: 5
- `POLLING_INTERVAL_MS`: 1 hour
- `LOADING_PROMISE_TIMEOUT_MS`: 30 seconds
- `ESSENTIAL_TRAFFIC_DENY_ON_MISS`: Set containing policies that fail closed in HIPAA mode (currently `allow_product_feedback`)

## Dependencies

### Internal Dependencies
- `auth` — API key and OAuth token retrieval
- `model/providers` — API provider detection
- `privacyLevel` — Essential-traffic-only mode check
- `api/withRetry` — Retry delay calculation
- `cleanupRegistry` — Shutdown cleanup registration

### External Dependencies
- `axios` — HTTP client
- `crypto` — SHA-256 checksum computation
- `fs/promises` — Async file I/O
- `fs` — Sync file read (for synchronous policy checks)

## Implementation Details

### Eligibility
- **Console users (API key)**: All eligible if actual key is available
- **OAuth users (Claude.ai)**: Only Team and Enterprise/C4E subscribers
- **3rd party providers**: Not eligible
- **Custom base URL users**: Not eligible

### Fetch Logic
1. Check eligibility — return null if not eligible
2. Load cached restrictions from file
3. Compute checksum from cached restrictions (sorted keys, SHA-256)
4. Fetch from API with If-None-Match header (ETag caching)
5. Handle responses:
   - 304: cached version still valid
   - 404: no restrictions exist, delete cache file
   - 200: apply new restrictions, save to cache
6. On failure: use stale cache if available, otherwise fail open (return null)

### Policy Evaluation (Synchronous)
- `isPolicyAllowed(policy)` checks restrictions from session cache or file
- **Fail-open**: Unknown policies or unavailable cache return `true`
- **Exception**: Policies in `ESSENTIAL_TRAFFIC_DENY_ON_MISS` fail closed when essential-traffic-only mode (HIPAA) is active and cache is unavailable
- Only blocked policies are included in the response — absent keys mean allowed

### Background Polling
- Polls every hour via `setInterval` (unref'd to not block exit)
- Compares previous and new cache state, logs changes
- Registered with cleanup registry for shutdown

### Checksum Computation
- Recursively sorts all keys for consistent hashing
- Uses `jsonStringify` (compact, no spaces) to match server-side Python `json.dumps(sort_keys=True, separators=(",", ":"))`
- Format: `sha256:<hex>`

## Data Flow

```
CLI Startup
    ↓
initializePolicyLimitsLoadingPromise() — set up promise early
    ↓
loadPolicyLimits()
    ↓
loadCachedRestrictions() — sync read from file
    ↓
computeChecksum() — from cached data
    ↓
fetchWithRetry() — API call with ETag, up to 5 retries
    ↓
Apply new restrictions / use stale cache / fail open
    ↓
startBackgroundPolling() — hourly refresh
    ↓
isPolicyAllowed() — synchronous check from session cache
```

## Integration Points
- **CLI Initialization**: `loadPolicyLimits()` called during startup
- **Init Module**: `initializePolicyLimitsLoadingPromise()` called early for other systems to await
- **Feature Gates**: `isPolicyAllowed()` called synchronously throughout codebase
- **Auth Changes**: `refreshPolicyLimits()` called on login/logout
- **HIPAA Mode**: `ESSENTIAL_TRAFFIC_DENY_ON_MISS` policies fail closed when essential-traffic-only

## Configuration
- API endpoint: `/api/claude_code/policy_limits`
- Cache file: `~/.claude/policy-limits.json` (mode 0o600)
- Polling interval: 1 hour
- Fetch timeout: 10 seconds
- Max retries: 5 with exponential backoff

## Error Handling
- **Fail-open**: API failures continue without restrictions (no blocking)
- **Stale cache**: On fetch failure, uses cached restrictions if available
- **Auth errors**: Skip retry (401/403 won't succeed on retry)
- **Empty restrictions**: 404 response deletes cache file to prevent stale settings
- **File errors**: Save/load failures logged but don't crash

## Testing
- `_resetPolicyLimitsForTesting()` clears session cache, loading promise, and stops polling
- Cheaper than `clearPolicyLimitsCache()` (no file I/O) for test beforeEach

## Related Modules
- [Remote Managed Settings](./remote-managed-settings.md) — similar patterns (ETag, polling, fail-open)
- [Analytics](./analytics-service.md) — telemetry events

## Notes
- Only blocked policies are included in the API response — absence means allowed
- The loading promise has a 30-second timeout to prevent deadlocks if loadPolicyLimits() is never called
- Session cache is populated from file on first read, avoiding repeated disk I/O
- Essential-traffic deny-on-miss protects HIPAA orgs from cache misses silently re-enabling features
