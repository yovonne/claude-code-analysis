# ListMcpResourcesTool

## Purpose

ListMcpResourcesTool is a built-in tool that discovers and lists available resources from all connected MCP servers. It queries each server's `resources/list` endpoint, aggregates results with server attribution, and returns structured resource metadata including URIs, names, MIME types, and descriptions. The tool is automatically added to the first MCP server that declares resource capability, enabling the model to explore what data sources are available across the MCP ecosystem.

## Location

- `restored-src/src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts` — Main tool definition and resource fetching (124 lines)
- `restored-src/src/tools/ListMcpResourcesTool/UI.tsx` — Terminal UI rendering (29 lines)
- `restored-src/src/tools/ListMcpResourcesTool/prompt.ts` — Tool description and prompt (21 lines)
- `restored-src/src/services/mcp/client.ts` — Resource fetching (`fetchResourcesForClient`), connection management (~2800 lines)
- `restored-src/src/services/mcp/types.ts` — `ServerResource` type definition (259 lines)

## Key Exports

### From ListMcpResourcesTool.ts

| Export | Description |
|--------|-------------|
| `ListMcpResourcesTool` | The complete tool definition built via `buildTool()` |
| `Output` | Zod-inferred output type: array of resource objects |

### From prompt.ts

| Export | Description |
|--------|-------------|
| `LIST_MCP_RESOURCES_TOOL_NAME` | Constant: `'ListMcpResourcesTool'` |
| `DESCRIPTION` | Tool description for model context |
| `PROMPT` | Detailed usage instructions for the model |

### From UI.tsx

| Export | Description |
|--------|-------------|
| `renderToolUseMessage(input)` | Renders tool invocation message |
| `renderToolResultMessage(output, _, { verbose })` | Renders resource list as formatted JSON |

### From client.ts (resource fetching)

| Export | Description |
|--------|-------------|
| `fetchResourcesForClient(client)` | LRU-cached resource discovery via `resources/list` |
| `ServerResource` | Type: `Resource & { server: string }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  server?: string,  // Optional: filter resources to a specific server
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `server` | No | Server name to filter by. If omitted, returns resources from all servers. |

### Output Schema

```typescript
[
  {
    uri: string,           // Resource URI (unique identifier)
    name: string,          // Human-readable resource name
    mimeType?: string,     // MIME type of the resource content
    description?: string,  // Resource description
    server: string,        // Server that provides this resource
  },
  // ... more resources
]
```

## MCP Resource Listing

### Resource Discovery Pipeline

```
1. TOOL INVOCATION
   - Model calls ListMcpResourcesTool with optional { server }
   - Input validated against Zod schema

2. SERVER FILTERING
   - If server specified:
     - Filter mcpClients to matching server name
     - Throw if server not found (list available servers in error)
   - If no server specified:
     - Process all connected mcpClients

3. PARALLEL RESOURCE FETCHING
   - Promise.all() across all target servers:
     a. ensureConnectedClient(client) — verify/reconnect
     b. fetchResourcesForClient(fresh) — query resources/list
   - Individual server failures don't sink the whole result
     (caught and logged, returns empty array for that server)

4. RESULT AGGREGATION
   - results.flat() — merge arrays from all servers
   - Each resource already has server field from fetchResourcesForClient
   - Return as { data: flattenedArray }

5. CACHE BEHAVIOR
   - fetchResourcesForClient is LRU-cached by server name
   - Max 20 entries (MCP_FETCH_CACHE_SIZE)
   - Cache invalidated on:
     a. client.onclose (connection dropped)
     b. resources/list_changed notification from server
     c. clearServerCache() (manual reconnection)
   - Warm from startup prefetch (prefetchAllMcpResources)
```

### fetchResourcesForClient Implementation

```typescript
fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (client.type !== 'connected') return []

    if (!client.capabilities?.resources) return []

    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema,
    )

    if (!result.resources) return []

    // Add server name to each resource
    return result.resources.map(resource => ({
      ...resource,
      server: client.name,
    }))
  },
  (client) => client.name,  // Cache key
  MCP_FETCH_CACHE_SIZE,     // Max 20 entries
)
```

### MCP Protocol: resources/list

```
Request:
{
  jsonrpc: "2.0",
  id: <number>,
  method: "resources/list",
  params: {}
}

Response:
{
  jsonrpc: "2.0",
  id: <number>,
  result: {
    resources: [
      {
        uri: string,           // Unique resource identifier
        name: string,          // Human-readable name
        description?: string,  // Optional description
        mimeType?: string,     // Optional MIME type
      }
    ],
    nextCursor?: string        // Pagination cursor (if more resources)
  }
}
```

## Resource Discovery

### When Resources Are Discovered

```
1. INITIAL CONNECTION
   - connectToServer() establishes connection
   - Server capabilities inspected: capabilities?.resources
   - If resources capability present:
     a. First such server triggers resource tool addition
     b. ListMcpResourcesTool added to tools array
     c. ReadMcpResourceTool added to tools array
     d. resourceToolsAdded flag prevents duplicate addition

