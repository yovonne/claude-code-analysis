# OAuth Service

## Purpose

The OAuth service implements OAuth 2.0 authorization code flow with PKCE for authenticating Claude Code CLI users with Anthropic services. It handles the complete authentication lifecycle: browser-based and manual auth flows, token exchange, token refresh, profile fetching, secure token storage, and cross-process token synchronization. It also manages API key storage and multi-provider authentication (Anthropic OAuth, API keys, AWS, GCP).

## Location

- `restored-src/src/services/oauth/index.ts` — OAuthService class, main auth flow orchestration
- `restored-src/src/services/oauth/client.ts` — HTTP client for token exchange, refresh, profile, roles, API key creation
- `restored-src/src/services/oauth/crypto.ts` — PKCE code verifier/challenge and state generation
- `restored-src/src/services/oauth/auth-code-listener.ts` — Local HTTP server for OAuth redirect capture
- `restored-src/src/services/oauth/getOauthProfile.ts` — Profile fetching via API key or OAuth token
- `restored-src/src/utils/auth.ts` — Auth utilities: API key management, OAuth token lifecycle, provider detection
- `restored-src/src/utils/secureStorage/index.ts` — Secure storage factory
- `restored-src/src/utils/secureStorage/macOsKeychainStorage.ts` — macOS Keychain implementation
- `restored-src/src/utils/secureStorage/macOsKeychainHelpers.ts` — Shared keychain helpers and cache state
- `restored-src/src/utils/secureStorage/keychainPrefetch.ts` — Parallel keychain read prefetch for startup optimization
- `restored-src/src/utils/secureStorage/fallbackStorage.ts` — Primary/secondary storage fallback wrapper
- `restored-src/src/utils/secureStorage/plainTextStorage.ts` — Plaintext file fallback for non-macOS platforms

## Key Exports

### OAuth Service (`services/oauth/index.ts`)

#### Classes
- `OAuthService`: Main OAuth 2.0 flow orchestrator with PKCE support

#### Methods
- `startOAuthFlow(authURLHandler, options)`: Initiates the full OAuth flow; returns `OAuthTokens`
- `handleManualAuthCodeInput(params)`: Handles user-pasted auth codes for non-browser environments
- `cleanup()`: Closes the auth code listener and clears resources

### OAuth Client (`services/oauth/client.ts`)

#### Functions
- `buildAuthUrl(options)`: Constructs the authorization URL with PKCE parameters
- `exchangeCodeForTokens(code, state, verifier, port, isManual, expiresIn)`: Exchanges auth code for tokens
- `refreshOAuthToken(refreshToken, options)`: Refreshes an access token; fetches profile if needed
- `fetchProfileInfo(accessToken)`: Fetches user profile (subscription type, rate limit tier, billing info)
- `fetchAndStoreUserRoles(accessToken)`: Fetches and stores organization/workspace roles
- `createAndStoreApiKey(accessToken)`: Creates an API key via OAuth and saves it
- `isOAuthTokenExpired(expiresAt)`: Checks if a token is expired (with 5-minute buffer)
- `getOrganizationUUID()`: Gets org UUID from config or profile endpoint
- `populateOAuthAccountInfoIfNeeded()`: Populates OAuth account info from env vars or profile
- `storeOAuthAccountInfo(info)`: Saves account info to global config
- `shouldUseClaudeAIAuth(scopes)`: Checks if Claude.ai auth scope is present
- `parseScopes(scopeString)`: Parses space-delimited scope string into array

### Crypto (`services/oauth/crypto.ts`)

#### Functions
- `generateCodeVerifier()`: Generates a cryptographically random PKCE code verifier (32 bytes, base64url)
- `generateCodeChallenge(verifier)`: Generates SHA-256 code challenge from verifier
- `generateState()`: Generates a random CSRF protection state parameter

### Auth Code Listener (`services/oauth/auth-code-listener.ts`)

#### Classes
- `AuthCodeListener`: Temporary localhost HTTP server that captures OAuth redirect codes

#### Methods
- `start(port?)`: Starts listening on an OS-assigned port
- `waitForAuthorization(state, onReady)`: Waits for the auth code redirect
- `handleSuccessRedirect(scopes, customHandler?)`: Sends browser to success page
- `handleErrorRedirect()`: Sends browser to error page
- `close()`: Cleans up the server and listeners

### Auth Utilities (`utils/auth.ts`)

