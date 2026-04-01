# McpAuthTool

## Purpose

McpAuthTool is a pseudo-tool created dynamically for MCP servers that are installed but not yet authenticated. It replaces the server's real tools during the unauthenticated state, allowing the model to discover the server exists and initiate the OAuth flow on the user's behalf. When called, it starts the OAuth authentication process, returns an authorization URL for the user to open in their browser, and automatically reconnects the server with real tools once authentication completes.

## Location

- `restored-src/src/tools/McpAuthTool/McpAuthTool.ts` — Main tool factory and OAuth orchestration (216 lines)
- `restored-src/src/services/mcp/auth.ts` — OAuth flow implementation, ClaudeAuthProvider, token management (~1700 lines)
- `restored-src/src/services/mcp/client.ts` — Server reconnection, cache management, auth cache (~2800 lines)
- `restored-src/src/services/mcp/types.ts` — Server configuration schemas (259 lines)
- `restored-src/src/services/mcp/mcpStringUtils.ts` — Tool name building utilities (107 lines)
- `restored-src/src/services/mcp/oauthPort.ts` — OAuth redirect port management (79 lines)
- `restored-src/src/services/mcp/xaa.ts` — Cross-App Access token exchange
- `restored-src/src/services/mcp/xaaIdpLogin.ts` — IdP login and OIDC discovery
- `restored-src/src/utils/secureStorage/` — Platform-specific secure credential storage
- `restored-src/src/utils/log.ts` — MCP debug/error logging

## Key Exports

### From McpAuthTool.ts

| Export | Description |
|--------|-------------|
| `createMcpAuthTool(serverName, config)` | Factory function that creates the auth pseudo-tool |
| `McpAuthOutput` | Output type: `{ status: 'auth_url' | 'unsupported' | 'error', message, authUrl? }` |

### From auth.ts (OAuth core)

| Export | Description |
|--------|-------------|
| `performMCPOAuthFlow(serverName, config, onAuthorizationUrl, abortSignal, options)` | Main OAuth flow orchestrator |
| `ClaudeAuthProvider` | OAuth client provider implementing `OAuthClientProvider` interface |
| `getServerKey(serverName, serverConfig)` | Generates unique storage key from name + config hash |
| `hasMcpDiscoveryButNoToken(serverName, config)` | Checks if server was probed but has no credentials |
| `revokeServerTokens(serverName, config, options)` | Revokes tokens on OAuth server (RFC 7009) |
| `clearServerTokensFromLocalStorage(serverName, config)` | Clears local token storage |
| `wrapFetchWithStepUpDetection(baseFetch, provider)` | Fetch wrapper for 403 step-up detection |
| `normalizeOAuthErrorBody(response)` | Normalizes non-standard OAuth error responses |
| `fetchAuthServerMetadata(serverName, serverUrl, configuredMetadataUrl, fetchFn, resourceMetadataUrl)` | Discovers OAuth server metadata |
| `AuthenticationCancelledError` | Error for user-initiated auth cancellation |

### From client.ts (Auth cache)

| Export | Description |
|--------|-------------|
| `McpAuthError` | Error class for auth failures during tool calls |
| `isMcpAuthCached(serverId)` | Checks 15-minute auth cache for needs-auth state |
| `setMcpAuthCacheEntry(serverId)` | Writes needs-auth entry to cache |
| `clearMcpAuthCache()` | Clears the entire auth cache file |
| `handleRemoteAuthFailure(name, serverRef, transportType)` | Handles 401 during connection |
| `createClaudeAiProxyFetch(innerFetch)` | Fetch wrapper with OAuth token retry for claude.ai proxy |

## Input/Output Schemas

### Input Schema

```typescript
// Empty object — no input parameters needed
{}
```

The tool takes no arguments. Authentication is triggered by calling the tool with an empty input object.

### Output Schema

```typescript
{
  status: 'auth_url' | 'unsupported' | 'error',
  message: string,
  authUrl?: string,  // Present when status is 'auth_url' and URL is available
}
```

| Status | Meaning |
|--------|---------|
| `auth_url` | OAuth flow started successfully; user should open the URL |
| `unsupported` | Server type doesn't support programmatic OAuth (e.g., claudeai-proxy) |
| `error` | OAuth flow failed to start |

## MCP Authentication Flow

### When McpAuthTool is Created

