# ReadMcpResourceTool

## Purpose

ReadMcpResourceTool is a built-in tool that reads specific resources from MCP servers by URI. It sends a `resources/read` request to the target server, handles both text and binary content types, persists binary blobs to disk with MIME-derived extensions, and returns structured content with URI, MIME type, and either text content or a file path for binary data. The tool enables the model to access data sources exposed by MCP servers as structured resources.

## Location

- `restored-src/src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` — Main tool definition and resource reading (159 lines)
- `restored-src/src/tools/ReadMcpResourceTool/UI.tsx` — Terminal UI rendering (37 lines)
- `restored-src/src/tools/ReadMcpResourceTool/prompt.ts` — Tool description and prompt (17 lines)
- `restored-src/src/services/mcp/client.ts` — Client connection, result transformation (~2800 lines)
- `restored-src/src/utils/mcpOutputStorage.ts` — Binary content persistence (190 lines)

## Key Exports

### From ReadMcpResourceTool.ts

| Export | Description |
|--------|-------------|
| `ReadMcpResourceTool` | The complete tool definition built via `buildTool()` |
| `inputSchema` | Lazy Zod schema: `{ server: string, uri: string }` |
| `outputSchema` | Lazy Zod schema: `{ contents: [{ uri, mimeType?, text?, blobSavedTo? }] }` |
| `Output` | Zod-inferred output type |

### From prompt.ts

| Export | Description |
|--------|-------------|
| `DESCRIPTION` | Tool description for model context |
| `PROMPT` | Detailed usage instructions for the model |

### From UI.tsx

| Export | Description |
|--------|-------------|
| `renderToolUseMessage(input)` | Renders tool invocation message |
| `userFacingName()` | Returns `'readMcpResource'` |
| `renderToolResultMessage(output, _, { verbose })` | Renders resource content as formatted JSON |

### From mcpOutputStorage.ts (binary persistence)

| Export | Description |
|--------|-------------|
| `persistBinaryContent(bytes, mimeType, persistId)` | Writes binary to disk, returns filepath |
| `getBinaryBlobSavedMessage(filepath, mimeType, size, sourceDescription)` | Formats saved message |
| `extensionForMimeType(mimeType)` | Maps MIME type to file extension |
| `isBinaryContentType(contentType)` | Heuristic for binary vs text content |

## Input/Output Schemas

### Input Schema

```typescript
{
  server: string,   // Required: The MCP server name
  uri: string,      // Required: The resource URI to read
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `server` | Yes | Name of the MCP server to read from |
| `uri` | Yes | URI of the specific resource to read |

### Output Schema

```typescript
{
  contents: [
    {
      uri: string,              // Resource URI
      mimeType?: string,        // MIME type of the content
      text?: string,            // Text content (for text resources)
      blobSavedTo?: string,     // File path where binary was saved (for binary resources)
    },
    // ... more content blocks (resources can return multiple contents)
  ],
}
```

## Resource Parsing

### Resource Read Pipeline

```
1. TOOL INVOCATION
   - Model calls ReadMcpResourceTool with { server, uri }
   - Input validated against Zod schema

2. SERVER LOOKUP
   - Find client in mcpClients by server name
   - Throw if server not found (list available servers)
   - Throw if server not connected (type !== 'connected')
   - Throw if server doesn't support resources (!capabilities?.resources)

3. CONNECTION ENSURANCE
   - ensureConnectedClient(client)
   - Reconnects if memo cache was cleared (onclose)
   - Throws if reconnection fails

4. RESOURCE REQUEST
   - connectedClient.client.request({
       method: 'resources/read',
       params: { uri }
     }, ReadResourceResultSchema)
   - Returns ReadResourceResult with contents array

5. CONTENT INTERCEPTION
   - For each content block in result.contents:
     a. Text content ('text' in c):
        - Pass through: { uri, mimeType, text }
     b. Binary content ('blob' in c):
        - Decode base64: Buffer.from(c.blob, 'base64')
        - Generate persistId: mcp-resource-<timestamp>-<index>-<random>
        - Persist to disk: persistBinaryContent(bytes, mimeType, persistId)
        - Replace with: { uri, mimeType, blobSavedTo, text: savedMessage }
     c. Unknown content:
        - Pass through: { uri, mimeType }

