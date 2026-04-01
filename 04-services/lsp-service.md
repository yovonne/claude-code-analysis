# LSP Service

## Purpose

The Language Server Protocol (LSP) service enables Claude Code to interface with external language servers for code intelligence. It manages the full lifecycle of LSP server processes — discovery, spawning, initialization, request routing, and graceful shutdown — while exposing language features (go-to-definition, hover, diagnostics, references, call hierarchy) through the LSPTool. Servers are discovered exclusively from plugins, not user or project settings.

## Location

| File | Lines | Role |
|------|-------|------|
| `restored-src/src/services/lsp/LSPClient.ts` | 448 | Low-level JSON-RPC client wrapping vscode-jsonrpc |
| `restored-src/src/services/lsp/LSPServerInstance.ts` | 512 | Single server lifecycle with state machine and retry |
| `restored-src/src/services/lsp/LSPServerManager.ts` | 421 | Multi-server manager routing by file extension |
| `restored-src/src/services/lsp/manager.ts` | 290 | Singleton manager facade with async init |
| `restored-src/src/services/lsp/config.ts` | 80 | Server configuration loader from plugins |
| `restored-src/src/services/lsp/LSPDiagnosticRegistry.ts` | 387 | Async diagnostic notification registry |
| `restored-src/src/services/lsp/passiveFeedback.ts` | 329 | Diagnostic handler registration and formatting |

## Key Exports

### LSPClient.ts

| Export | Description |
|--------|-------------|
| `LSPClient` (type) | Interface: `start`, `initialize`, `sendRequest`, `sendNotification`, `onNotification`, `onRequest`, `stop`, `capabilities`, `isInitialized` |
| `createLSPClient(serverName, onCrash?)` | Factory that spawns a child process and creates a `MessageConnection` over stdio |

### LSPServerInstance.ts

| Export | Description |
|--------|-------------|
| `LSPServerInstance` (type) | Interface: `name`, `config`, `state`, `startTime`, `lastError`, `restartCount`, `start()`, `stop()`, `restart()`, `isHealthy()`, `sendRequest()`, `sendNotification()`, `onNotification()`, `onRequest()` |
| `createLSPServerInstance(name, config)` | Factory wrapping `LSPClient` with state tracking, crash recovery, and transient-error retry |

### LSPServerManager.ts

| Export | Description |
|--------|-------------|
| `LSPServerManager` (type) | Interface: `initialize()`, `shutdown()`, `getServerForFile()`, `ensureServerStarted()`, `sendRequest()`, `getAllServers()`, `openFile()`, `changeFile()`, `saveFile()`, `closeFile()`, `isFileOpen()` |
| `createLSPServerManager()` | Factory managing multiple server instances with extension-based routing |

### manager.ts

| Export | Description |
|--------|-------------|
| `getLspServerManager()` | Returns the singleton manager instance (or `undefined` if not initialized/failed) |
| `getInitializationStatus()` | Returns `{ status: 'not-started' | 'pending' | 'success' | 'failed', error? }` |
| `isLspConnected()` | Returns `true` if at least one server is not in error state |
| `waitForInitialization()` | Awaits async initialization completion |
| `initializeLspServerManager()` | Creates singleton and starts async init (called at startup) |
| `reinitializeLspServerManager()` | Forces re-init after plugin refresh |
| `shutdownLspServerManager()` | Stops all servers and clears state |

### LSPDiagnosticRegistry.ts

| Export | Description |
|--------|-------------|
| `PendingLSPDiagnostic` (type) | `{ serverName, files, timestamp, attachmentSent }` |
| `registerPendingLSPDiagnostic({ serverName, files })` | Stores diagnostics for async delivery |
| `checkForLSPDiagnostics()` | Retrieves and deduplicates pending diagnostics |
| `clearAllLSPDiagnostics()` | Clears pending diagnostics |
| `resetAllLSPDiagnosticState()` | Clears both pending and delivered tracking |
| `clearDeliveredDiagnosticsForFile(fileUri)` | Clears cross-turn dedup for a specific file |
| `getPendingLSPDiagnosticCount()` | Returns count of pending diagnostics |

### passiveFeedback.ts

| Export | Description |
|--------|-------------|
| `formatDiagnosticsForAttachment(params)` | Converts `PublishDiagnosticsParams` to `DiagnosticFile[]` |
| `registerLSPNotificationHandlers(manager)` | Registers `textDocument/publishDiagnostics` handlers on all servers |
| `HandlerRegistrationResult` (type) | `{ totalServers, successCount, registrationErrors, diagnosticFailures }` |

