# MCPTool

## Purpose

MCPTool is the generic wrapper that represents any tool provided by a connected Model Context Protocol (MCP) server. It acts as a pseudo-tool stub whose actual behavior (name, description, schema, call logic) is overridden at runtime when MCP servers are connected. The stub defines the interface contract; `client.ts` fills in the real implementation for each discovered tool. It handles tool invocation, server communication via the MCP SDK, progress streaming, result transformation, truncation, and binary content persistence.

## Location

- `restored-src/src/tools/MCPTool/MCPTool.ts` — Main tool stub definition (78 lines)
- `restored-src/src/tools/MCPTool/UI.tsx` — Terminal UI rendering for tool use, progress, and results (403 lines)
- `restored-src/src/tools/MCPTool/prompt.ts` — Placeholder prompt/description (overridden at runtime) (4 lines)
- `restored-src/src/tools/MCPTool/classifyForCollapse.ts` — Search/read classification for UI collapsing (605 lines)
- `restored-src/src/services/mcp/client.ts` — Runtime tool registration, call logic, result processing (~2800 lines)
- `restored-src/src/services/mcp/auth.ts` — OAuth authentication, token management, ClaudeAuthProvider (~1700 lines)
- `restored-src/src/services/mcp/types.ts` — Server config schemas, connection types (259 lines)
- `restored-src/src/services/mcp/mcpStringUtils.ts` — Tool name parsing, prefix generation (107 lines)
- `restored-src/src/utils/mcpOutputStorage.ts` — Binary content persistence (190 lines)
- `restored-src/src/utils/mcpValidation.ts` — Content size estimation, truncation logic
- `restored-src/src/utils/mcpWebSocketTransport.ts` — WebSocket transport wrapper
- `restored-src/src/services/mcp/elicitationHandler.ts` — Elicitation request handling
- `restored-src/src/services/mcp/headersHelper.ts` — Dynamic header generation
- `restored-src/src/services/mcp/normalization.ts` — Name normalization utilities
- `restored-src/src/services/mcp/oauthPort.ts` — OAuth redirect port management (79 lines)
- `restored-src/src/utils/mcp/dateTimeParser.ts` — Date/time parsing for MCP results
- `restored-src/src/utils/mcp/elicitationValidation.ts` — Elicitation input validation

## Key Exports

### From MCPTool.ts

| Export | Description |
|--------|-------------|
| `MCPTool` | The complete tool stub definition built via `buildTool()` |
| `inputSchema` | Lazy Zod schema: `z.object({}).passthrough()` — allows any input |
| `outputSchema` | Lazy Zod schema: `z.string()` — MCP tool execution result |
| `Output` | Zod-inferred output type (string) |
| `MCPProgress` | Progress data type (re-exported from `types/tools.js`) |

### From UI.tsx

| Export | Description |
|--------|-------------|
| `renderToolUseMessage(input, { verbose })` | Renders tool invocation arguments |
| `renderToolUseProgressMessage(progressMessages)` | Renders progress with progress bar |
| `renderToolResultMessage(output, _, { verbose, input })` | Renders tool result with rich formatting |
| `tryFlattenJson(content)` | Flattens JSON object to key-value display pairs |
| `tryUnwrapTextPayload(content)` | Extracts dominant text from JSON wrapper |
| `trySlackSendCompact(output, input)` | Detects Slack send results for compact display |

### From classifyForCollapse.ts

| Export | Description |
|--------|-------------|
| `classifyMcpToolForCollapse(serverName, toolName)` | Returns `{ isSearch, isRead }` for UI collapsing |
| `SEARCH_TOOLS` | Set of ~120+ search tool names across MCP servers |
| `READ_TOOLS` | Set of ~400+ read tool names across MCP servers |

### From client.ts (MCP tool execution)

| Export | Description |
|--------|-------------|
| `fetchToolsForClient(client)` | LRU-cached tool discovery via `tools/list` |
| `callMCPTool(params)` | Core tool invocation with session retry |
| `callMCPToolWithUrlElicitationRetry(params)` | Tool call with URL elicitation fallback |
| `transformMCPResult(result, tool, name)` | Normalizes tool result into content/type/schema |
| `processMCPResult(result, tool, name)` | Full result processing with truncation/persistence |
| `McpAuthError` | Error class for authentication failures |
| `McpToolCallError` | Error class for tool execution failures (carries `_meta`) |
| `McpSessionExpiredError` | Error for expired MCP sessions |
| `isMcpSessionExpiredError(error)` | Detects 404 + JSON-RPC -32001 session expiry |
| `mcpToolInputToAutoClassifierInput(input, toolName)` | Encodes input for auto-mode security classifier |

