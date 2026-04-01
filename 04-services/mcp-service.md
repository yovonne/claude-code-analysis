# MCP Service

## Overview

The MCP (Model Context Protocol) service manages connections to MCP servers, enabling Claude Code to interact with external tools, resources, and prompts through the standardized MCP protocol. It supports multiple transport types (stdio, SSE, HTTP, WebSocket, SDK, in-process), OAuth authentication, server configuration across multiple scopes, and an official MCP registry.

**Key files:**
- `types.ts` — Type definitions, config schemas, connection states
- `client.ts` — MCP client creation, transport setup, tool execution
- `config.ts` — Multi-scope configuration management, policy filtering
- `auth.ts` — OAuth authentication for MCP servers
- `officialRegistry.ts` — Official MCP server URL registry
- `normalization.ts` — Server name normalization
- `utils.ts` — Tool/command/resource filtering utilities
- `MCPConnectionManager.tsx` — React context for connection management
- `useManageMCPConnections.ts` — Connection lifecycle hook

---

## MCP Client Architecture

### Client Creation (`client.ts`)

The `connectToServer()` function (memoized by server name + config) is the core entry point for establishing MCP connections. It:

1. Selects the appropriate transport based on server config type
2. Creates an MCP `Client` instance with capability declarations
3. Connects via the transport with timeout protection
4. Registers request handlers (ListRoots, Elicitation)
5. Sets up connection lifecycle monitoring (error/close handlers)
6. Returns a `ConnectedMCPServer`, `FailedMCPServer`, or `NeedsAuthMCPServer`

### Client Instance

Each MCP client is initialized with:
```typescript
new Client({
  name: 'claude-code',
  title: 'Claude Code',
  version: MACRO.VERSION,
  description: "Anthropic's agentic coding tool",
  websiteUrl: PRODUCT_URL,
}, {
  capabilities: {
    roots: {},        // Supports listRoots
    elicitation: {},  // Supports elicitation requests
  },
})
```

### Connection States

Servers transition through five states defined in `types.ts`:

| State | Description |
|---|---|
| `connected` | Active connection with client, capabilities, server info |
| `failed` | Connection failed with error details |
| `needs-auth` | OAuth authentication required |
| `pending` | Connection in progress with reconnect tracking |
| `disabled` | Server explicitly disabled by user |

### Memoization

`connectToServer` is memoized using lodash `memoize` with a cache key of `${name}-${JSON.stringify(serverRef)}`, preventing duplicate connections to the same server configuration within a session.

---

## Server Management (Transports)

### Transport Types

The service supports eight transport types, each with distinct setup:

#### stdio

- Spawns a subprocess via `StdioClientTransport`
- Command from config, args merged with env vars
- `CLAUDE_CODE_SHELL_PREFIX` env var wraps command for shell execution
- stderr piped for debugging (capped at 64MB to prevent memory growth)
- Subprocess environment built from `subprocessEnv()` + server-specific env

#### SSE (Server-Sent Events)

- Uses `SSEClientTransport` with `ClaudeAuthProvider`
- EventSource connection (GET) is **not** timeout-wrapped (long-lived stream)
- POST requests wrapped with `wrapFetchWithTimeout()` (60s per request)
- Step-up auth detection wraps innermost for 403 visibility
- Custom headers from static config + dynamic `headersHelper` functions

#### SSE-IDE

- Variant for IDE extensions
- No authentication required
- Proxy-aware EventSource setup

#### HTTP (Streamable HTTP)

- Uses `StreamableHTTPClientTransport`
- Requires `Accept: application/json, text/event-stream` header per MCP spec
- OAuth auth provider with step-up detection
- Session ingress token support for CCR proxy mode
- HTTP connectivity test before connection attempt
- Basic connectivity logging (URL parsing, DNS resolution)

#### WebSocket

- Uses custom `WebSocketTransport` wrapping `ws` or native WebSocket
- Protocol: `['mcp']`
- TLS options via `getWebSocketTLSOptions()`
- Proxy support via `getWebSocketProxyAgent()`
- Session ingress auth token via `X-Claude-Code-Ide-Authorization` header

#### WS-IDE

- IDE-specific WebSocket variant
- Auth token from config
- IDE name tracking

#### SDK

- Placeholder for SDK-managed transports
- Tool calls route back to SDK via `mcp_tool_call`
- Exempt from enterprise policy filtering

#### claudeai-proxy