6. RESULT RETURN
   - { data: { contents: processedContentArray } }
```

### MCP Protocol: resources/read

```
Request:
{
  jsonrpc: "2.0",
  id: <number>,
  method: "resources/read",
  params: {
    uri: string    // Resource URI to read
  }
}

Response:
{
  jsonrpc: "2.0",
  id: <number>,
  result: {
    contents: [
      {
        uri: string,
        mimeType?: string,
        // Either text or blob:
        text?: string,     // Text content
        // OR
        blob?: string,     // Base64-encoded binary content
      }
    ]
  }
}
```

## Content Extraction

### Text Content Handling

When a resource returns text content, processing is straightforward:

1. Check `'text' in c` returns true
2. Return as-is: `{ uri, mimeType, text }`
3. Text included directly in tool result
4. No file persistence needed

Example:
```
{
  uri: "resource://example/data",
  mimeType: "application/json",
  text: '{"key": "value"}'
}
```

### Binary Content Handling

When a resource returns binary content (blob), the content is intercepted and persisted to disk:

1. Check `'blob' in c` returns true
2. Decode base64: `Buffer.from(c.blob, 'base64')`
3. Generate unique persistId: `mcp-resource-<timestamp>-<index>-<random>`
4. Persist to disk via `persistBinaryContent(bytes, mimeType, persistId)`:
   - Determine extension from MIME type (application/pdf -> .pdf, etc.)
   - Write to tool-results directory: `<tool-results-dir>/<persistId>.<ext>`
   - Return `{ filepath, size, ext }`
5. Build result object with `blobSavedTo` path and human-readable message

Example result:
```
{
  uri: c.uri,
  mimeType: c.mimeType,
  blobSavedTo: persisted.filepath,
  text: "Binary content (application/pdf, 1.2 MB) saved to /path/to/file.pdf"
}
```

Why persist to disk:
- Base64 blobs in context waste tokens
- Native tools can open files by extension (PDF readers, etc.)
- Read tool dispatches on extension for proper handling

### MIME Type to Extension Mapping

The `extensionForMimeType` function maps MIME types to file extensions:

| MIME Type Pattern | Extension |
|-------------------|-----------|
| `application/pdf` | pdf |
| `application/json` | json |
| `text/csv` | csv |
| `text/plain` | txt |
| `text/html` | html |
| `text/markdown` | md |
| `application/zip` | zip |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | docx |
| `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | xlsx |
| `application/vnd.openxmlformats-officedocument.presentationml.presentation` | pptx |
| `application/msword` | doc |
| `application/vnd.ms-excel` | xls |
| `audio/mpeg`, `audio/wav`, `audio/ogg` | mp3, wav, ogg |
| `video/mp4`, `video/webm` | mp4, webm |
| `image/png`, `image/jpeg`, `image/gif`, `image/webp` | png, jpg, gif, webp |
| `image/svg+xml` | svg |
| Anything else | bin |

Extension matters because:
- Read tool dispatches on file extension
- PDFs, images, etc. need correct extension for proper handling
- Unknown types get 'bin' as safe fallback

### Binary Content Detection

The `isBinaryContentType` function classifies content types:

Treated as TEXT (not binary):
- `text/*` (all text types)
- `application/json` and `*/*+json` variants
- `application/xml` and `*/*+xml` variants
- `application/javascript`
- `application/x-www-form-urlencoded`

Treated as BINARY (persist to disk):
- `application/pdf`, `application/zip`, `application/octet-stream`
- `image/*`, `audio/*`, `video/*`
- `application/vnd.*` (Office formats)
- Anything not matching text patterns above

## Error Handling

### Pre-Request Validation

| Error Condition | Error Message |
|----------------|---------------|
| Server not found | `Server "<name>" not found. Available servers: <list>` |
| Server not connected | `Server "<name>" is not connected` |
| Server lacks resources capability | `Server "<name>" does not support resources` |

### Binary Persistence Errors

If `persistBinaryContent` fails, the error is handled gracefully:

```
{
  uri: c.uri,
  mimeType: c.mimeType,
  text: "Binary content could not be saved to disk: <error message>"
}
```

The error is logged but does not fail the entire tool call. Other content blocks in the same response are still processed normally.