### config.ts

| Export | Description |
|--------|-------------|
| `getAllLspServers()` | Loads all server configs from enabled plugins in parallel |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/plugins/lspPluginIntegration.ts` | Plugin-scoped server loading and env var resolution |
| `utils/plugins/pluginLoader.ts` | Plugin discovery |
| `utils/debug.ts` | Debug logging |
| `utils/log.ts` | Error logging |
| `utils/errors.ts` | Error formatting |
| `utils/subprocessEnv.ts` | Environment for spawned processes |
| `utils/cwd.ts` | Current working directory |
| `utils/sleep.ts` | Delay utility for retry backoff |
| `utils/attachments.ts` | Diagnostic attachment formatting |
| `services/diagnosticTracking.ts` | `DiagnosticFile` type |

### External

| Package | Purpose |
|---------|---------|
| `vscode-jsonrpc` | JSON-RPC message connection over stdio streams |
| `vscode-languageserver-protocol` | LSP protocol types (`InitializeParams`, `ServerCapabilities`, etc.) |
| `child_process` | Spawning LSP server processes |
| `lru-cache` | Bounded deduplication cache for delivered diagnostics |
| `path`, `url` | File path and URI conversion |

## Implementation Details

### Architecture Overview

The LSP service uses a three-layer factory pattern with closures (no classes):

```
┌─────────────────────────────────────────────────┐
│  manager.ts (Singleton Facade)                  │
│  - initializeLspServerManager()                 │
│  - getLspServerManager()                        │
│  - Async init with generation counter           │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LSPServerManager (Multi-Server Router)         │
│  - Extension → server mapping                   │
│  - File open/close/change/save sync             │
│  - Request routing by file path                 │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LSPServerInstance (Per-Server Lifecycle)       │
│  - State machine: stopped→starting→running      │
│  - Crash recovery with max restart cap          │
│  - Transient error retry (ContentModified)      │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  LSPClient (JSON-RPC Transport)                 │
│  - Child process spawn                          │
│  - StreamMessageReader/Writer over stdio        │
│  - Request/notification dispatch                │
└─────────────────────────────────────────────────┘
```

### Server Lifecycle Management

#### State Machine

Each `LSPServerInstance` follows a strict state machine:

```
stopped ──start()──▶ starting ──init ok──▶ running
  ▲                    │                      │
  │                    │ error                │ error
  │                    ▼                      ▼
  │                  error ◀──── restart() ── error
  │                    │                      │
  │                    │ stop()               │ stop()
  │                    ▼                      ▼
  └────────────── stopped ◀──── stopping ◀── running