#### API Key Management
- `getAnthropicApiKey()`: Gets the API key from any source
- `getAnthropicApiKeyWithSource(options)`: Gets API key with source identification
- `saveApiKey(apiKey)`: Saves API key to keychain (macOS) or config
- `removeApiKey()`: Removes API key from all storage locations
- `getApiKeyFromApiKeyHelper(isNonInteractive)`: Executes apiKeyHelper command with caching
- `isCustomApiKeyApproved(apiKey)`: Checks if a custom API key is approved

#### OAuth Token Management
- `getClaudeAIOAuthTokens()`: Gets OAuth tokens from env, file descriptor, or secure storage (memoized)
- `getClaudeAIOAuthTokensAsync()`: Async version that avoids blocking keychain reads
- `saveOAuthTokensIfNeeded(tokens)`: Saves OAuth tokens to secure storage
- `checkAndRefreshOAuthTokenIfNeeded(retryCount, force)`: Auto-refreshes expired tokens with lockfile dedup
- `handleOAuth401Error(failedAccessToken)`: Handles server-side 401 by forcing refresh
- `clearOAuthTokenCache()`: Clears all OAuth token caches
- `isClaudeAISubscriber()`: Checks if user is a Claude.ai subscriber
- `hasProfileScope()`: Checks if token has `user:profile` scope

#### Provider Detection
- `isAnthropicAuthEnabled()`: Checks if direct Anthropic authentication is supported
- `getAuthTokenSource()`: Identifies where the auth token is sourced from
- `isUsing3PServices()`: Checks if using Bedrock, Vertex, or Foundry
- `is1PApiCustomer()`: Checks if user is a direct API customer (not subscriber or 3P)

#### Subscription Management
- `getSubscriptionType()`: Gets subscription type (max, pro, enterprise, team, or null)
- `getSubscriptionName()`: Gets human-readable subscription name
- `getRateLimitTier()`: Gets rate limit tier from OAuth tokens
- `isMaxSubscriber()`, `isProSubscriber()`, `isTeamSubscriber()`, `isEnterpriseSubscriber()`: Type guards

#### Cloud Provider Auth
- `refreshAndGetAwsCredentials()`: Refreshes AWS auth and gets STS credentials (memoized, 1h TTL)
- `refreshGcpCredentialsIfNeeded()`: Refreshes GCP credentials if needed (memoized, 1h TTL)
- `checkGcpCredentialsValid()`: Probes GCP credential validity

#### Organization Validation
- `validateForceLoginOrg()`: Validates OAuth token belongs to the required organization (managed settings)

### Secure Storage (`utils/secureStorage/`)

#### Factory (`index.ts`)
- `getSecureStorage()`: Returns platform-appropriate storage (macOS keychain with plaintext fallback, or plaintext only)

#### macOS Keychain Storage (`macOsKeychainStorage.ts`)
- `macOsKeychainStorage`: SecureStorage implementation using `security` CLI
  - `read()`: Sync read with 30s TTL cache and stale-while-error
  - `readAsync()`: Async read with deduplication and generation tracking
  - `update(data)`: Writes via `security -i` (stdin) to hide payload from process monitors
  - `delete()`: Removes keychain entry
- `isMacOsKeychainLocked()`: Checks if keychain is locked (exit code 36), cached for process lifetime

#### Keychain Helpers (`macOsKeychainHelpers.ts`)
- `getMacOsKeychainStorageServiceName(serviceSuffix)`: Constructs stable service name from config dir hash
- `getUsername()`: Gets current OS username
- `clearKeychainCache()`: Invalidates cache, increments generation, clears in-flight promise
- `primeKeychainCacheFromPrefetch(stdout)`: Primes cache from prefetch result

#### Keychain Prefetch (`keychainPrefetch.ts`)
- `startKeychainPrefetch()`: Fires parallel `security` subprocesses at main.tsx top-level
- `ensureKeychainPrefetchCompleted()`: Awaits prefetch completion in preAction
- `getLegacyApiKeyPrefetchResult()`: Gets legacy API key prefetch result for sync reader bypass
- `clearLegacyApiKeyPrefetch()`: Clears prefetch result on cache invalidation

#### Fallback Storage (`fallbackStorage.ts`)
- `createFallbackStorage(primary, secondary)`: Creates storage that tries primary first, falls back to secondary

#### Plaintext Storage (`plainTextStorage.ts`)
- `plainTextStorage`: SecureStorage implementation using `.credentials.json` file (chmod 0o600)