- Routes through Claude.ai MCP proxy
- URL constructed from OAuth config: `{MCP_PROXY_URL}{MCP_PROXY_PATH.replace('{server_id}', id)}`
- Custom fetch wrapper (`createClaudeAiProxyFetch`) with OAuth token refresh on 401
- Session ID header: `X-Mcp-Client-Session-Id`

#### In-Process (special stdio)

- Chrome MCP and Computer Use MCP run in-process to avoid ~325MB subprocess overhead
- Uses `InProcessTransport` with linked transport pair
- Server created via factory functions, connected to server-side transport

### Connection Timeout

- Default: `MCP_TIMEOUT` env var or 30 seconds
- Enforced via `Promise.race([connectPromise, timeoutPromise])`
- Timeout cleans up in-process servers and transport

### Connection Drop Detection

Enhanced error monitoring tracks:
- Consecutive connection errors (threshold: 3 before manual close)
- Terminal error detection: `ECONNRESET`, `ETIMEDOUT`, `EPIPE`, `EHOSTUNREACH`, `ECONNREFUSED`, `Body Timeout Error`, `terminated`, SSE stream errors
- Re-entry guard prevents double-close during in-flight stream abort
- Manual close triggers reconnection on next tool call via memo cache clear

### Session Expiry

`isMcpSessionExpiredError()` detects expired MCP sessions via HTTP 404 + JSON-RPC code `-32001`. Throws `McpSessionExpiredError` to signal caller to reconnect.

---

## Tool Discovery and Registration

### Tool Discovery

After connection, tools are discovered via `client.listTools()`:

1. Tools are fetched from the connected server
2. Each tool is wrapped in an `MCPTool` instance
3. Tool names are normalized: `mcp__<normalized_server_name>__<tool_name>`
4. Descriptions capped at 2048 characters (prevents OpenAPI-generated bloat)
5. Tool result persistence configured via `persistToolResult`

### Tool Naming

The `buildMcpToolName()` function constructs tool names:
```
mcp__<normalized_server_name>__<original_tool_name>
```

Normalization (`normalizeNameForMCP`):
- Replaces invalid characters (dots, spaces) with underscores
- For claude.ai servers: collapses consecutive underscores, strips leading/trailing
- API-compatible pattern: `^[a-zA-Z0-9_-]{1,64}$`

### Tool Execution

`callTool()` on the MCP client:
- Default timeout: `MCP_TOOL_TIMEOUT` env var or ~27.8 hours (effectively infinite)
- Result content processed: text, images, binary content persistence
- Image content resized/downsampled via `maybeResizeAndDownsampleImageBuffer()`
- Binary content persisted to disk via `persistBinaryContent()`
- Truncation applied if content exceeds limits
- `isError: true` results throw `McpToolCallError` (carries `_meta` for consumers)
- Auth failures (401) throw `McpAuthError` and update server status to `needs-auth`

### IDE Tool Filtering

For IDE MCP servers, only specific tools are included:
- `mcp__ide__executeCode`
- `mcp__ide__getDiagnostics`

---

## Resource Handling

### Resource Discovery

Resources are discovered via `client.listResources()` after connection:
- Each resource tagged with its server name: `ServerResource = Resource & { server: string }`
- Stored per-server in the connection state

### Resource Reading

`ReadMcpResourceTool` handles resource reads:
- Calls `client.readResource(uri)`
- Content processed similarly to tool results (text, images, binary)
- Resource subscription support detected via `capabilities.resources?.subscribe`

### Resource Listing

`ListMcpResourcesTool` provides a CLI tool to list resources across all connected servers.

---

## MCP Protocol Implementation

### Request Handlers

The client registers handlers for server-to-client requests:

#### ListRoots