## Input/Output Schemas

### Input Schema

```typescript
// Passthrough — allows any input object since MCP tools define their own schemas
{
  // Whatever the MCP server's tool defines
  [key: string]: unknown
}
```

The actual input schema is set per-tool at runtime by `fetchToolsForClient`, which reads `tool.inputSchema` from the MCP server's `tools/list` response and stores it as `inputJSONSchema`.

### Output Schema

```typescript
{
  data: string | ContentBlockParam[] | MCPToolResult,  // Tool result content
  mcpMeta?: {
    _meta?: Record<string, unknown>,                   // MCP spec _meta
    structuredContent?: unknown,                        // Structured content
  },
}
```

Output can be:
- Plain string (text result)
- Array of `ContentBlockParam` (text, image, resource blocks)
- `MCPToolResult` (union of string or content block array)

## MCP Tool Invocation

### Tool Discovery Pipeline

```
1. SERVER CONNECTION
   - connectToServer() establishes transport (stdio/sse/http/ws)
   - Client negotiates capabilities with server

2. TOOL LISTING
   - fetchToolsForClient() sends tools/list request
   - LRU-cached by server name (max 20 entries)
   - Cache invalidated on onclose and reconnection

3. TOOL REGISTRATION
   - Each MCP tool is wrapped as a Tool object:
     a. name: mcp__<server>__<tool> (fully qualified)
     b. mcpInfo: { serverName, toolName }
     c. isMcp: true
     d. description/prompt: from server metadata (capped at 2048 chars)
     e. inputJSONSchema: from server's tool definition
     f. checkPermissions: returns passthrough (user-configurable)
     g. call: delegates to callMCPToolWithUrlElicitationRetry
     h. isConcurrencySafe: from tool.annotations.readOnlyHint
     i. isReadOnly: from tool.annotations.readOnlyHint
     j. isSearchOrReadCommand: from classifyMcpToolForCollapse

4. RESOURCE TOOLS
   - First server with resources capability gets:
     - ListMcpResourcesTool (list available resources)
     - ReadMcpResourceTool (read specific resource)
   - Added only once across all servers (resourceToolsAdded flag)
```

### Tool Call Execution

```
1. MODEL REQUEST
   - Model calls mcp__server__tool with { args }

2. SESSION ENSURANCE
   - ensureConnectedClient() checks connection health
   - Reconnects if onclose cleared the memo cache
   - Throws if server cannot be connected

3. TOOL INVOCATION
   - callMCPToolWithUrlElicitationRetry() sends:
     { method: "tools/call", params: { name, arguments: args, _meta } }
   - _meta includes claudecode/toolUseId for progress correlation

4. PROGRESS STREAMING
   - onProgress callback fires with MCPProgress:
     { type: 'mcp_progress', status: 'started'|'completed'|'failed', ... }
   - Progress includes serverName, toolName, elapsedTimeMs

5. SESSION RETRY
   - McpSessionExpiredError triggers one automatic retry
   - MAX_SESSION_RETRIES = 1
   - Fresh client obtained via ensureConnectedClient

6. RESULT TRANSFORMATION
   - transformMCPResult() normalizes response:
     a. toolResult → plain text
     b. structuredContent → JSON string + inferred schema
     c. content array → transformed ContentBlockParam[]
   - processMCPResult() handles large output:
     a. Size estimation via getContentSizeEstimate()
     b. If too large: persist to file or truncate
     c. Image content → truncation (not persistence)

7. ERROR HANDLING
   - MCP SDK errors wrapped in TelemetrySafeError
   - McpError code preserved in message (e.g., "McpError -32000")
   - McpToolCallError carries _meta for SDK consumers
```

### Tool Call Code Path