### Result Truncation

The `isResultTruncated` function:
- Serializes output via `jsonStringify`
- Checks if output line is truncated via `isOutputLineTruncated`
- Returns true if result was truncated
- `maxResultSizeChars`: 100,000 (results exceeding this may be truncated)

## UI Rendering

### Tool Use Message

```
renderToolUseMessage(input):
  - Missing uri or server -> null
  - Complete: 'Read resource "<uri>" from server "<server>"'
```

### User-Facing Name

```
userFacingName(): 'readMcpResource'
```

### Tool Result Message

```
renderToolResultMessage(output, _, { verbose }):
  - Empty or no contents: '(No content)' in dim color
  - Non-empty: jsonStringify(output, null, 2) via OutputLine
    - Pretty-printed JSON with 2-space indentation
    - Truncation handled by OutputLine component
```

## Configuration

### Tool Properties

| Property | Value | Description |
|----------|-------|-------------|
| `name` | `'ReadMcpResourceTool'` | Internal tool name |
| `userFacingName` | `'readMcpResource'` | Display name in UI |
| `isConcurrencySafe` | `true` | Read-only, safe for concurrent calls |
| `isReadOnly` | `true` | Does not modify server state |
| `shouldDefer` | `true` | Can be deferred in auto-mode |
| `maxResultSizeChars` | `100,000` | Maximum result size before truncation |
| `searchHint` | `'read a specific MCP resource by URI'` | Auto-mode search hint |

### Persist ID Format

```
mcp-resource-<timestamp>-<index>-<random>

Example: mcp-resource-1712345678901-0-a3f7b2

Components:
- timestamp: Date.now() for uniqueness and temporal ordering
- index: Content block index in the response array
- random: 6-char random string (Math.random().toString(36).slice(2, 8))
```

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/mcp/client.ts` | Client connection management, ensureConnectedClient |
| `utils/mcpOutputStorage.ts` | Binary content persistence, MIME mapping |
| `utils/slowOperations.ts` | JSON stringify |
| `utils/terminal.ts` | Output truncation detection |
| `utils/lazySchema.ts` | Lazy Zod schema evaluation |
| `utils/errors.ts` | Error utilities |
| `components/MessageResponse.js` | UI message component |
| `components/shell/OutputLine.js` | Terminal output rendering |
| `ink.js` | Ink terminal UI primitives |

### External

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP types, ReadResourceResultSchema |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |
| `fs/promises` | File system operations (writeFile) |
| `path` | Path joining for file paths |

## Data Flow

```
Model Request
    |
    v
ReadMcpResourceTool Input { server: string, uri: string }
    |
    v
+-----------------------------------------------------------------+
| RESOURCE READING                                                 |
|                                                                 |
|  1. Server validation                                           |
|     - Find client by server name                                |
|     - Check connected status                                    |
|     - Check resources capability                                |
|                                                                 |
|  2. Connection assurance                                        |
|     - ensureConnectedClient()                                   |
|     - Reconnect if needed                                       |
|                                                                 |
|  3. Resource request                                            |
|     - client.request({ method: 'resources/read', params })      |
|     - Schema: ReadResourceResultSchema                          |
|                                                                 |
|  4. Content processing (Promise.all)                            |
|     For each content block:                                     |
|     a. Text content -> pass through                             |
|     b. Binary content ->                                        |
|        - Decode base64                                          |
|        - Persist to disk with MIME-derived extension            |
|        - Replace with filepath + saved message                  |
|     c. Unknown content -> pass through with uri + mimeType      |
|                                                                 |
|  5. Return { data: { contents: processedArray } }               |
+-----------------------------------------------------------------+
    |
    v
Output { contents: [{ uri, mimeType?, text?, blobSavedTo? }, ...] }
    |
    v
+-----------------------------------------------------------------+
| UI RENDERING                                                    |
|                                                                 |
|  renderToolResultMessage:                                       |
|  - Empty: '(No content)'                                        |
|  - Non-empty: Pretty-printed JSON (2-space indent)              |
|                                                                 |
|  Binary content shows:                                          |
|  "Binary content (application/pdf, 1.2 MB) saved to /path/..."  |
+-----------------------------------------------------------------+
    |
    v
Model Response (tool_result block with resource content)
```
