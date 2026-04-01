# Auth Utility

**File:** `restored-src/src/utils/auth.ts`

## Purpose

Handles all authentication flows for Claude Code, including API key management, OAuth token handling, and cloud provider (AWS/GCP) credential management.

## Key Functions

### API Key Management

- `getAnthropicApiKey()` - Returns the API key string or null
- `getAnthropicApiKeyWithSource()` - Returns key with its source (`ANTHROPIC_API_KEY`, `apiKeyHelper`, `/login managed key`, or `none`)
- `hasAnthropicApiKeyAuth()` - Checks if API key authentication is available
- `saveApiKey(apiKey)` - Saves API key to macOS keychain or config file
- `removeApiKey()` - Removes stored API key
- `isCustomApiKeyApproved(apiKey)` - Checks if a key is in the approved list

### API Key Helper

- `getConfiguredApiKeyHelper()` - Gets the configured helper command from settings
- `getApiKeyFromApiKeyHelper()` - Async execution of the helper with caching and TTL
- `getApiKeyFromApiKeyHelperCached()` - Sync cache read (non-blocking)
- `clearApiKeyHelperCache()` - Clears the helper cache
- `calculateApiKeyHelperTTL()` - Calculates cache TTL from env var (default 5 min)

### OAuth Token Management

- `getClaudeAIOAuthTokens()` - Returns cached OAuth tokens from secure storage
- `getClaudeAIOAuthTokensAsync()` - Async version that reads from storage
- `saveOAuthTokensIfNeeded(tokens)` - Saves OAuth tokens to secure storage
- `clearOAuthTokenCache()` - Clears all OAuth token caches
- `handleOAuth401Error(failedAccessToken)` - Handles 401 errors by forcing token refresh
- `checkAndRefreshOAuthTokenIfNeeded()` - Checks expiration and refreshes if needed

### Cloud Provider Auth

- `refreshAndGetAwsCredentials()` - Refreshes and caches AWS STS credentials (1hr TTL)
- `clearAwsCredentialsCache()` - Clears AWS credential cache
- `refreshGcpCredentialsIfNeeded()` - Refreshes GCP credentials if expired (1hr TTL)
- `clearGcpCredentialsCache()` - Clears GCP credential cache
- `checkGcpCredentialsValid()` - Validates GCP credentials with 5s timeout
- `prefetchAwsCredentialsAndBedRockInfoIfSafe()` - Prefetches AWS creds after trust check
- `prefetchGcpCredentialsIfSafe()` - Prefetches GCP creds after trust check

### Auth Context Detection

- `isAnthropicAuthEnabled()` - Determines if Anthropic auth is active (vs 3P providers)
- `isManagedOAuthContext()` - Detects CCR/claude-desktop managed OAuth context
- `getAuthTokenSource()` - Returns where the current auth token originates

## Security Considerations

- API key helper commands from project/local settings require trust dialog acceptance before execution
- AWS/GCP credential export commands from project settings require trust dialog acceptance
- API keys stored in macOS keychain using hexadecimal encoding to avoid shell escaping issues
- OAuth tokens stored in secure storage with proper scope validation
- Concurrent 401 handler deduplication prevents multiple simultaneous refresh attempts
- Cross-process staleness detection via file mtime monitoring

## Caching Strategy

- API key helper: memoized with configurable TTL (default 5 min), epoch-based invalidation
- AWS credentials: memoized with 1 hour TTL
- GCP credentials: memoized with 1 hour TTL
- OAuth tokens: memoized, invalidated on disk change or 401 errors
- Keychain reads: memoized with 30s TTL

## Dependencies

- `authFileDescriptor.ts` - File descriptor-based token reading
- `authPortable.ts` - Portable auth utilities
- `config.ts` - Global config for approved keys
- `secureStorage/` - Platform-specific secure storage
- `settings/settings.ts` - Settings for helper commands
- `aws.ts` / `awsAuthStatusManager.ts` - AWS-specific auth