```typescript
async call(args, context, _canUseTool, parentMessage, onProgress) {
  const connectedClient = await ensureConnectedClient(client)
  const mcpResult = await callMCPToolWithUrlElicitationRetry({
    client: connectedClient,
    tool: tool.name,
    args,
    meta: { 'claudecode/toolUseId': toolUseId },
    signal: context.abortController.signal,
    onProgress,
    handleElicitation: context.handleElicitation,
  })
  return {
    data: mcpResult.content,
    mcpMeta: { _meta, structuredContent },
  }
}
```

## Server Communication

### Transport Types

| Transport | Protocol | Auth | Use Case |
|-----------|----------|------|----------|
| stdio | stdin/stdout | None | Local MCP servers (subprocess) |
| sse | SSE + HTTP POST | OAuth (ClaudeAuthProvider) | Remote streaming servers |
| http | Streamable HTTP | OAuth (ClaudeAuthProvider) | Modern MCP servers (2025-03-26 spec) |
| ws | WebSocket | Custom headers | IDE extensions |
| sse-ide | SSE | None (IDE auth token) | IDE extensions |
| ws-ide | WebSocket | X-Claude-Code-Ide-Authorization | IDE extensions |
| claudeai-proxy | Streamable HTTP via proxy | Claude.ai OAuth token | Claude.ai connector servers |
| sdk | In-process | None | SDK-embedded servers |

### Connection Lifecycle

```
1. INITIALIZATION
   - Transport created based on server config type
   - Auth provider attached for sse/http (ClaudeAuthProvider)
   - Headers merged: User-Agent + static + dynamic + session ingress

2. CONNECTION
   - client.connect(transport) with timeout (MCP_TIMEOUT env, default 30s)
   - SetCapabilities: roots, elicitation
   - Request handlers: ListRoots, ElicitRequest (cancel default)

3. HEALTH MONITORING
   - onerror: tracks consecutive terminal errors (max 3)
   - Detects: ECONNRESET, ETIMEDOUT, EPIPE, EHOSTUNREACH, ECONNREFUSED
   - Session expiry: HTTP 404 + JSON-RPC code -32001
   - SSE reconnection exhaustion: "Maximum reconnection attempts"

4. RECONNECTION
   - onclose clears memoization caches:
     - connectToServer.cache
     - fetchToolsForClient.cache
     - fetchResourcesForClient.cache
     - fetchCommandsForClient.cache
   - Next ensureConnectedClient() triggers fresh connection

5. CLEANUP
   - stdio: SIGINT → 100ms → SIGTERM → 400ms → SIGKILL (500ms total)
   - In-process: server.close() then client.close()
   - Registered via registerCleanup() for process shutdown
```

### Fetch Wrapping Stack

For HTTP/SSE transports, fetch calls pass through layers:

```
fetch()
  → wrapFetchWithTimeout()          // 60s timeout per request, Accept header
    → wrapFetchWithStepUpDetection() // 403 insufficient_scope → markStepUpPending
      → createFetchWithInit()       // Base fetch with init merge
```

- **wrapFetchWithTimeout**: Fresh timeout signal per request (not stale), guarantees `Accept: application/json, text/event-stream` for Streamable HTTP
- **wrapFetchWithStepUpDetection**: Detects 403 `insufficient_scope` in WWW-Authenticate header, extracts scope, calls `provider.markStepUpPending(scope)` to trigger step-up OAuth flow
- GET requests skip timeout (long-lived SSE streams)

### Batched Server Connections

```
Local servers (stdio/sdk): concurrency = getMcpServerConnectionBatchSize() (default 3)
Remote servers (sse/http/ws/claudeai-proxy): concurrency = getRemoteMcpServerConnectionBatchSize() (default 20)

Uses pMap for slot-based scheduling (not rigid batches):
- Each slot frees when its server completes
- Slow server occupies one slot, doesn't block entire batch
```

### Auth Caching

```
MCP_AUTH_CACHE_TTL_MS = 15 minutes
Cache file: mcp-needs-auth-cache.json

On 401 during connect:
1. handleRemoteAuthFailure() emits tengu_mcp_server_needs_auth
2. setMcpAuthCacheEntry() writes timestamp
3. Server marked as needs-auth, auth tool created

On reconnect:
- isMcpAuthCached() checks TTL
- hasMcpDiscoveryButNoToken() checks secure storage
- Skips connection if both conditions met (prevents 401 loop)
```

## Tool Result Handling

### Result Transformation