## Dependencies

### Internal Dependencies
- `constants/oauth.js` — OAuth URLs, client ID, scopes, beta headers
- `utils/config.js` — Global config read/write, account info, trust dialog state
- `utils/browser.js` — Browser opening utility
- `utils/http.js` — Auth header generation
- `utils/user.js` — User data and device ID
- `utils/debug.js` — Debug logging for ant builds
- `utils/log.js` — Error logging
- `utils/envUtils.js` — Config directory paths, environment detection
- `utils/errors.js` — Error utilities
- `utils/betas.js` — Beta flag caches
- `utils/lockfile.js` — Cross-process lockfile for token refresh dedup
- `utils/sleep.js` — Sleep utility
- `utils/authFileDescriptor.js` — File descriptor-based token reading
- `utils/authPortable.js` — Cross-platform API key normalization
- `utils/aws.js` — AWS STS caller identity validation
- `utils/awsAuthStatusManager.js` — Cloud provider auth status tracking
- `utils/settings/settings.js` — Settings loading from multiple sources
- `utils/model/model.js` — Model name utilities
- `utils/model/providers.js` — API provider detection
- `services/analytics/index.js` — Event logging for OAuth events
- `bootstrap/state.js` — Session state (non-interactive, third-party auth preference)

### External Dependencies
- `axios` — HTTP client for OAuth token exchange, profile, and API calls
- `execa` / `execaSync` — Process spawning for `security` CLI and helper commands
- `child_process` — Native Node.js subprocess for keychain prefetch
- `crypto` — Node.js crypto for PKCE (SHA-256, random bytes) and keychain hex encoding
- `fs` / `fs/promises` — File operations for plaintext storage and credential files
- `path` / `os` — Path and OS utilities
- `chalk` — Terminal output formatting for auth errors
- `lodash-es/memoize` — Memoization for caching

## Implementation Details

### OAuth Flow Implementation

The OAuth service implements the **Authorization Code Flow with PKCE** (RFC 7636), supporting two parallel authorization paths:

#### Automatic Flow (Browser-Based)

1. **PKCE Generation**: `generateCodeVerifier()` creates a 32-byte random verifier; `generateCodeChallenge()` derives the SHA-256 challenge
2. **State Generation**: `generateState()` creates a CSRF protection state parameter
3. **Local Server**: `AuthCodeListener` starts an HTTP server on an OS-assigned localhost port
4. **Auth URL Construction**: `buildAuthUrl()` builds the authorization URL with:
   - `client_id`, `response_type=code`, `code_challenge`, `code_challenge_method=S256`
   - `redirect_uri=http://localhost:{port}/callback`
   - `scope` (either `user:inference` for inference-only or full `ALL_OAUTH_SCOPES`)
   - Optional: `orgUUID`, `login_hint`, `login_method`
5. **Browser Launch**: `openBrowser()` opens the auth URL
6. **Redirect Capture**: The local server receives the redirect at `/callback?code=...&state=...`
7. **State Validation**: The received state is compared against the expected state (CSRF protection)
8. **Token Exchange**: `exchangeCodeForTokens()` POSTs to the token endpoint with the auth code and code verifier
9. **Profile Fetch**: `fetchProfileInfo()` retrieves subscription type and rate limit tier
10. **Success Redirect**: Browser is redirected to the appropriate success page (Claude.ai or Console)

#### Manual Flow (Non-Browser Environments)

1. Same PKCE/state generation as automatic flow
2. Auth URL built with `redirect_uri` pointing to the manual redirect URL
3. URL displayed to user for manual copying
4. User pastes the auth code via `handleManualAuthCodeInput()`
5. Token exchange proceeds identically

#### Flow Options

- `loginWithClaudeAi`: Routes to Claude.ai authorize URL vs. Console authorize URL
- `inferenceOnly`: Requests only `user:inference` scope (long-lived tokens without full profile access)
- `orgUUID`: Pre-selects an organization
- `loginHint`: Pre-populates email on login form
- `loginMethod`: Requests specific login method (SSO, magic link, Google)
- `skipBrowserOpen`: Caller handles URL display (used by SDK control protocol)
- `expiresIn`: Custom token expiration

### Token Management

#### Token Structure (`OAuthTokens`)