```typescript
client.setRequestHandler(ListRootsRequestSchema, async () => ({
  roots: [{ uri: `file://${getOriginalCwd()}` }],
}))
```

Returns the project root as the only available root.

#### Elicitation

- Default handler returns `{ action: 'cancel' }` during initialization
- Overwritten by `registerElicitationHandler` in `useManageMCPConnections`
- Hooks: `runElicitationHooks()` and `runElicitationResultHooks()` manage the elicitation lifecycle
- Waiting state tracked via `ElicitationWaitingState`

### Capability Declaration

The client declares these capabilities to servers:
- `roots: {}` — Supports root listing
- `elicitation: {}` — Supports elicitation (empty object to avoid breaking Java MCP SDK servers)

### Transport Protocol Details

#### Streamable HTTP Accept Header

All HTTP POST requests include:
```
Accept: application/json, text/event-stream
```

This is enforced by `wrapFetchWithTimeout()` which normalizes headers and guarantees the value, as some runtimes/agents drop it from SDK-set headers.

#### Request Timeout Wrapping

`wrapFetchWithTimeout()`:
- Skips GET requests (long-lived SSE streams)
- Creates fresh `AbortController` per request (avoids `AbortSignal.timeout()` memory leak in Bun)
- 60-second timeout for POST requests
- Properly cleans up timers and abort listeners

#### Step-Up Auth Detection

`wrapFetchWithStepUpDetection()` wraps the fetch to detect 403 step-up auth requirements and trigger re-authentication before the SDK's auth handler sees the error.

---

## OAuth Authentication

### ClaudeAuthProvider (`auth.ts`)

Implements `OAuthClientProvider` for MCP SDK authentication:

1. **Metadata Discovery:** Fetches OAuth metadata from `.well-known/oauth-authorization-server` or configured `authServerMetadataUrl`
2. **Dynamic Client Registration:** Registers client if no stored client info exists
3. **Token Exchange:** PKCE flow with local HTTP server for callback
4. **Token Refresh:** Automatic refresh with retry for transient errors
5. **Token Storage:** Secure storage via platform keychain/secure storage

### OAuth Flow

1. Find available port for callback (`findAvailablePort`)
2. Build redirect URI with `buildRedirectUri`
3. Open browser for authorization
4. Local HTTP server receives callback
5. Exchange code for tokens
6. Store tokens in secure storage
7. Cross-App Access (XAA) token exchange if enabled

### Token Refresh Retry

Refresh failures are retried for transient errors:
- `ServerError`, `TemporarilyUnavailableError`, `TooManyRequestsError`
- Max retries for transient errors
- `InvalidGrantError` triggers token invalidation (requires re-auth)

### Non-Standard OAuth Servers

Some servers (notably Slack) return HTTP 200 with error JSON body. The custom fetch wrapper detects this and rewrites to 400 so the SDK's error mapping works correctly. Non-standard `invalid_grant` aliases are normalized:
- `invalid_refresh_token`
- `expired_refresh_token`
- `token_expired`

### MCP Auth Cache

A file-based cache (`mcp-needs-auth-cache.json`) tracks servers needing auth:
- TTL: 15 minutes
- Memoized reads prevent N file reads during batched connection
- Serialized writes prevent concurrent read-modify-write races
- Cleared on successful auth

### Claude.ai Proxy Auth

`createClaudeAiProxyFetch()` wraps fetch for claude.ai proxy connections:
- Attaches OAuth bearer token
- Retries once on 401 via `handleOAuth401Error` (force-refresh)
- Prevents mass-401 of all connectors from stale token in memoize cache

---

## Official MCP Registry

### Overview (`officialRegistry.ts`)

Fire-and-forget fetch of Anthropic's official MCP server registry at startup:

```
GET https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial
```

### Behavior

1. Skipped if `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` is set
2. 5-second timeout
3. Extracts server remote URLs, normalizes them (strip query string, trailing slash)
4. Stores in `Set<string>` for O(1) lookup
5. `isOfficialMcpUrl(normalizedUrl)` checks membership
6. Fails closed: undefined registry → `false`

### Purpose

Used for analytics to distinguish official vs third-party MCP servers in telemetry events.

---

## Server Configuration

### Configuration Scopes (`config.ts`)

MCP servers can be configured at multiple scopes with precedence order:

| Scope | Source | File/Location |
|---|---|---|
| `enterprise` | Managed policy | `{configHome}/managed-mcp.json` |
| `local` | Project-local (in-memory) | Current project config |
| `project` | `.mcp.json` files | Traverses from CWD to root |
| `user` | Global config | Global Claude config |
| `dynamic` | Runtime injection | Passed at startup |
| `claudeai` | Claude.ai connectors | Fetched from API |
| `managed` | Enterprise managed | Same as enterprise |

### Config Schema Validation

All configs validated against Zod schemas:
- `McpStdioServerConfigSchema` — command (required, non-empty), args, env
- `McpSSEServerConfigSchema` — url, headers, headersHelper, oauth
- `McpHTTPServerConfigSchema` — url, headers, headersHelper, oauth
- `McpWebSocketServerConfigSchema` — url, headers, headersHelper
- `McpSSEIDEServerConfigSchema` — url, ideName
- `McpWebSocketIDEServerConfigSchema` — url, ideName, authToken
- `McpSdkServerConfigSchema` — name
- `McpClaudeAIProxyServerConfigSchema` — url, id

### Environment Variable Expansion

Server configs support `${VAR}` and `$VAR` syntax:
- Expanded at load time via `expandEnvVarsInString()`
- Missing variables reported as warnings (not errors)
- Applied to: command, args, env values, URLs, headers

### Enterprise Policy

#### Allowlist/Denylist

Enterprise admins can control MCP server access via:
- `allowedMcpServers` — Name, command, or URL-based allowlist (wildcard URL patterns supported)
- `deniedMcpServers` — Name, command, or URL-based denylist (always takes precedence)

Policy settings can be locked to managed-only via `allowManagedMcpServersOnly`.

#### Exclusive Control

When enterprise MCP config exists (`managed-mcp.json`):
- All other scopes are ignored
- Enterprise config has exclusive control
- Users cannot add their own servers

### Deduplication

#### Plugin Server Dedup

`dedupPluginMcpServers()` prevents duplicate servers from plugins:
- Signature based on command array (stdio) or URL (remote)
- Manual servers win over plugin servers
- Between plugins, first-loaded wins
- Suppressed servers tracked for UI display

#### Claude.ai Connector Dedup

`dedupClaudeAiMcpServers()` prevents connector/ manual duplicates:
- Content-based signature matching (URL/command)
- Manual servers win over connectors
- Only enabled manual servers count as dedup targets

#### CCR Proxy URL Unwrapping

`unwrapCcrProxyUrl()` extracts original vendor URL from CCR proxy URLs for signature matching:
- Detects `/v2/session_ingress/shttp/mcp/` or `/v2/ccr-sessions/` path markers
- Extracts `mcp_url` query parameter

### Server Enable/Disable

- `disabledMcpServers` — List of server names to disable
- `enabledMcpServers` — Explicit opt-in list (overrides disabled)
- Built-in servers (e.g., Computer Use) default to disabled, require explicit opt-in
- Reserved names: `claude-in-chrome`, computer use server name

### Config File Operations

`.mcp.json` writes use atomic file operations:
1. Write to temp file (`{path}.tmp.{pid}.{timestamp}`)
2. `fsync` to flush to disk
3. Atomic rename
4. Preserve original file permissions

---

## Connection Management

### MCPConnectionManager

React context component (`MCPConnectionManager.tsx`) provides:
- `reconnectMcpServer(serverName)` — Reconnect a specific server
- `toggleMcpServer(serverName)` — Enable/disable a server

Backed by `useManageMCPConnections()` hook which manages:
- Initial connection batch (parallel with configurable batch size)
- Reconnection on failure
- Server enable/disable toggling
- Dynamic config updates

### Connection Batching

- Local servers: batch size 3 (`MCP_SERVER_CONNECTION_BATCH_SIZE`)
- Remote servers: batch size 20 (`MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE`)
- Prevents resource exhaustion during mass connection

### Auth Status Checking

`isMcpAuthCached()` checks the file-based auth cache before attempting connection:
- 15-minute TTL
- Servers in `needs-auth` state skip connection attempt
- Cache cleared on successful auth or explicit cache clear

---

## Tool Result Storage

### Binary Content Persistence

Large binary tool results are persisted to disk:
- Saved to Claude config directory
- Referenced by path in tool result
- Format description included for model context
- Size estimate included for truncation decisions

### Output Truncation

`truncateMcpContentIfNeeded()` applies truncation when:
- Content size exceeds configured limits
- Large output instructions appended for model awareness
- Text content truncated with indicator

---

## Additional Features

### MCP Skills

Feature-gated (`MCP_SKILLS`) skill fetching from MCP servers:
- `fetchMcpSkillsForClient()` loads skills associated with connected servers
- Skills appear in `/skills` command, separate from prompts

### MCP Instructions Delta

Feature-gated (`MCP_INSTRUCTIONS_DELTA`) attachment-based delivery of MCP server instructions:
- Carries Chrome tool-search instructions as client-side block
- Avoids prompt cache busting when Chrome connects late

### Computer Use MCP

Feature-gated (`CHICAGO_MCP`) computer use support:
- In-process server to avoid subprocess overhead
- Native module dispatch via wrapper
- Reserved server name

### Claude-in-Chrome MCP

- In-process server via `@ant/claude-for-chrome-mcp`
- Custom tool rendering for UI display
- Reserved server name: `claude-in-chrome`