```typescript
transformMCPResult(result, tool, name) → { content, type, schema }

Input shapes:
  { toolResult: string }           → type: 'toolResult'
  { structuredContent: object }    → type: 'structuredContent', schema: inferred
  { content: ContentBlock[] }      → type: 'contentArray', schema: inferred

Schema inference (inferCompactSchema):
  - Depth-limited (depth=2)
  - Objects: {key: type, ...} (max 10 keys)
  - Arrays: [elementType]
  - Primitives: typeof value
```

### Large Output Handling

```
1. Size estimation: getContentSizeEstimate(content) → token count
2. If exceeds threshold:
   a. ENABLE_MCP_LARGE_OUTPUT_FILES disabled → truncate inline
   b. Contains images → truncate (preserve compression)
   c. Otherwise → persist to file:
      - persistToolResult() writes JSON to tool-results directory
      - Returns <persisted-output> with path and instructions
      - Instructions include: format description, jq usage, chunk reading

3. Truncation:
   - truncateMcpContentIfNeeded() with EndTruncatingAccumulator
   - Preserves content structure where possible
```

### Binary Content Persistence

```
For blob content in tool results:
1. persistBinaryContent(bytes, mimeType, persistId)
   - Derives extension from MIME type (pdf, json, png, etc.)
   - Writes to tool-results directory
   - Returns { filepath, size, ext }

2. getBinaryBlobSavedMessage(filepath, mimeType, size, sourceDescription)
   - Returns: "Binary content (mime, size) saved to filepath"

3. Image content (jpeg, png, gif, webp):
   - Resized via maybeResizeAndDownsampleImageBuffer()
   - Max dimensions enforced by API limits
   - Returned as base64 image blocks
```

### UI Rendering Strategies

```
renderToolResultMessage uses three strategies for text content:

1. Text payload unwrapping (tryUnwrapTextPayload):
   - Detects JSON with one dominant string field (e.g., {"messages": "..."})
   - Unwraps body, shows extras as dim metadata
   - Threshold: string > 200 chars or contains newlines + > 50 chars

2. JSON flattening (tryFlattenJson):
   - Small flat objects (≤ 12 keys, ≤ 5000 chars)
   - Renders as aligned key: value pairs
   - Nested objects → one-line JSON (≤ 120 chars)

3. Fallback (OutputLine):
   - Pretty-printed JSON with truncation
   - URL linkification

Special cases:
- Slack send results → compact "Sent message to #channel" display
- Large responses (> 10,000 tokens) → warning banner
- Image blocks → [Image] placeholder
```

### Progress Display

```
renderToolUseProgressMessage(progressMessages):
  - No progress data → "Running…"
  - Progress with total → ProgressBar + percentage
  - Progress without total → "Processing… N" or custom message

Progress events:
  { type: 'mcp_progress', status: 'started'|'completed'|'failed',
    serverName, toolName, elapsedTimeMs }
```

## Tool Classification for UI Collapsing

### classifyMcpToolForCollapse

```
Input: serverName, toolName
Output: { isSearch: boolean, isRead: boolean }

Normalization:
  - camelCase → snake_case: searchCode → search_code
  - kebab-case → snake_case: search-files → search_files
  - Lowercased

SEARCH_TOOLS (~120 entries):
  - Slack: slack_search_public, slack_search_channels, ...
  - GitHub: search_code, search_repositories, search_issues, ...
  - Datadog: search_logs, search_spans, find_slow_spans, ...
  - Sentry: search_docs, search_events, find_organizations, ...
  - Brave: brave_web_search, brave_local_search
  - Exa: web_search_exa, deep_search_exa, ...
  - And many more (Notion, Asana, MongoDB, Airtable, etc.)

READ_TOOLS (~400 entries):
  - GitHub: get_me, get_file_contents, list_branches, ...
  - Linear: list_projects, get_team, list_users, ...
  - Datadog: query_metrics, list_monitors, get_dashboard, ...
  - Filesystem: read_file, list_directory, get_file_info, ...
  - Git: git_status, git_diff, git_log, git_show, ...
  - Kubernetes: kubectl_get, kubectl_describe, pods_list, ...
  - And many more (PagerDuty, Supabase, Stripe, etc.)

Unknown tool names → { isSearch: false, isRead: false } (conservative)
```

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `MCP_TOOL_TIMEOUT` | Tool call timeout in ms (default ~27.8 hours) |
| `MCP_TIMEOUT` | Connection timeout in ms (default 30000) |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | Local server concurrency (default 3) |
| `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` | Remote server concurrency (default 20) |
| `MCP_OAUTH_CALLBACK_PORT` | Fixed OAuth redirect port |
| `CLAUDE_CODE_ENABLE_XAA` | Enable Cross-App Access authentication |
| `ENABLE_MCP_LARGE_OUTPUT_FILES` | Enable file persistence for large results |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | Skip mcp__ prefix for SDK tools |