```
1. SERVER CONNECTION ATTEMPT
   - connectToServer() attempts to connect to MCP server
   - Server returns HTTP 401 (Unauthorized)

2. AUTH FAILURE DETECTION
   - SSE: UnauthorizedError from SSEClientTransport
   - HTTP: UnauthorizedError from StreamableHTTPClientTransport
   - claudeai-proxy: HTTP 401 status code

3. NEEDS-AUTH STATE
   - handleRemoteAuthFailure() called:
     a. Emits tengu_mcp_server_needs_auth analytics event
     b. Writes to auth cache (15-min TTL)
     c. Returns { name, type: 'needs-auth', config }

4. AUTH TOOL CREATION
   - createMcpAuthTool(serverName, config) creates pseudo-tool
   - Tool name: mcp__<server>__authenticate
   - Replaces real tools in appState until auth completes

5. SKIP CONDITIONS (avoids unnecessary 401 probes)
   - isMcpAuthCached(): server in 15-min needs-auth cache
   - hasMcpDiscoveryButNoToken(): probed before, no tokens stored
   - XAA servers excluded (can silently re-auth via cached id_token)
```

### OAuth Flow Execution

```
1. TOOL CALL
   - Model calls mcp__server__authenticate with {}

2. TRANSPORT VALIDATION
   - claudeai-proxy → unsupported (manual /mcp auth required)
   - stdio → unsupported (doesn't support OAuth from tool)
   - sse/http → proceed with OAuth flow

3. OAUTH INITIALIZATION
   - performMCPOAuthFlow() called with:
     a. serverName, serverConfig
     b. onAuthorizationUrl callback (captures URL)
     c. abortSignal for cancellation
     d. { skipBrowserOpen: true } (no automatic browser open)

4. AUTH URL CAPTURE
   - Promise race between:
     a. authUrlPromise: resolved when onAuthorizationUrl fires
     b. oauthPromise: resolves when full flow completes
   - Returns authUrl immediately (non-blocking)

5. BACKGROUND COMPLETION
   - OAuth flow continues in background:
     a. User opens URL in browser
     b. Authorizes on provider
     c. Browser redirects to localhost callback
     d. ClaudeAuthProvider receives authorization code
     e. SDK exchanges code for tokens
     f. Tokens saved to secure storage

6. SERVER RECONNECTION
   - oauthPromise.then():
     a. clearMcpAuthCache() — remove needs-auth entry
     b. reconnectMcpServerImpl(serverName, config):
        - clearKeychainCache() — read fresh credentials
        - clearServerCache() — clear memo caches
        - connectToServer() — reconnect with tokens
        - fetchToolsForClient() — discover real tools
        - fetchCommandsForClient() — discover commands
        - fetchResourcesForClient() — discover resources
     c. setAppState():
        - Replace client in mcp.clients
        - Remove old tools (prefix-based: mcp__<server>__*)
        - Add new real tools
        - Update resources
     d. Log: "OAuth complete, reconnected with N tool(s)"

7. ERROR HANDLING
   - oauthPromise.catch():
     - logMCPError() with failure details
     - Auth tool remains (user must retry or run /mcp)
```

### XAA (Cross-App Access) Authentication

```
When oauth.xaa is true in server config:

1. IDP CONFIG
   - getXaaIdpSettings() reads IdP connection details
   - IdP issuer, clientId, callbackPort from settings.xaaIdp
   - Configured once via: claude mcp xaa setup --issuer <url> ...

2. IDP LOGIN
   - acquireIdpIdToken():
     a. Check cache (getCachedIdpIdToken)
     b. If cached → silent (no browser)
     c. If missing/expired → OIDC authorization_code+PKCE flow
        - One browser pop at IdP (shared across all XAA servers)
        - id_token cached in keychain by issuer

3. TOKEN EXCHANGE
   - performCrossAppAccess():
     a. RFC 8693 token exchange (no browser)
     b. RFC 7523 jwt-bearer grant
     c. IdP id_token → MCP server access/refresh tokens

4. TOKEN STORAGE
   - Saved to same keychain slot as normal OAuth
   - Same ClaudeAuthProvider.tokens() path works unchanged
   - discoveryState persisted for refresh/revocation

5. ERROR RECOVERY
   - XaaTokenExchangeError with shouldClearIdToken:
     - 4xx / invalid body → clear cached id_token
     - 5xx IdP outage → preserve cached id_token
   - Next attempt does fresh IdP login if cleared
```

## OAuth Handling for MCP

### ClaudeAuthProvider

The `ClaudeAuthProvider` class implements the MCP SDK's `OAuthClientProvider` interface:

```typescript
class ClaudeAuthProvider implements OAuthClientProvider {
  // Required interface methods:
  get redirectUrl(): string           // http://localhost:<port>/callback
  get clientMetadata(): OAuthClientMetadata  // Client registration info
  clientInformation(): OAuthClientInformation | undefined  // Stored client creds
  saveClientInformation(info: OAuthClientInformationFull): void  // Save DCR result
  tokens(): OAuthTokens | undefined   // Read stored tokens
  saveTokens(tokens: OAuthTokens): void  // Save tokens to secure storage
  redirectToAuthorization(url: string): void  // Trigger browser redirect
  onInvalidAuthorizationCode(code: string): Promise<void>  // Handle bad codes

  // Custom extensions:
  setMetadata(metadata: AuthorizationServerMetadata): void  // Store discovered metadata
  markStepUpPending(scope: string): void  // Flag for step-up auth
  state(): Promise<string>  // Get current OAuth state (for CSRF validation)
}
```

### OAuth Discovery Flow

```
1. METADATA DISCOVERY (fetchAuthServerMetadata)
   Order of attempts:

   a. Configured metadata URL (if set in server config):
      - Must be https://
      - Direct fetch to URL
      - Parse as OAuthMetadataSchema

   b. RFC 9728 discovery (preferred):
      - Probe /.well-known/oauth-protected-resource on MCP server
      - Read authorization_servers[0] from response
      - RFC 8414 discovery against that authorization server URL

   c. RFC 8414 fallback (legacy):
      - Only when server URL has path component
      - Probe /.well-known/oauth-authorization-server/{path}
      - Covers legacy servers co-hosting auth metadata

2. DYNAMIC CLIENT REGISTRATION (DCR)
   - If no client_id configured:
     a. POST to registration_endpoint from metadata
     b. Send client_metadata (redirect_uri, grant_types, etc.)
     c. Save returned client_id and client_secret

3. AUTHORIZATION CODE FLOW (PKCE)
   a. Generate code_verifier (random bytes)
   b. Compute code_challenge = SHA256(code_verifier)
   c. Generate state (random UUID)
   d. Build authorization URL:
      - authorization_endpoint from metadata
      - response_type=code
      - client_id
      - redirect_uri
      - code_challenge + code_challenge_method=S256
      - state
      - scope (if available from WWW-Authenticate or metadata)
      - resource_metadata_url (if available)
   e. Open browser to authorization URL
   f. Wait for callback on localhost:<port>/callback

4. TOKEN EXCHANGE
   a. Receive authorization code from callback
   b. Validate state (prevent CSRF)
   c. POST to token_endpoint:
      - grant_type=authorization_code
      - code
      - redirect_uri
      - code_verifier
      - client_id (and client_secret if confidential)
   d. Receive access_token, refresh_token, expires_in, scope
   e. Save tokens to secure storage

5. TOKEN REFRESH
   - When access_token expires:
     a. POST to token_endpoint:
        - grant_type=refresh_token
        - refresh_token
        - client_id (and client_secret if confidential)
     b. Receive new access_token (and possibly new refresh_token)
     c. Update secure storage
```

### Step-Up Authentication

```
Scenario: Server returns 403 insufficient_scope

1. DETECTION (wrapFetchWithStepUpDetection)
   - Intercept 403 response
   - Parse WWW-Authenticate header
   - Extract scope parameter: scope="read:admin"
   - Call provider.markStepUpPending(scope)

2. STEP-UP TRIGGER
   - tokens() omits refresh_token when stepUpPending is set
   - SDK falls through to PKCE flow (not refresh)
   - User re-authorizes with elevated scope

3. SCOPE PERSISTENCE
   - New scope saved as stepUpScope in secure storage
   - Cached for next performMCPOAuthFlow call
   - Avoids re-probing server for scope requirement

4. CACHED STEP-UP
   - On next auth flow:
     a. Read cachedStepUpScope from storage
     b. Read cachedResourceMetadataUrl from storage
     c. Pass to WWWAuthenticateParams
     d. Skip server probe (already know what's needed)
```

### Token Revocation