2. SERVER RECONNECTION
   - reconnectMcpServerImpl() after auth or config change:
     a. Fetches resources via fetchResourcesForClient
     b. Checks if resource tools already present
     c. Adds them if not (hasResourceTools check)

3. RESOURCE CHANGE NOTIFICATIONS
   - Server sends resources/list_changed notification
   - Cache invalidated: fetchResourcesForClient.cache.delete(name)
   - Next ListMcpResourcesTool call fetches fresh data

4. STARTUP PREFETCH
   - prefetchAllMcpResources() called at startup
   - Warms resource caches for all servers
   - Avoids cold latency on first tool call
```

### Server Capability Detection

```typescript
// From connectToServer():
const capabilities = client.getServerCapabilities()
// capabilities.resources indicates server supports resources

// From getMcpToolsCommandsAndResources():
const supportsResources = !!client.capabilities?.resources
if (supportsResources && !resourceToolsAdded) {
  resourceToolsAdded = true
  resourceTools.push(ListMcpResourcesTool, ReadMcpResourceTool)
}
```

### Resource Tool Addition Logic

```
Only ONE server gets the resource tools (first with resources capability):

1. resourceToolsAdded = false (module-level flag)
2. For each server in connection order:
   a. If server supports resources AND !resourceToolsAdded:
      - Add ListMcpResourcesTool
      - Add ReadMcpResourceTool
      - Set resourceToolsAdded = true
   b. Subsequent servers with resources: skip addition

This ensures:
- No duplicate resource tools in the tool list
- Resource tools are available as long as any server has resources
- Model sees a unified resource interface across all servers
```

## Metadata Retrieval

### Resource Object Structure

Each resource returned by the tool includes:

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `uri` | string | MCP server | Unique resource identifier (used to read the resource) |
| `name` | string | MCP server | Human-readable name |
| `mimeType` | string? | MCP server | MIME type (e.g., `application/json`, `text/plain`) |
| `description` | string? | MCP server | Optional description of the resource |
| `server` | string | Added by client | Server name that provides this resource |

### Empty Result Handling

```
When no resources found:
- mapToolResultToToolResultBlockParam returns:
  "No resources found. MCP servers may still provide tools even if they have no resources."

This message clarifies that:
- Resources and tools are separate MCP capabilities
- A server can have tools without resources
- Absence of resources doesn't indicate a problem
```

## UI Rendering

### Tool Use Message

```
renderToolUseMessage(input):
  - With server: 'List MCP resources from server "<server>"'
  - Without server: 'List all MCP resources'
```

### Tool Result Message

```
renderToolResultMessage(output, _, { verbose }):
  - Empty: '(No resources found)' in dim color
  - Non-empty: jsonStringify(output, null, 2) via OutputLine
    - Pretty-printed JSON with 2-space indentation
    - Truncation handled by OutputLine component
```

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `MCP_FETCH_CACHE_SIZE` | Max LRU cache entries for resource fetching (default 20) |

### Tool Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'ListMcpResourcesTool'` | Internal tool name |
| `userFacingName` | `'listMcpResources'` | Display name in UI |
| `isConcurrencySafe` | `true` | Read-only, safe for concurrent calls |
| `isReadOnly` | `true` | Does not modify server state |
| `shouldDefer` | `true` | Can be deferred in auto-mode |
| `maxResultSizeChars` | `100,000` | Maximum result size before truncation |
| `searchHint` | `'list resources from connected MCP servers'` | Auto-mode search hint |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/mcp/client.ts` | Resource fetching, client connection management |
| `services/mcp/types.ts` | ServerResource type, connection types |
| `utils/errors.ts` | Error message extraction |
| `utils/log.ts` | MCP error logging |
| `utils/slowOperations.ts` | JSON stringify |
| `utils/terminal.ts` | Output truncation detection |
| `utils/lazySchema.ts` | Lazy Zod schema evaluation |
| `components/MessageResponse.js` | UI message component |
| `components/shell/OutputLine.js` | Terminal output rendering |
| `ink.js` | Ink terminal UI primitives |

### External

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP types, ListResourcesResultSchema |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |

## Data Flow

```
Model Request
    |
    v
ListMcpResourcesTool Input { server?: string }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESOURCE DISCOVERY                                               │
│                                                                 │
│  1. Server filtering                                            │
│     - If server specified: filter to matching server            │
│     - If not: use all mcpClients                                │
│                                                                 │
│  2. Parallel fetch (Promise.all)                                │
│     For each server:                                            │
│     a. ensureConnectedClient() — verify connection              │
│     b. fetchResourcesForClient() — LRU-cached                   │
│        - If cached: return immediately                          │
│        - If not: client.request({ method: 'resources/list' })   │
│        - Add server field to each resource                      │
│     c. On error: log and return []                              │
│                                                                 │
│  3. Aggregate results                                           │
│     - results.flat() — merge all server arrays                  │
│     - Return { data: flattenedArray }                           │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output [ { uri, name, mimeType?, description?, server }, ... ]
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ UI RENDERING                                                    │
│                                                                 │
│  renderToolResultMessage:                                       │
│  - Empty: '(No resources found)'                                │
│  - Non-empty: Pretty-printed JSON (2-space indent)              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Model Response (tool_result block with resource list)
```