```typescript
{
  accessToken: string
  refreshToken: string | null
  expiresAt: number | null
  scopes: string[]
  subscriptionType: 'max' | 'pro' | 'enterprise' | 'team' | null
  rateLimitTier: string | null
  profile?: OAuthProfileResponse
  tokenAccount?: { uuid, emailAddress, organizationUuid }
}
```

#### Token Storage

Tokens are stored in **secure storage** with a platform-appropriate backend:

| Platform | Primary Storage | Fallback |
|----------|----------------|----------|
| macOS | Keychain (`security` CLI) | Plaintext `.credentials.json` |
| Linux/Windows | Plaintext `.credentials.json` | None |

The storage path is `~/.claude/.credentials.json` (or `CLAUDE_CONFIG_DIR` override). On macOS, the keychain service name is `Claude Code-credentials` with an optional config dir hash suffix.

#### Token Retrieval Priority

`getClaudeAIOAuthTokens()` checks sources in order:

1. **Bare mode check**: Returns `null` immediately in `--bare` mode
2. **Environment variable**: `CLAUDE_CODE_OAUTH_TOKEN` (inference-only token)
3. **File descriptor**: `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` or CCR OAuth token file
4. **Secure storage**: Keychain (macOS) or `.credentials.json` (all platforms)

The result is **memoized** to avoid repeated expensive keychain reads.

### Authentication State

#### Auth Source Detection

`getAuthTokenSource()` identifies where authentication is coming from:

| Source | Description |
|--------|-------------|
| `apiKeyHelper` | External command configured in settings |
| `ANTHROPIC_AUTH_TOKEN` | Direct bearer token env var |
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuth token from env var |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | OAuth token from inherited FD |
| `CCR_OAUTH_TOKEN_FILE` | OAuth token from CCR disk fallback |
| `claude.ai` | OAuth tokens from secure storage |
| `none` | No authentication configured |

#### Anthropic Auth Enablement

`isAnthropicAuthEnabled()` returns `false` when:
- Bare mode (`--bare`)
- SSH remote with no OAuth token placeholder
- Third-party providers (Bedrock, Vertex, Foundry)
- External API key or auth token configured (unless in managed OAuth context)

Managed OAuth contexts (CCR remote, Claude Desktop) prevent fallback to user's local API key settings.

### Session Handling

#### Token Refresh

`checkAndRefreshOAuthTokenIfNeeded()` implements a robust refresh mechanism:

1. **Cache invalidation**: Checks if `.credentials.json` file has been modified by another process
2. **Expiration check**: Uses 5-minute buffer (`isOAuthTokenExpired()`) to preemptively refresh
3. **Async re-read**: Clears memoize cache and re-reads from secure storage (another process may have refreshed)
4. **Lockfile dedup**: Acquires a file-based lock to prevent concurrent refreshes across processes
   - Retry with jitter on lock contention (max 5 retries, 1-2s backoff)
   - Double-check expiration after acquiring lock (another process may have refreshed)
5. **Scope handling**: Omits scopes for Claude.ai subscribers to allow scope expansion on refresh
6. **Token save**: Stores refreshed tokens to secure storage
7. **Cache clearing**: Clears all caches after refresh

#### 401 Error Handling

`handleOAuth401Error()` handles server-side token expiration:

1. **Deduplication**: Concurrent calls with the same failed token are deduplicated
2. **Cache clearing**: Forces re-read from secure storage
3. **Token comparison**: If keychain has a different token, another tab already refreshed — use it
4. **Force refresh**: If same token, forces refresh bypassing local expiration check

#### Cross-Process Staleness

Multiple Claude Code instances may run simultaneously. The system handles this via:

- **File modification time tracking**: `invalidateOAuthCacheIfDiskChanged()` compares `.credentials.json` mtime
- **Lockfile synchronization**: Prevents concurrent token refreshes
- **Async re-reads**: `getClaudeAIOAuthTokensAsync()` avoids blocking the sync memoize cache
- **401 handler dedup**: `pending401Handlers` Map prevents duplicate refresh attempts

### Keychain/Secure Storage Integration

#### macOS Keychain Architecture

The keychain storage uses the `security` CLI tool with several optimizations:

1. **Service naming**: `Claude Code-credentials[-{configDirHash}]` — unique per config directory
2. **Hex encoding**: Payload is hex-encoded to avoid shell escaping issues
3. **Stdin mode**: Uses `security -i` (interactive mode) so process monitors see only `security -i`, not the payload (INC-3028)
4. **Argv fallback**: Falls back to argv when payload exceeds 4032-byte stdin line limit (4096 - 64 headroom)
5. **30-second TTL cache**: Bounds cross-process staleness without forcing blocking spawns on every read
6. **Stale-while-error**: If a read fails but a cached value exists, serve the stale value rather than returning null
7. **Generation tracking**: `readAsync()` captures generation before spawning; skips cache write if a newer generation exists (prevents stale subprocess results from overwriting fresh `update()` data)
8. **In-flight deduplication**: Concurrent `readAsync()` calls share a single subprocess

#### Keychain Prefetch Optimization

At `main.tsx` top-level, two `security` subprocesses are fired in parallel:
1. OAuth credentials entry (`Claude Code-credentials`)
2. Legacy API key entry (`Claude Code`)

These run in parallel with the ~65ms of module imports. `ensureKeychainPrefetchCompleted()` is awaited in preAction alongside MDM settings loading — nearly free since subprocesses finish during import evaluation. The sync `read()` and `getApiKeyFromConfigOrMacOSKeychain()` then hit their caches instead of spawning new subprocesses.

#### Fallback Storage Strategy

`createFallbackStorage(primary, secondary)` implements a resilient storage pattern:

- **Read**: Try primary first; fall back to secondary
- **Update**: Write to primary; if primary fails, write to secondary
  - If primary write fails but secondary succeeds, delete the stale primary entry (prevents shadowing fresh data)
  - If primary was empty before update, delete secondary (migration cleanup)
- **Delete**: Delete from both primary and secondary

### Token Refresh Flow

```
checkAndRefreshOAuthTokenIfNeeded()
  │
  ├─ invalidateOAuthCacheIfDiskChanged()
  │    └─ stat('.credentials.json') → mtime changed? → clearOAuthTokenCache()
  │
  ├─ getClaudeAIOAuthTokens() (memoized)
  │    └─ Not expired? → return false (no refresh needed)
  │
  ├─ getClaudeAIOAuthTokens.cache.clear() + clearKeychainCache()
  ├─ getClaudeAIOAuthTokensAsync() (async re-read)
  │    └─ Not expired? → return false (another process refreshed)
  │
  ├─ lockfile.lock(claudeDir)
  │    └─ ELOCKED? → retry with jitter (max 5)
  │
  ├─ Double-check expiration after lock
  │    └─ Not expired? → return false (race resolved)
  │
  ├─ refreshOAuthToken(refreshToken)
  │    ├─ POST to token endpoint with grant_type=refresh_token
  │    ├─ Parse response (access_token, refresh_token, expires_in, scope)
  │    └─ fetchProfileInfo(accessToken) (if profile fields not cached)
  │
  ├─ saveOAuthTokensIfNeeded(refreshedTokens)
  │    └─ secureStorage.update({ claudeAiOauth: {...} })
  │
  └─ clearOAuthTokenCache() + clearKeychainCache()
```

### Cross-App Authentication (XAA)

The system supports authentication across multiple Claude Code entry points:

#### Claude Code Remote (CCR)

- Launcher spawns CLI with OAuth token via file descriptor
- `CLAUDE_CODE_OAUTH_TOKEN` env var set as placeholder iff local side is a subscriber
- Remote's local settings (apiKeyHelper, ANTHROPIC_API_KEY) must not override — would cause header mismatch with the auth-injecting proxy
- CCR subprocesses that can't inherit the pipe FD use a disk fallback file

#### Claude Desktop (CCD)

- `CLAUDE_CODE_ENTRYPOINT === 'claude-desktop'` identifies managed sessions
- Prevents fallback to user's terminal CLI settings (apiKeyHelper, ANTHROPIC_API_KEY)
- OAuth tokens managed by Claude Desktop, not the CLI

#### SDK / Cowork

- Environment variables provide account info directly: `CLAUDE_CODE_ACCOUNT_UUID`, `CLAUDE_CODE_USER_EMAIL`, `CLAUDE_CODE_ORGANIZATION_UUID`
- `skipBrowserOpen` option lets SDK caller handle URL display
- `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` for token passing via FD

#### SSH Remote

- `ANTHROPIC_UNIX_SOCKET` indicates SSH tunnel with local auth-injecting proxy
- Placeholder token in `CLAUDE_CODE_OAUTH_TOKEN` indicates subscriber status
- Local side ran org validation before establishing session; remote skips redundant validation

### API Key Management

#### Key Sources (Priority Order)

