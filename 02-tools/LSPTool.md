# LSPTool

## Purpose

LSPTool provides code intelligence features by interfacing with Language Server Protocol (LSP) servers. It enables Claude to perform operations like go-to-definition, find-references, hover information, document/workspace symbols, call hierarchy analysis, and implementation lookup. The tool communicates with configured LSP servers, handles file opening/closing, filters gitignored results, and formats responses into human-readable output grouped by file. It is only enabled when an LSP server is connected and available for the file type.

## Location

- `restored-src/src/tools/LSPTool/LSPTool.ts` — Main tool definition and LSP request handling (861 lines)
- `restored-src/src/tools/LSPTool/prompt.ts` — Tool description and operation documentation (22 lines)
- `restored-src/src/tools/LSPTool/schemas.ts` — Zod schemas for all LSP operations (216 lines)
- `restored-src/src/tools/LSPTool/formatters.ts` — Result formatting for each operation type (593 lines)
- `restored-src/src/tools/LSPTool/symbolContext.ts` — Symbol extraction at file position for UI context (91 lines)
- `restored-src/src/tools/LSPTool/UI.tsx` — Terminal UI rendering with collapsed/expanded views (228 lines)

## Key Exports

### From LSPTool.ts

| Export | Description |
|--------|-------------|
| `LSPTool` | The complete tool definition built via `buildTool()` |
| `Input` | Zod-inferred input type: `{ operation, filePath, line, character }` |
| `Output` | Zod-inferred output type: `{ operation, result, filePath, resultCount?, fileCount? }` |

### From schemas.ts

| Export | Description |
|--------|-------------|
| `lspToolInputSchema` | Discriminated union schema for all 9 LSP operations |
| `LSPToolInput` | TypeScript type for tool input |
| `isValidLSPOperation(operation)` | Type guard for valid operation names |

### From formatters.ts

| Export | Description |
|--------|-------------|
| `formatGoToDefinitionResult(result, cwd)` | Formats definition locations |
| `formatFindReferencesResult(result, cwd)` | Formats reference locations grouped by file |
| `formatHoverResult(result, cwd)` | Formats hover information |
| `formatDocumentSymbolResult(result, cwd)` | Formats document symbols (hierarchical) |
| `formatWorkspaceSymbolResult(result, cwd)` | Formats workspace symbols grouped by file |
| `formatPrepareCallHierarchyResult(result, cwd)` | Formats call hierarchy items |
| `formatIncomingCallsResult(result, cwd)` | Formats incoming call references |
| `formatOutgoingCallsResult(result, cwd)` | Formats outgoing call references |

### From symbolContext.ts

| Export | Description |
|--------|-------------|
| `getSymbolAtPosition(filePath, line, character)` | Extracts the symbol/word at a specific file position for UI context |

## Input/Output Schemas

### Input Schema

```typescript
{
  operation: 'goToDefinition' | 'findReferences' | 'hover' | 'documentSymbol' |
             'workspaceSymbol' | 'goToImplementation' | 'prepareCallHierarchy' |
             'incomingCalls' | 'outgoingCalls',
  filePath: string,    // Absolute or relative path to the file
  line: number,        // 1-based line number (as shown in editors)
  character: number,   // 1-based character offset (as shown in editors)
}
```

### Output Schema

```typescript
{
  operation: string,       // The LSP operation that was performed
  result: string,          // Formatted result text
  filePath: string,        // The file path the operation was performed on
  resultCount?: number,    // Number of results (definitions, references, symbols)
  fileCount?: number,      // Number of files containing results
}
```

## Supported Operations

| Operation | LSP Method | Description |
|-----------|-----------|-------------|
| `goToDefinition` | `textDocument/definition` | Find where a symbol is defined |
| `findReferences` | `textDocument/references` | Find all references to a symbol |
| `hover` | `textDocument/hover` | Get hover info (docs, type info) |
| `documentSymbol` | `textDocument/documentSymbol` | Get symbols in a document |
| `workspaceSymbol` | `workspace/symbol` | Search symbols across workspace |
| `goToImplementation` | `textDocument/implementation` | Find implementations of interface/abstract method |
| `prepareCallHierarchy` | `textDocument/prepareCallHierarchy` | Get call hierarchy item at position |
| `incomingCalls` | `textDocument/prepareCallHierarchy` → `callHierarchy/incomingCalls` | Find what calls this function |
| `outgoingCalls` | `textDocument/prepareCallHierarchy` → `callHierarchy/outgoingCalls` | Find what this function calls |

## LSP Request Pipeline

### Execution Flow