```
revokeServerTokens(serverName, config, { preserveStepUpState }):

1. READ STORED TOKENS
   - getSecureStorage().read().mcpOAuth[serverKey]
   - Extract accessToken, refreshToken, clientId, clientSecret

2. DISCOVER REVOCATION ENDPOINT
   - fetchAuthServerMetadata() for OAuth metadata
   - Read revocation_endpoint from metadata
   - Determine auth method:
     - revocation_endpoint_auth_methods_supported (preferred)
     - token_endpoint_auth_methods_supported (fallback)
     - Default: client_secret_basic

3. REVOKE REFRESH TOKEN FIRST (RFC 7009)
   - revokeToken({ token: refreshToken, tokenTypeHint: 'refresh_token' })
   - RFC 7009 compliant: client_id in body, no Authorization header
   - Fallback on 401: retry with Bearer auth (non-compliant servers)

4. REVOKE ACCESS TOKEN
   - revokeToken({ token: accessToken, tokenTypeHint: 'access_token' })
   - May already be invalidated by refresh token revocation

5. CLEAR LOCAL STORAGE
   - clearServerTokensFromLocalStorage()
   - Delete entire mcpOAuth[serverKey] entry

6. PRESERVE STEP-UP STATE (optional)
   - If preserveStepUpState=true:
     - Keep stepUpScope and discoveryState
     - Clear only access/refresh tokens
     - Next auth flow skips server probe
```

### OAuth Error Normalization

```
normalizeOAuthErrorBody(response):

Problem: Some OAuth servers (notably Slack) return HTTP 200 for errors,
signaling via JSON body: {"error": "invalid_refresh_token"}

Solution:
1. Check if response is 2xx
2. Parse JSON body
3. If matches OAuthErrorResponseSchema but not OAuthTokensSchema:
   a. Normalize non-standard error codes:
      - invalid_refresh_token → invalid_grant
      - expired_refresh_token → invalid_grant
      - token_expired → invalid_grant
   b. Rewrite to HTTP 400 response
   c. SDK error mapping applies correctly (InvalidGrantError)

This ensures refresh retry/invalidation logic treats errors correctly.
```

## Token Management

### Secure Storage

```
Storage location: Platform-specific secure storage (keychain/credential manager)
Storage key: getServerKey(serverName, serverConfig)
  - Format: "<serverName>|<configHash>"
  - Config hash: SHA256 of { type, url, headers } (first 16 hex chars)
  - Prevents credential reuse across different configs with same name

Stored data structure:
{
  mcpOAuth: {
    "<serverKey>": {
      serverName: string,
      serverUrl: string,
      accessToken: string,
      refreshToken: string,
      expiresAt: number,           // Timestamp (ms)
      scope: string,
      clientId: string,
      clientSecret: string,
      stepUpScope?: string,        // Cached elevated scope
      discoveryState?: {           // OAuth discovery state
        authorizationServerUrl: string,
        resourceMetadataUrl: string,
      },
    }
  }
}
```

### Auth Cache (Needs-Auth)

```
Cache file: mcp-needs-auth-cache.json
TTL: 15 minutes (MCP_AUTH_CACHE_TTL_MS)

Structure:
{
  "<serverId>": { timestamp: number }
}

Operations:
- isMcpAuthCached(serverId): Check if entry exists and is within TTL
- setMcpAuthCacheEntry(serverId): Write timestamp (serialized via writeChain)
- clearMcpAuthCache(): Delete file and null read cache

Write serialization:
- writeChain: Promise chain prevents concurrent read-modify-write races
- Multiple 401s in same batch don't corrupt cache
- Read cache invalidated on write
```

### OAuth Callback Server

```
Port selection (findAvailablePort):
1. Try MCP_OAUTH_CALLBACK_PORT env var (if set)
2. Random selection in range:
   - Windows: 39152-49151 (below dynamic range)
   - Unix: 49152-65535 (dynamic range)
3. Fallback: 3118 (if random selection fails)
4. Max 100 random attempts

Redirect URI: http://localhost:<port>/callback
  - RFC 8252 Section 7.3: loopback URIs match any port

Callback handling:
1. Create HTTP server on selected port
2. Listen for GET /callback?code=...&state=...
3. Validate state (prevent CSRF)
4. Handle errors:
   - error param → display error page, reject promise
   - state mismatch → reject with "possible CSRF attack"
5. On success:
   - Display "Authentication Successful" page
   - Resolve promise with authorization code
6. Cleanup:
   - server.close()
   - Remove event listeners
   - Clear timeout (5-minute default)

Manual callback URL paste:
- For remote/browser-based environments where localhost unreachable
- onWaitingForCallback callback accepts pasted URL
- Parses code and state, validates same as HTTP callback
```

### Token Refresh

```
Automatic refresh flow (ClaudeAuthProvider.tokens()):

1. Check stored tokens
2. If access_token not expired → return as-is
3. If access_token expired and refresh_token available:
   a. POST to token_endpoint with refresh_token
   b. Handle OAuth errors:
      - invalid_grant → invalidate tokens, trigger re-auth
      - server_error/temporarily_unavailable → retry (max 3 attempts)
      - too_many_requests → retry with backoff
   c. Save new tokens
   d. Return new access_token
4. If no refresh_token → return undefined (triggers re-auth)

Retry behavior for transient errors:
- ServerError (5xx): retry up to 3 times
- TemporarilyUnavailableError: retry up to 3 times
- TooManyRequestsError: retry with Retry-After header
- InvalidGrantError: no retry, invalidate tokens immediately
```