1. `ANTHROPIC_API_KEY` env var (if approved in config)
2. File descriptor (`CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`)
3. `apiKeyHelper` command (external script, cached with TTL)
4. macOS Keychain (`/login managed key`)
5. Config file (`primaryApiKey`)

#### API Key Helper

The `apiKeyHelper` is a user-configured external command that returns an API key:

- **Async execution with sync cache**: `getApiKeyFromApiKeyHelper()` runs the command asynchronously but caches the result with configurable TTL (default 5 minutes, env: `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`)
- **Epoch-based invalidation**: Settings changes bump the epoch, orphaning in-flight executions
- **Stale-while-refresh**: Stale cache returns immediately while background refresh runs
- **Trust gating**: If helper is from project/local settings, workspace trust must be established first
- **Error handling**: Failures cache a sentinel value (`' '`) to prevent fallback to OAuth

#### Key Storage (macOS)

- Stored in keychain via `security add-generic-password` with stdin mode (process monitor protection)
- Also added to `customApiKeyResponses.approved` list in config for approval tracking
- Memoize cache cleared after save

## Configuration

### OAuth Constants

| Constant | Purpose |
|----------|---------|
| `CLIENT_ID` | OAuth client ID |
| `CONSOLE_AUTHORIZE_URL` | Console authorization endpoint |
| `CLAUDE_AI_AUTHORIZE_URL` | Claude.ai authorization endpoint |
| `TOKEN_URL` | Token exchange endpoint |
| `MANUAL_REDIRECT_URL` | Redirect URL for manual auth flow |
| `CONSOLE_SUCCESS_URL` | Console post-auth success page |
| `CLAUDEAI_SUCCESS_URL` | Claude.ai post-auth success page |
| `ROLES_URL` | User roles endpoint |
| `API_KEY_URL` | API key creation endpoint |
| `BASE_API_URL` | Base API URL for profile fetching |
| `OAUTH_FILE_SUFFIX` | Suffix for keychain service naming |
| `ALL_OAUTH_SCOPES` | Full set of OAuth scopes |
| `CLAUDE_AI_OAUTH_SCOPES` | Claude.ai-specific scopes |
| `CLAUDE_AI_INFERENCE_SCOPE` | Inference-only scope (`user:inference`) |
| `CLAUDE_AI_PROFILE_SCOPE` | Profile scope (`user:profile`) |
| `OAUTH_BETA_HEADER` | Beta header for OAuth profile endpoint |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Force-set OAuth access token (inference-only) |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | OAuth token via file descriptor |
| `CLAUDE_CODE_ACCOUNT_UUID` | Account UUID (SDK/cowork) |
| `CLAUDE_CODE_USER_EMAIL` | User email (SDK/cowork) |
| `CLAUDE_CODE_ORGANIZATION_UUID` | Organization UUID (SDK/cowork) |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | API key helper cache TTL override |
| `CLAUDE_CODE_ENTRYPOINT` | Entry point identifier (`claude-desktop`, `local-agent`, etc.) |
| `CLAUDE_CODE_REMOTE` | Remote session flag |
| `ANTHROPIC_UNIX_SOCKET` | SSH auth proxy socket path |
| `CLAUDE_CONFIG_DIR` | Custom config directory (affects keychain service name) |

## Error Handling

- **Token exchange failures**: Throw with descriptive messages (401 = invalid code, other = status + statusText)
- **Token refresh failures**: Logged to analytics with error message and response body; re-thrown to caller
- **401 retry without auth**: 1P event logging retries without auth headers on 401
- **Keychain failures**: Graceful fallback to plaintext storage; stale-while-error serves cached values
- **Lockfile contention**: Retry with jitter up to 5 times; return false if max retries exceeded
- **API key helper failures**: Error message printed to console; sentinel value cached to prevent OAuth fallback
- **Profile fetch failures**: Silently logged; existing cached profile values preserved (don't clobber valid subscription type with null)
- **Org validation failures**: Returns descriptive error message rather than throwing; fail-closed when `forceLoginOrgUUID` is set

## Related Modules

- [Analytics Service](./analytics-service.md) — Event logging for OAuth events
- [Auth Commands](../03-commands/auth-commands.md) — CLI auth commands (login, logout)
- [Config Utils](../05-utils/config.md) — Global configuration and account info storage
- [Secure Storage](../05-utils/secure-storage.md) — Credential storage implementations
- [Bootstrap State](../08-internals/bootstrap.md) — Session state and trust management