```

States: `stopped`, `starting`, `running`, `stopping`, `error`

#### Initialization Flow

1. **manager.ts** creates `LSPServerManager` instance synchronously, marks state as `pending`
2. Async `manager.initialize()` calls `getAllLspServers()` to load plugin configs
3. For each server config, creates `LSPServerInstance` and builds extension map
4. Registers `workspace/configuration` request handler (returns `null` for each item)
5. On success, `registerLSPNotificationHandlers()` attaches diagnostic listeners
6. State transitions to `success`; manager instance is available

#### Startup Sequence (per server)

1. `client.start(command, args, options)` — spawns child process with piped stdio
2. Waits for `spawn` event (critical: `spawn()` returns before `error` fires asynchronously)
3. Creates `MessageConnection` from `StreamMessageReader(process.stdout)` and `StreamMessageWriter(process.stdin)`
4. Registers error/close handlers BEFORE `connection.listen()` to prevent unhandled rejections
5. Enables verbose protocol tracing for debugging
6. Applies any queued notification/request handlers (lazy init support)
7. `client.initialize(params)` — sends `initialize` request with workspace info, client capabilities
8. Sends `initialized` notification
9. State transitions to `running`

#### Shutdown Sequence

1. `isStopping = true` to suppress error logging during intentional shutdown
2. Sends `shutdown` request, then `exit` notification
3. Disposes connection, removes all process event listeners
4. Kills process (catches errors — process may already be dead)
5. Resets `isInitialized`, `capabilities`, `isStopping`

#### Crash Recovery

- `onCrash` callback sets state to `error` and increments `crashRecoveryCount`
- `ensureServerStarted()` detects `error` state and restarts automatically
- Max crash recovery capped at `config.maxRestarts` (default: 3)
- Prevents unbounded child process spawning from persistently crashing servers

### Language Feature Integration

The LSP service supports these language features through `LSPTool`:

| Feature | LSP Method | Client Capability Declared |
|---------|-----------|--------------------------|
| Go to Definition | `textDocument/definition` | `linkSupport: true` |
| Find References | `textDocument/references` | `dynamicRegistration: false` |
| Hover | `textDocument/hover` | `contentFormat: ['markdown', 'plaintext']` |
| Document Symbols | `textDocument/documentSymbol` | `hierarchicalDocumentSymbolSupport: true` |
| Workspace Symbols | `workspace/symbol` | — |
| Go to Implementation | `textDocument/implementation` | — |
| Call Hierarchy | `textDocument/prepareCallHierarchy` → `callHierarchy/incomingCalls` / `callHierarchy/outgoingCalls` | `dynamicRegistration: false` |
| Diagnostics | `textDocument/publishDiagnostics` (notification) | `relatedInformation`, `tagSupport`, `codeDescriptionSupport` |

#### Client Capabilities

Declared during initialization:

```typescript
capabilities: {
  workspace: {
    configuration: false,      // Don't claim support (prevents server requests)
    workspaceFolders: false,   // Don't claim support (not handled)
  },
  textDocument: {
    synchronization: { willSave: false, willSaveWaitUntil: false, didSave: true },
    publishDiagnostics: { relatedInformation: true, tagSupport: { valueSet: [1, 2] } },
    hover: { contentFormat: ['markdown', 'plaintext'] },
    definition: { linkSupport: true },
    documentSymbol: { hierarchicalDocumentSymbolSupport: true },
  },
  general: { positionEncodings: ['utf-16'] },
}
```

### LSPTool Integration

The `LSPTool` (documented separately in `analysis/02-tools/LSPTool.md`) integrates with the LSP service through:

1. **Server Discovery**: `getServerForFile(filePath)` routes by file extension
2. **File Synchronization**: `openFile()`, `changeFile()`, `saveFile()` send `didOpen`, `didChange`, `didSave` notifications
3. **Request Routing**: `sendRequest(filePath, method, params)` auto-starts server if needed
4. **Diagnostic Delivery**: `LSPDiagnosticRegistry` stores async diagnostics for attachment system
5. **Availability Check**: `isLspConnected()` gates tool enablement

### Server Discovery and Configuration

#### Plugin-Based Discovery

LSP servers are discovered exclusively from plugins through two mechanisms:

1. **`.lsp.json` file** in plugin directory — JSON object mapping server names to configs
2. **`manifest.lspServers`** field — can be a string (path to JSON), inline config object, or array of either

#### Configuration Resolution

```
Plugin Load
    │
    ├── .lsp.json file (optional)
    ├── manifest.lspServers (optional)
    │
    ▼
resolvePluginLspEnvironment()
    ├── ${CLAUDE_PLUGIN_ROOT} substitution
    ├── ${user_config.X} substitution
    ├── ${VAR} general env expansion
    └── CLAUDE_PLUGIN_ROOT / CLAUDE_PLUGIN_DATA injected into env
    │
    ▼
addPluginScopeToLspServers()
    └── Server names prefixed: "plugin:${pluginName}:${serverName}"
    │
    ▼
getAllLspServers() — merges all plugin results
```

#### ScopedLspServerConfig Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | `string` | Yes | Executable to spawn |
| `args` | `string[]` | No | Command arguments |
| `env` | `Record<string, string>` | No | Environment variables |
| `cwd` | `string` | No | Working directory |
| `workspaceFolder` | `string` | No | Workspace root for LSP |
| `extensionToLanguage` | `Record<string, string>` | Yes | File extension → language ID mapping |
| `initializationOptions` | `object` | No | Server-specific init options (e.g., vue-language-server) |
| `startupTimeout` | `number` | No | Milliseconds before init timeout |
| `maxRestarts` | `number` | No | Max restart attempts (default: 3) |
| `scope` | `'dynamic'` | Auto | Set by plugin scoping |
| `source` | `string` | Auto | Plugin name |

### Diagnostics System

#### Flow

```
LSP Server sends publishDiagnostics
    │
    ▼
passiveFeedback.ts handler
    ├── formatDiagnosticsForAttachment() — converts LSP → Claude format
    └── registerPendingLSPDiagnostic() — stores in registry
    │
    ▼