```
1. INPUT RECEIVED
   - Model returns LSP tool_use with { operation, filePath, line, character }

2. VALIDATION
   - validateInput():
     a. Parse against discriminated union schema
     b. Check file exists and is a regular file
     c. Skip UNC path stats (security: prevent NTLM credential leaks)

3. PERMISSION CHECK
   - checkReadPermissionForTool(): filesystem read permission check

4. LSP INITIALIZATION WAIT
   - If initialization is 'pending', await waitForInitialization()
   - Ensures server is ready before returning "no server available"

5. FILE OPENING
   - If file not already open in LSP server:
     a. Check file size (max 10 MB)
     b. Read file content
     c. Send textDocument/didOpen to LSP server

6. LSP REQUEST
   - Map operation to LSP method and params
   - Convert 1-based line/character to 0-based LSP protocol
   - Send request to LSP server

7. CALL HIERARCHY TWO-STEP (incomingCalls/outgoingCalls)
   - First request returns CallHierarchyItem(s)
   - Second request uses item to get actual calls

8. GITIGNORED FILE FILTERING
   - For location-based results:
     a. Extract URIs from results
     b. Batch-check paths with `git check-ignore` (50 per batch)
     c. Filter out gitignored locations

9. RESULT FORMATTING
   - Format based on operation type
   - Count results and unique files
   - Group by file with relative paths
```

### Position Conversion

User input uses 1-based indexing (editor convention). LSP protocol uses 0-based indexing:

```typescript
const position = {
  line: input.line - 1,         // 1-based → 0-based
  character: input.character -  // 1-based → 0-based
}
```

### Call Hierarchy Two-Step Process

For `incomingCalls` and `outgoingCalls`:

```
Step 1: textDocument/prepareCallHierarchy
  → Returns CallHierarchyItem[] (functions/methods at position)

Step 2: callHierarchy/incomingCalls or callHierarchy/outgoingCalls
  → Uses first item from Step 1
  → Returns CallHierarchyIncomingCall[] or CallHierarchyOutgoingCall[]
```

### Gitignore Filtering

Results are filtered to exclude gitignored files using batched `git check-ignore`:

```
1. Extract unique file paths from result URIs
2. Batch into groups of 50
3. Run: git check-ignore <path1> <path2> ...
4. Collect ignored paths (exit code 0 = at least one ignored)
5. Filter results to exclude ignored paths
```

Batches of 50 balance efficiency with command-line length limits. Timeout is 5 seconds per batch.

## Result Formatting

### Location Formatting

URIs are converted to relative paths when possible:

```
file:///home/user/project/src/file.ts → src/file.ts:42:10
```

Rules:
- Remove `file://` protocol prefix
- Decode percent-encoded characters
- Handle Windows drive letters (`/C:/path` → `C:/path`)
- Use relative path if shorter and doesn't start with `../../`
- Normalize backslashes to forward slashes

### Operation-Specific Formats

| Operation | Format |
|-----------|--------|
| `goToDefinition` | "Defined in file:line:col" or "Found N definitions:\n  file:line:col" |
| `findReferences` | "Found N references across M files:\n\nfile:\n  Line L:C\n  Line L:C" |
| `hover` | Raw hover content (markdown/plain text) |
| `documentSymbol` | Hierarchical tree: "name (Kind) detail - Line N" with indentation |
| `workspaceSymbol` | "Found N symbols in workspace:\n\nfile:\n  name (Kind) - Line N in container" |
| `prepareCallHierarchy` | "Call hierarchy item: name (Kind) - file:line [detail]" |
| `incomingCalls` | "Found N incoming calls:\n\nfile:\n  name (Kind) - Line L [calls at: L:C, L:C]" |
| `outgoingCalls` | "Found N outgoing calls:\n\nfile:\n  name (Kind) - Line L [called from: L:C, L:C]" |

### Empty Result Messages

Each operation provides helpful context when no results are found:

| Operation | Empty Message |
|-----------|--------------|
| `goToDefinition` | "No definition found. This may occur if the cursor is not on a symbol, or if the definition is in an external library not indexed by the LSP server." |
| `findReferences` | "No references found. This may occur if the symbol has no usages, or if the LSP server has not fully indexed the workspace." |
| `hover` | "No hover information available. This may occur if the cursor is not on a symbol, or if the LSP server has not fully indexed the file." |
| `documentSymbol` | "No symbols found in document. This may occur if the file is empty, not supported by the LSP server, or if the server has not fully indexed the file." |
| `workspaceSymbol` | "No symbols found in workspace. This may occur if the workspace is empty, or if the LSP server has not finished indexing the project." |