## Configuration

### Server OAuth Configuration

```typescript
McpOAuthConfigSchema:
{
  clientId?: string,              // Pre-configured client ID
  callbackPort?: number,          // Fixed OAuth callback port
  authServerMetadataUrl?: string, // Pre-configured metadata URL (must be https://)
  xaa?: boolean,                  // Enable Cross-App Access
}
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `MCP_OAUTH_CALLBACK_PORT` | Fixed OAuth redirect port |
| `CLAUDE_CODE_ENABLE_XAA` | Enable Cross-App Access (must be "1") |

### Sensitive OAuth Parameters (Redacted from Logs)

| Parameter | Reason |
|-----------|--------|
| `state` | CSRF prevention token |
| `nonce` | Replay attack prevention |
| `code_challenge` | PKCE challenge |
| `code_verifier` | PKCE verifier (secret) |
| `code` | Authorization code |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/mcp/auth.ts` | OAuth flow, ClaudeAuthProvider, token management |
| `services/mcp/client.ts` | Server reconnection, auth cache |
| `services/mcp/types.ts` | Server configuration schemas |
| `services/mcp/mcpStringUtils.ts` | Tool name building |
| `services/mcp/oauthPort.ts` | Port selection, redirect URI |
| `services/mcp/xaa.ts` | Cross-App Access token exchange |
| `services/mcp/xaaIdpLogin.ts` | IdP login, OIDC discovery |
| `utils/secureStorage/` | Platform-specific credential storage |
| `utils/browser.ts` | Browser opening utility |
| `utils/errors.ts` | Error message extraction |
| `utils/log.ts` | MCP debug/error logging |
| `utils/lockfile.ts` | File locking for concurrent access |
| `utils/sleep.ts` | Sleep utility for retries |
| `utils/slowOperations.ts` | JSON parse/stringify |
| `utils/envUtils.ts` | Config home directory resolution |
| `utils/platform.ts` | Platform detection |
| `constants/oauth.ts` | OAuth configuration constants |
| `services/analytics/` | Event logging |

### External

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` | OAuth client, auth types, schemas |
| `axios` | HTTP client for token revocation |
| `crypto` | Hash generation, random bytes, UUID |
| `http` | Local callback server |
| `fs/promises` | Secure storage file operations |
| `xss` | HTML sanitization for error pages |
| `url` | URL parsing |
| `path` | Path joining |
| `lodash-es/reject` | Array filtering for tool replacement |
| `zod/v4` | Input schema validation |

## Data Flow

```
Model Calls Auth Tool
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ AUTH TOOL CALL                                                   │
│                                                                 │
│  1. Transport check                                             │
│     - claudeai-proxy → unsupported                              │
│     - stdio → unsupported                                       │
│     - sse/http → proceed                                        │
│                                                                 │
│  2. OAuth flow start                                            │
│     - performMCPOAuthFlow()                                     │
│     - skipBrowserOpen: true                                     │
│     - Capture auth URL via callback                             │
│                                                                 │
│  3. Return auth URL to model                                    │
│     - status: 'auth_url'                                        │
│     - message: instructions for user                            │
│     - authUrl: URL to open                                      │
└─────────────────────────────────────────────────────────────────┘
    |
    v (background)
┌─────────────────────────────────────────────────────────────────┐
│ BACKGROUND OAUTH COMPLETION                                      │
│                                                                 │
│  1. User opens URL in browser                                   │
│  2. Authorizes on provider                                      │
│  3. Browser redirects to localhost callback                     │
│  4. ClaudeAuthProvider receives code                            │
│  5. SDK exchanges code for tokens                               │
│  6. Tokens saved to secure storage                              │
│                                                                 │
│  7. clearMcpAuthCache()                                         │
│  8. reconnectMcpServerImpl():                                   │
│     - Clear keychain cache                                      │
│     - Clear server caches                                       │
│     - Reconnect with tokens                                     │
│     - Fetch tools, commands, resources                          │
│                                                                 │
│  9. setAppState():                                              │
│     - Replace client                                            │
│     - Remove auth tool (prefix match)                           │
│     - Add real tools                                            │
│     - Update resources                                          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Real MCP Tools Available (auth tool automatically removed)
```