LSPDiagnosticRegistry
    ├── Deduplication (within-batch + cross-turn via LRU cache)
    ├── Volume limiting (max 10/file, 30 total)
    └── Severity sorting (Error > Warning > Info > Hint)
    │
    ▼
checkForLSPDiagnostics() — called by attachment system
    └── Returns deduplicated, limited diagnostics as Attachment[]
```

#### Deduplication

- **Within-batch**: Groups by file URI, deduplicates by content hash (message + severity + range + source + code)
- **Cross-turn**: `LRUCache<string, Set<string>>` tracks delivered diagnostics (max 500 files), cleared per-file on edit

#### Volume Limits

| Limit | Value |
|-------|-------|
| Max diagnostics per file | 10 |
| Max total diagnostics | 30 |
| Max tracked files (LRU) | 500 |

### Request Retry Logic

Transient `ContentModified` errors (code `-32801`) are retried with exponential backoff:

- Max retries: 3
- Base delay: 500ms
- Delays: 500ms, 1000ms, 2000ms
- Common with rust-analyzer during project indexing

### File Synchronization

The manager tracks opened files via `openedFiles: Map<URI, serverName>`:

| Method | LSP Notification | Behavior |
|--------|-----------------|----------|
| `openFile(path, content)` | `textDocument/didOpen` | Skips if already open; gets languageId from config |
| `changeFile(path, content)` | `textDocument/didChange` | Falls back to `openFile` if not yet opened |
| `saveFile(path)` | `textDocument/didSave` | Triggers server-side diagnostics |
| `closeFile(path)` | `textDocument/didClose` | Removes from tracking (TODO: integrate with compact) |

### Singleton Initialization

`manager.ts` uses a generation counter pattern to prevent stale promise races:

```typescript
let initializationGeneration = 0

initializeLspServerManager():
  currentGeneration = ++initializationGeneration
  initializationPromise = manager.initialize().then(() => {
    if (currentGeneration === initializationGeneration) {
      initializationState = 'success'
    }
  })
```

Re-initialization (after `/reload-plugins`) increments the generation, invalidating in-flight promises.

### Error Handling

| Scenario | Behavior |
|----------|----------|
| Server spawn fails | Sets `startFailed = true`, throws on next operation |
| Server crashes (non-zero exit) | Calls `onCrash`, sets state to `error`, auto-restarts on next use |
| Connection error during operation | Logged, state set to `error` |
| Initialization timeout | Process killed, state set to `error` |
| Request fails | Error wrapped with context, re-thrown |
| Notification fails | Logged but not re-thrown (fire-and-forget) |
| Shutdown fails | Cleanup proceeds, error logged but not propagated |

### Configuration

#### Environment Variables

| Variable | Effect |
|----------|--------|
| `--bare` mode | LSP disabled entirely (no editor integration needed) |

#### Feature Flags

| Flag | Effect |
|------|--------|
| `isLspConnected()` returns false | LSPTool is disabled |

### Data Flow

```
Claude Code Startup
    │
    ▼
initializeLspServerManager()
    ├── Create LSPServerManager instance
    ├── Async: getAllLspServers() from plugins
    ├── For each server: createLSPServerInstance()
    └── On success: registerLSPNotificationHandlers()
    │
    ▼
LSPTool Request (e.g., goToDefinition)
    ├── getServerForFile(filePath) — find server by extension
    ├── ensureServerStarted(filePath) — lazy start if needed
    ├── openFile(filePath, content) — sync file to server
    └── sendRequest(method, params) — execute LSP request
    │
    ▼
Passive Diagnostics (async)
    ├── LSP server → publishDiagnostics notification
    ├── formatDiagnosticsForAttachment() → registerPendingLSPDiagnostic()
    └── checkForLSPDiagnostics() → delivered as attachments
    │
    ▼
Shutdown
    └── shutdownLspServerManager() → stop all servers
```

## Notes

- LSP is disabled in `--bare` mode (scripted `-p` calls don't need editor integration)
- `vscode-jsonrpc` (~129KB) is lazy-required only when a server is instantiated
- `closeFile()` exists but is not yet integrated with the compact flow (marked TODO)
- `restartOnCrash` and `shutdownTimeout` config fields are defined but throw if used (not yet implemented)
- Servers are lazy-started — they only spawn when first accessed via `ensureServerStarted()`
- The `workspace/configuration` request handler returns `null` for all items, satisfying the protocol without providing actual configuration