### Feature Flags

| Flag | Purpose |
|------|---------|
| `MCP_RICH_OUTPUT` | Enables rich UI rendering (text unwrapping, JSON flattening) |
| `MCP_SKILLS` | Enables skill discovery from MCP resources |
| `CHICAGO_MCP` | Computer Use MCP server support |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/mcp/client.ts` | Core MCP client, tool call, connection management |
| `services/mcp/auth.ts` | OAuth flows, ClaudeAuthProvider, token management |
| `services/mcp/types.ts` | Server config schemas, connection types |
| `services/mcp/mcpStringUtils.ts` | Tool name building, prefix generation |
| `services/mcp/normalization.ts` | Name normalization for MCP |
| `utils/mcpOutputStorage.ts` | Binary content persistence |
| `utils/mcpValidation.ts` | Content size estimation, truncation |
| `utils/mcpWebSocketTransport.ts` | WebSocket transport wrapper |
| `utils/toolResultStorage.ts` | Large result persistence to disk |
| `utils/imageResizer.ts` | Image resizing for API limits |
| `utils/errors.ts` | Error handling utilities |
| `utils/log.ts` | MCP debug/error logging |
| `utils/slowOperations.ts` | JSON parse/stringify with limits |
| `utils/memoize.ts` | LRU memoization for fetch caches |
| `utils/sanitization.ts` | Unicode sanitization for server data |
| `utils/proxy.ts` | HTTP/WebSocket proxy support |
| `utils/mtls.ts` | mTLS WebSocket options |
| `utils/subprocessEnv.ts` | Environment for stdio subprocesses |
| `utils/abortController.ts` | Abort controller creation |
| `utils/cleanupRegistry.ts` | Process shutdown cleanup |
| `services/analytics/` | Event logging |

### External

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP protocol implementation |
| `@anthropic-ai/sdk` | API types (ContentBlockParam, etc.) |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |
| `lodash-es` | reject, mapValues, memoize, zipObject |
| `p-map` | Concurrent processing with concurrency limit |
| `figures` | Terminal icons (warning symbol) |
| `axios` | HTTP client for token revocation |
| `ws` | WebSocket client (Node.js) |

## Data Flow

```
Model Request
    |
    v
MCPTool Input { [args from server schema] }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ TOOL CALL EXECUTION                                             │
│                                                                 │
│  1. ensureConnectedClient()                                     │
│     - Check memo cache                                          │
│     - Reconnect if needed                                       │
│                                                                 │
│  2. callMCPToolWithUrlElicitationRetry()                        │
│     - Send tools/call request                                   │
│     - Handle elicitation (URL confirmation)                     │
│     - Stream progress via onProgress                            │
│                                                                 │
│  3. Session retry (max 1)                                       │
│     - McpSessionExpiredError → fresh client → retry             │
│                                                                 │
│  4. transformMCPResult()                                        │
│     - toolResult → string                                       │
│     - structuredContent → JSON + schema                         │
│     - content array → ContentBlockParam[]                       │
│                                                                 │
│  5. processMCPResult()                                          │
│     - Size estimation                                           │
│     - Large output → persist or truncate                        │
│     - Image content → special handling                          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { data: content, mcpMeta?: { _meta, structuredContent } }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ UI RENDERING                                                    │
│                                                                 │
│  renderToolResultMessage:                                       │
│  - Slack compact → "Sent to #channel"                           │
│  - Large warning → "~N tokens" banner                           │
│  - Text unwrap → body + extras                                  │
│  - JSON flatten → aligned key: value                            │
│  - Fallback → pretty-printed JSON                               │
│                                                                 │
│  renderToolUseProgressMessage:                                  │
│  - Progress bar with percentage                                 │
│  - "Running…" / "Processing…" fallback                          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Model Response (tool_result block)
```