## Symbol Context Extraction

`getSymbolAtPosition()` extracts the symbol at a cursor position for display in the tool use message:

- Reads only the first 64KB of the file (covers ~1000 lines)
- Uses regex pattern: `/[\w$'!]+|[+\-*/%&|^~<>=]+/g`
- Handles standard identifiers, Rust lifetimes (`'a`), Rust macros (`macro_name!`), and operators
- Truncates symbols to 30 characters
- Falls back to position display (`line:character`) if extraction fails
- Uses synchronous I/O (called from sync React render)

## UI Rendering

The `LSPResultSummary` component provides collapsed/expanded views:

- **Non-verbose**: Shows count summary (e.g., "Found 3 references across 2 files") with Ctrl+O to expand
- **Verbose**: Shows full result content below the summary

Operation-specific labels are mapped:

| Operation | Singular | Plural |
|-----------|----------|--------|
| `goToDefinition` | definition | definitions |
| `findReferences` | reference | references |
| `hover` | hover info | hover info (special: "available") |
| `documentSymbol` | symbol | symbols |
| `workspaceSymbol` | symbol | symbols |
| `goToImplementation` | implementation | implementations |
| `prepareCallHierarchy` | call item | call items |
| `incomingCalls` | caller | callers |
| `outgoingCalls` | callee | callees |

## Error Handling

| Error Condition | Response |
|----------------|----------|
| File does not exist | `{ result: false, message: 'File does not exist: <path>', errorCode: 1 }` |
| Path is not a file | `{ result: false, message: 'Path is not a file: <path>', errorCode: 2 }` |
| Invalid input schema | `{ result: false, message: 'Invalid input: <error>', errorCode: 3 }` |
| File access error | `{ result: false, message: 'Cannot access file: <path>. <error>', errorCode: 4 }` |
| File too large (>10MB) | `{ result: 'File too large for LSP analysis (XMB exceeds 10MB limit)' }` |
| No LSP server available | `{ result: 'No LSP server available for file type: <ext>' }` |
| LSP request failed | `{ result: 'Error performing <operation>: <error>' }` |
| LSP server manager not initialized | `{ result: 'LSP server manager not initialized' }` |

## Configuration

### Limits

| Limit | Value | Description |
|-------|-------|-------------|
| `MAX_LSP_FILE_SIZE_BYTES` | 10 MB | Maximum file size for LSP analysis |
| `maxResultSizeChars` | 100,000 | Maximum result size stored in context |
| Git check-ignore batch size | 50 | Paths per batch for gitignore checking |
| Git check-ignore timeout | 5 seconds | Per-batch timeout |

### Feature Flags

| Condition | Effect |
|-----------|--------|
| `isLspConnected()` returns false | Tool is disabled |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/lsp/manager.js` | LSP server lifecycle and request handling |
| `utils/cwd.js` | Current working directory |
| `utils/path.js` | Path expansion |
| `utils/fsOperations.js` | File system abstraction |
| `utils/permissions/filesystem.js` | Read permission checks |
| `utils/execFileNoThrow.js` | Git check-ignore execution |
| `utils/debug.js` | Debug logging |
| `utils/format.js` | Truncation utilities |
| `utils/messages.js` | Message extraction |
| `utils/file.js` | Display path formatting |

### External

| Package | Purpose |
|---------|---------|
| `vscode-languageserver-types` | LSP type definitions (Location, SymbolInformation, etc.) |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |
| `fs/promises` | File operations |
| `path`, `url` | Path and URI handling |

## Data Flow

```
Model Request
    |
    v
LSPTool Input { operation, filePath, line, character }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                      │
│ - Schema validation (discriminated union)                       │
│ - File exists and is regular file                               │
│ - Skip UNC paths (NTLM security)                                │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  - Filesystem read permission                                   │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ LSP INITIALIZATION                                              │
│  - Wait for initialization if pending                           │
│  - Get LSP server manager                                       │
│  - Open file in LSP if not already open (max 10MB)              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ LSP REQUEST                                                       │
│                                                                 │
│  1. Map operation → LSP method + params                         │
│  2. Convert 1-based → 0-based position                          │
│  3. Send request to LSP server                                  │
│  4. For call hierarchy: two-step process                        │
│  5. Filter gitignored results                                   │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT FORMATTING                                               │
│  - Format based on operation type                               │
│  - Count results and unique files                               │
│  - Group by file with relative paths                            │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { operation, result, filePath, resultCount?, fileCount? }
    |
    v
Model Response (tool_result block)
```
