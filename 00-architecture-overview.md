# Architecture Overview

## System Summary

Claude Code CLI is a terminal-based AI coding assistant that provides an interactive REPL (Read-Eval-Print Loop) interface powered by Anthropic's Claude API. The application combines a rich terminal UI (built with Ink/React) with a sophisticated tool-use system that allows the AI model to read, edit, and execute code in the user's environment.

The system operates in two primary modes:
- **Interactive mode**: Full TUI with real-time streaming, permission prompts, task panels, and visual feedback
- **Non-interactive/SDK mode** (`-p/--print`): Headless mode for programmatic access via stdin/stdout with structured JSON streaming

Key capabilities include:
- File operations (read, edit, write) with safety controls
- Shell command execution with sandboxing and permission management
- Sub-agent spawning for parallel task execution (Agent swarms/teammates)
- MCP (Model Context Protocol) server integration for extensible tooling
- Skills system for domain-specific capabilities
- Plugin system for third-party extensions
- Session persistence and resumption
- Context management (compaction, collapse, snipping) for long conversations
- Remote control via bridge (claude.ai web interface integration)
- Multiple authentication providers (OAuth, API key, Bedrock, Vertex)

## Entry Points

The codebase has multiple entry points, each serving a distinct deployment scenario:

### Primary Entry Points

| Entry Point | File | Purpose |
|---|---|---|
| **CLI Bootstrap** | `src/entrypoints/cli.tsx` | Main CLI entry point. Handles fast-path dispatch for special commands (`--version`, `daemon`, `bridge`, `bg`, `templates`, etc.) before falling through to the full CLI. All imports are dynamic to minimize module evaluation. |
| **MCP Server** | `src/entrypoints/mcp.ts` | Runs Claude Code as an MCP server for integration with other MCP clients. |
| **SDK Types** | `src/entrypoints/agentSdkTypes.ts` | Type definitions and stub functions for the Agent SDK public API. The actual SDK implementation lives in separate files; this file provides the type surface. |

### Fast-Path Dispatch (cli.tsx)

The CLI bootstrap implements a fast-path routing pattern that avoids loading the full application for simple commands:

- `--version/-v/-V`: Zero-import version output
- `--dump-system-prompt`: System prompt extraction (ant-only)
- `--claude-in-chrome-mcp`: Chrome integration MCP server
- `--daemon-worker=<kind>`: Daemon worker process (internal)
- `remote-control/rc/remote/sync/bridge`: Bridge mode for remote control
- `daemon`: Long-running supervisor process
- `ps/logs/attach/kill/--bg`: Background session management
- `new/list/reply`: Template job commands
- `environment-runner`: BYOC (Bring Your Own Compute) runner
- `self-hosted-runner`: Self-hosted runner targeting API
- `--worktree --tmux`: Fast tmux worktree exec

### Secondary Entrypoints

| File | Purpose |
|---|---|
| `src/entrypoints/init.ts` | Initialization logic: config validation, environment setup, telemetry, graceful shutdown registration. Memoized to prevent double-init. |

## Core Architecture Layers

The architecture follows a layered design with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│  Entry Points (cli.tsx, mcp.ts)                                 │
│  Fast-path dispatch, dynamic imports                            │
├─────────────────────────────────────────────────────────────────┤
│  CLI Layer (main.tsx)                                           │
│  Commander.js CLI parsing, option processing, mode detection    │
├─────────────────────────────────────────────────────────────────┤
│  Setup Layer (setup.ts)                                         │
│  Environment setup, worktree, hooks, terminal backup, UDS       │
├─────────────────────────────────────────────────────────────────┤
│  Interactive Layer (REPL.tsx + Ink)                             │
│  Terminal UI, message rendering, input handling, focus management│
├─────────────────────────────────────────────────────────────────┤
│  Query Engine Layer (QueryEngine.ts, query.ts)                  │
│  API interaction, streaming, tool execution, context management │
├─────────────────────────────────────────────────────────────────┤
│  Tool Layer (Tool.ts, tools.ts, tools/*/)                       │
│  Tool definitions, permission checks, execution orchestration   │
├─────────────────────────────────────────────────────────────────┤
│  Services Layer (services/*)                                    │
│  API clients, MCP, analytics, compact, LSP, plugins, skills     │
├─────────────────────────────────────────────────────────────────┤
│  State Layer (state/*, bootstrap/state.ts)                      │
│  AppState store, global session state, bootstrap state          │
├─────────────────────────────────────────────────────────────────┤
│  Utilities Layer (utils/*)                                      │
│  Shared helpers, git, auth, settings, permissions, etc.         │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Descriptions

**1. Entry Points Layer**
- Minimal bootstrapping code with dynamic imports
- Fast-path routing for simple commands to avoid loading the full application
- Feature-gated code paths for different deployment modes

**2. CLI Layer (`main.tsx`)**
- Built on Commander.js for argument parsing
- Handles all CLI flags, options, and subcommands
- Determines interactive vs. non-interactive mode
- Orchestrates the startup sequence: settings loading → permission context → setup → REPL/headless execution
- Extensive feature flag gating for conditional imports (dead code elimination)

**3. Setup Layer (`setup.ts`)**
- Environment initialization after trust is established
- Worktree creation and tmux session management
- Hook system initialization (Setup, SessionStart hooks)
- Terminal backup/restore (iTerm2, Terminal.app)
- UDS (Unix Domain Socket) messaging server startup
- Background task registration (attribution hooks, team memory watcher)

**4. Interactive Layer (REPL + Ink)**
- React-based terminal UI using Ink framework
- Custom Ink wrapper (`src/ink.ts`) with theme provider integration
- Message rendering, input handling, focus management
- Terminal viewport tracking, animation frames
- Components for: messages, tools, tasks, teammates, permissions, modals, overlays

**5. Query Engine Layer (`QueryEngine.ts`, `query.ts`)**
- Core conversation loop: send messages to API, receive responses, execute tools
- `QueryEngine` class: Manages a conversation's lifecycle (multi-turn, stateful)
- `query()` function: The main query loop generator that handles:
  - Message normalization and context preparation
  - Auto-compaction, micro-compaction, snipping, context collapse
  - API streaming with fallback model support
  - Tool execution orchestration (streaming and batch)
  - Token budget tracking and recovery
  - Post-sampling hooks and stop hooks
- `ask()` convenience wrapper: One-shot query for headless/SDK mode

**6. Tool Layer (`Tool.ts`, `tools.ts`, `tools/*/`)**
- `Tool<T>` type: Comprehensive tool interface with input/output schemas, permission checks, rendering
- `buildTool()`: Factory function with safe defaults
- `getTools()`: Assembles available tools based on permission context and feature flags
- `assembleToolPool()`: Combines built-in tools with MCP tools
- Each tool is a self-contained module with: schema, description, call logic, permission checks, UI rendering
- 60+ built-in tools: Bash, Read, Edit, Write, Glob, Grep, WebSearch, Agent, Task*, Skill, and many more

**7. Services Layer (`services/*`)**
- **API**: Anthropic API client, retry logic, logging, usage tracking
- **MCP**: Model Context Protocol server/client management, OAuth, resource handling
- **Analytics**: GrowthBook feature flags, event logging, telemetry (OpenTelemetry)
- **Compact**: Auto-compaction, micro-compaction, reactive compact, session memory
- **LSP**: Language Server Protocol integration for diagnostics
- **Plugins**: Plugin loading, versioning, marketplace integration
- **Skills**: Skill discovery, loading, hot-reload detection

**8. State Layer**
- **AppState** (`state/AppStateStore.ts`): The central reactive state tree wrapped in `DeepImmutable<>`. Contains: settings, tools, MCP state, plugins, tasks, todos, notifications, bridge state, speculation state, and more.
- **Bootstrap State** (`bootstrap/state.ts`): Global singleton state for session-level data (cost, duration, session ID, model, telemetry counters). Accessed via getter/setter functions.
- **Store** (`state/store.ts`): Generic state store implementation (likely Zustand or similar pattern)

**9. Utilities Layer (`utils/*`)**
- 200+ utility modules covering: git operations, authentication, settings management, permission setup, file operations, string manipulation, platform detection, and domain-specific helpers

## Module Organization

The codebase is organized into the following top-level directories under `src/`:

### Core Modules

| Directory | Purpose | Key Files |
|---|---|---|
| `entrypoints/` | Application entry points | `cli.tsx`, `mcp.ts`, `init.ts`, `agentSdkTypes.ts` |
| `bootstrap/` | Global session state management | `state.ts` (1500+ lines of getters/setters) |
| `state/` | Reactive application state | `AppStateStore.ts`, `AppState.tsx`, `store.ts`, `selectors.ts` |
| `commands/` | Slash commands (/, local commands) | 90+ command modules organized by feature |
| `tools/` | Tool implementations | 60+ tool modules, each in its own directory |
| `services/` | External service integrations | API, MCP, analytics, compact, LSP, plugins |
| `components/` | React/Ink UI components | Message rendering, modals, panels, dialogs |
| `ink/` | Ink framework customization | Custom Ink components, hooks, events, DOM |
| `query/` | Query loop internals | `config.ts`, `deps.ts`, `stopHooks.ts`, `tokenBudget.ts`, `transitions.ts` |
| `types/` | TypeScript type definitions | Messages, permissions, tools, hooks, commands |
| `constants/` | Application constants | System prompts, tool limits, XML tags, API limits |
| `utils/` | Utility functions | 200+ helper modules |

### Feature-Specific Modules

| Directory | Purpose |
|---|---|
| `coordinator/` | Coordinator mode for multi-agent orchestration |
| `assistant/` | Assistant mode (Kairos) - proactive assistant features |
| `bridge/` | Remote control bridge (claude.ai integration) |
| `daemon/` | Long-running daemon process management |
| `buddy/` | Companion/sprite feature |
| `vim/` | Vim mode implementation |
| `voice/` | Voice mode implementation |
| `upstreamproxy/` | CCR upstream proxy for credential injection |
| `jobs/` | Template job classification |
| `plugins/` | Plugin system core |
| `skills/` | Skill system core |
| `memdir/` | Memory directory management |
| `native-ts/` | Native module type definitions |
| `outputStyles/` | Output style customization |
| `remote/` | Remote session management |
| `server/` | Server-side components (sessions, direct connect) |
| `self-hosted-runner/` | Self-hosted runner implementation |
| `environment-runner/` | BYOC environment runner |

### Sub-Directory Organization

**`tools/`** - Each tool follows a consistent pattern:
```
tools/
├── BashTool/
│   ├── BashTool.ts          # Main tool implementation
│   ├── prompt.ts            # System prompt instructions
│   ├── constants.ts         # Tool-specific constants
│   ├── bashSecurity.ts      # Security validation
│   ├── bashPermissions.ts   # Permission logic
│   └── ...                  # Additional helpers
├── FileEditTool/
├── FileReadTool/
├── AgentTool/
└── ...
```

**`services/`** - Organized by service domain:
```
services/
├── api/                     # Anthropic API client
├── mcp/                     # Model Context Protocol
├── analytics/               # Feature flags, telemetry
├── compact/                 # Context compaction strategies
├── lsp/                     # Language Server Protocol
├── tips/                    # User tip system
├── teamMemorySync/          # Team memory synchronization
└── ...
```

**`commands/`** - Each command is self-contained:
```
commands/
├── mcp/                     # MCP subcommands
│   ├── index.ts
│   └── addCommand.ts
├── plugin/
├── skills/
├── compact/
└── ... (each in its own directory or file)
```

## Key Design Patterns

### 1. Feature Flagging with Dead Code Elimination

The codebase uses `feature()` from `bun:bundle` extensively for compile-time feature gating:

```typescript
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

This enables dead code elimination (DCE) at build time — entire modules are excluded from the external build when their feature flag is disabled. This is critical for maintaining a single codebase that serves both internal (ant) and external (customer) builds.

Key feature flags include:
- `COORDINATOR_MODE` - Multi-agent coordinator
- `KAIROS` - Assistant/proactive mode
- `BRIDGE_MODE` - Remote control bridge
- `DAEMON` - Long-running daemon
- `BG_SESSIONS` - Background session management
- `HISTORY_SNIP` - History snipping compaction
- `CONTEXT_COLLAPSE` - Context collapse feature
- `SSH_REMOTE` - SSH remote sessions
- `DIRECT_CONNECT` - Direct connect sessions
- `WEB_BROWSER_TOOL` - Web browser tool
- `WORKFLOW_SCRIPTS` - Workflow automation
- `TRANSCRIPT_CLASSIFIER` - Auto-mode classifier
- `UDS_INBOX` - Unix domain socket messaging
- `CHICAGO_MCP` - Computer Use MCP

### 2. Dynamic Imports for Lazy Loading

Heavy modules are loaded dynamically to minimize startup time:

```typescript
const { setup } = await import('./setup.js')
const { bridgeMain } = await import('../bridge/bridgeMain.js')
```

This is combined with the startup profiler (`profileCheckpoint`) to track module evaluation time.

### 3. Memoized Global State

Bootstrap state uses a singleton pattern with getter/setter functions:

```typescript
const STATE: State = getInitialState()
export function getSessionId(): SessionId {
  return STATE.sessionId
}
export function switchSession(sessionId: SessionId, projectDir: string | null): void {
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)
}
```

Signal-based reactivity (`createSignal`) allows subscribers to react to state changes.

### 4. Immutable AppState with DeepImmutable Type

AppState is wrapped in `DeepImmutable<>` to enforce immutability at the type level:

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  // ...
}> & {
  tasks: { [taskId: string]: TaskState }  // Excluded from DeepImmutable (contains functions)
  // ...
}
```

State updates use the functional update pattern: `setAppState(prev => ({ ...prev, key: newValue }))`

### 5. Async Generator Pattern for Query Loop

The query system uses async generators for streaming:

```typescript
export async function* query(params: QueryParams): AsyncGenerator<Message | StreamEvent, Terminal> {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // ...
}
```

This enables real-time streaming of messages, tool executions, and events to consumers.

### 6. Tool Abstraction Pattern

Tools follow a comprehensive interface with:
- Zod input schemas for validation
- Dynamic description generation
- Permission checking (`checkPermissions`)
- Concurrency safety flags
- UI rendering methods (`renderToolUseMessage`, `renderToolResultMessage`)
- Progress reporting
- Auto-classifier input for security classification

### 7. Context Passing Pattern

Tool execution uses a `ToolUseContext` object that carries:
- Options (commands, tools, models, MCP clients)
- Abort controller
- State accessors (`getAppState`, `setAppState`)
- Callbacks (JSX rendering, notifications, prompts)
- Tracking state (query chain, skill discovery, memory paths)

### 8. Hook System

Extensible hooks at key lifecycle points:
- `Setup` hooks: Run during initialization
- `SessionStart` hooks: Run at session start
- `PreToolUse`/`PostToolUse` hooks: Run around tool execution
- `PreCompact`/`PostCompact` hooks: Run around compaction
- `Stop` hooks: Run when model stops generating

### 9. Prefetch Pattern

Startup performance is optimized through aggressive prefetching:
- Early prefetches: MDM settings, keychain, API preconnect
- Deferred prefetches: User context, tips, file counts, model capabilities
- Parallel execution: Independent prefetches run concurrently
- Trust-gated: Some prefetches only run after trust dialog acceptance

### 10. Command Pattern

Slash commands follow a consistent pattern with types:
- `prompt` commands: Expand to text sent to the model
- `local` commands: Execute locally (text output)
- `local-jsx` commands: Render Ink UI components

## Data Flow Overview

### Interactive Session Flow

```
User Input
    │
    ▼
┌──────────────┐
│  CLI Parser   │ (Commander.js - main.tsx)
│  (flags, opts)│
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│  Settings     │────▶│ Permission   │
│  Loading      │     │ Context      │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│  Setup        │────▶│ Tool Pool    │
│  (hooks, env) │     │ Assembly     │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
┌─────────────────────────────────────┐
│  REPL (Interactive TUI)              │
│  ┌───────────────────────────────┐  │
│  │  Message Queue                 │  │
│  │  ↓                             │  │
│  │  QueryEngine.submitMessage()   │  │
│  │  ↓                             │  │
│  │  query() loop:                 │  │
│  │    1. Context prep             │  │
│  │    2. API streaming            │  │
│  │    3. Tool execution           │  │
│  │    4. Hook execution           │  │
│  │    5. Repeat until done        │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  Transcript   │
│  Persistence  │
└──────────────┘
```

### Non-Interactive (SDK) Flow

```
stdin (stream-json)
    │
    ▼
┌──────────────┐
│  CLI Parser   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  QueryEngine  │
│  submitMessage│
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  query() loop │
│  (same as     │
│   interactive)│
└──────┬───────┘
       │
       ▼
stdout (stream-json)
```

### Tool Execution Flow

```
Model requests tool use
    │
    ▼
┌──────────────┐
│  validateInput│ (Tool-specific validation)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  checkPermissions│ (Permission rules evaluation)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  canUseTool   │ (Hook execution, classifier)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  tool.call()  │ (Actual execution)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  ToolResult   │ (Result + optional new messages)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  API response │ (Tool result sent back to model)
└──────────────┘
```

## Feature Flags

The feature flag system operates at two levels:

### Build-Time Flags (`feature()` from `bun:bundle`)

These are evaluated at build time and enable dead code elimination:

| Flag | Purpose | Build Impact |
|---|---|---|
| `COORDINATOR_MODE` | Multi-agent coordinator | Excludes coordinator modules |
| `KAIROS` | Assistant/proactive mode | Excludes assistant modules |
| `BRIDGE_MODE` | Remote control | Excludes bridge modules |
| `DAEMON` | Daemon process | Excludes daemon modules |
| `BG_SESSIONS` | Background sessions | Excludes bg management |
| `HISTORY_SNIP` | History snipping | Excludes snip modules |
| `CONTEXT_COLLAPSE` | Context collapse | Excludes collapse modules |
| `SSH_REMOTE` | SSH sessions | Excludes SSH modules |
| `DIRECT_CONNECT` | Direct connect | Excludes direct connect |
| `WEB_BROWSER_TOOL` | Browser tool | Excludes browser tool |
| `WORKFLOW_SCRIPTS` | Workflows | Excludes workflow modules |
| `TRANSCRIPT_CLASSIFIER` | Auto-mode | Excludes classifier |
| `UDS_INBOX` | UDS messaging | Excludes UDS modules |
| `CHICAGO_MCP` | Computer Use | Excludes CU modules |
| `KAIROS_PUSH_NOTIFICATION` | Push notifications | Excludes push modules |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhooks | Excludes webhook modules |
| `AGENT_TRIGGERS` | Cron triggers | Excludes cron modules |
| `MONITOR_TOOL` | Monitor tool | Excludes monitor modules |
| `OVERFLOW_TEST_TOOL` | Test tool | Excludes test modules |
| `TERMINAL_PANEL` | Terminal panel | Excludes terminal capture |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill search | Excludes skill search |
| `CCR_REMOTE_SETUP` | Remote setup | Excludes remote setup |
| `FORK_SUBAGENT` | Fork subagents | Excludes fork modules |
| `BUDDY` | Companion feature | Excludes buddy modules |
| `TORCH` | Torch feature | Excludes torch modules |
| `ULTRAPLAN` | Ultraplan feature | Excludes ultraplan modules |
| `PROACTIVE` | Proactive mode | Excludes proactive modules |
| `REACTIVE_COMPACT` | Reactive compact | Excludes reactive compact |
| `CACHED_MICROCOMPACT` | Cached microcompact | Excludes cached MC |
| `TEMPLATES` | Templates | Excludes template modules |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC runner | Excludes BYOC modules |
| `SELF_HOSTED_RUNNER` | Self-hosted runner | Excludes SH runner |
| `ABLATION_BASELINE` | Ablation testing | Excludes ablation code |
| `DUMP_SYSTEM_PROMPT` | System prompt dump | Excludes dump command |
| `BREAK_CACHE_COMMAND` | Cache breaking | Excludes cache break |
| `MCP_SKILLS` | MCP skills | Excludes MCP skill code |
| `TEAMMEM` | Team memory | Excludes team memory |
| `COMMIT_ATTRIBUTION` | Commit attribution | Excludes attribution |
| `TOKEN_BUDGET` | Token budget | Excludes budget code |
| `LODESTONE` | Deep links | Excludes deep link handler |

### Runtime Flags (GrowthBook)

Feature values fetched from GrowthBook at runtime:
- `tengu_kairos` - Assistant mode gate
- `tengu_ccr_bridge` - Bridge gate
- `tengu_tool_pear` - Tool strict mode
- `tengu_otk_slot_v1` - Output token escalation
- Various config-driven feature values

### Environment Variable Flags

Runtime toggles via environment variables:
- `CLAUDE_CODE_SIMPLE` - Minimal mode
- `CLAUDE_CODE_REMOTE` - Remote mode
- `CLAUDE_CODE_TASK_LIST_ID` - Task list mode
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS` - Disable CLAUDE.md
- `CLAUDE_CODE_USE_BEDROCK` - Bedrock provider
- `CLAUDE_CODE_USE_VERTEX` - Vertex provider
- `USER_TYPE=ant` - Internal Anthropic build
- `NODE_ENV=test` - Test mode

## External Dependencies

### Core Framework Dependencies

| Dependency | Purpose |
|---|---|
| `@anthropic-ai/sdk` | Anthropic API client for Claude messages |
| `@commander-js/extra-typings` | CLI argument parsing with TypeScript support |
| `react` + `ink` | Terminal UI rendering framework |
| `zod/v4` | Runtime schema validation for tool inputs |
| `@modelcontextprotocol/sdk` | MCP protocol implementation |
| `@opentelemetry/*` | Telemetry (metrics, logs, traces) |
| `bun:bundle` | Build-time feature flagging (Bun-specific) |
| `chalk` | Terminal color output |
| `lodash-es` | Utility functions (memoize, mapValues, pickBy, etc.) |

### Infrastructure Dependencies

| Dependency | Purpose |
|---|---|
| `@anthropic-ai/sdk` | API communication with Claude |
| `@modelcontextprotocol/sdk` | MCP server/client protocol |
| OpenTelemetry SDK | Customer-facing telemetry |
| GrowthBook | Feature flag evaluation |
| Datadog | Internal analytics (ant-only) |

### System Dependencies

| Dependency | Purpose |
|---|---|
| `node:crypto` | UUID generation, random bytes |
| `node:fs` | File system operations |
| `node:child_process` | Shell command execution |
| `node:vm` | REPL tool VM context |
| `node:net` | Unix domain sockets, networking |
| `node:readline` | Terminal input handling |
| `node:events` | Event emitter patterns |

## Module Relationships

### Dependency Graph (High-Level)

```
cli.tsx (entrypoint)
    │
    ├──► main.tsx (CLI orchestration)
    │       │
    │       ├──► bootstrap/state.ts (global session state)
    │       ├──► setup.ts (environment setup)
    │       ├──► commands.ts (command registry)
    │       ├──► tools.ts (tool assembly)
    │       │       └──► Tool.ts (tool type definitions)
    │       │               └──► tools/* (individual tools)
    │       ├──► QueryEngine.ts (query engine)
    │       │       └──► query.ts (query loop)
    │       │               └──► services/api/claude.ts (API client)
    │       │               └──► services/tools/toolOrchestration.ts
    │       ├──► state/AppStateStore.ts (reactive state)
    │       ├──► context.ts (system/user context)
    │       └──► interactiveHelpers.ts (REPL helpers)
    │
    ├──► services/mcp/ (MCP integration)
    ├──► services/analytics/ (telemetry, GrowthBook)
    ├──► services/compact/ (context management)
    ├──► utils/* (200+ utility modules)
    └──► components/* (React/Ink UI components)
```

### Key Module Relationships

**main.tsx → setup.ts → bootstrap/state.ts**
- `main.tsx` orchestrates startup, calling `setup()` after parsing CLI options
- `setup.ts` initializes the environment, hooks, and worktree
- Both read/write to `bootstrap/state.ts` for session-level state

**main.tsx → tools.ts → Tool.ts → tools/**
- `main.tsx` calls `getTools()` to assemble the tool pool
- `tools.ts` imports individual tool implementations and filters by permissions
- `Tool.ts` defines the tool interface and `buildTool()` factory
- Each tool in `tools/` implements the `Tool` interface

**QueryEngine.ts → query.ts → services/api/claude.ts**
- `QueryEngine.submitMessage()` wraps the `query()` generator
- `query()` handles the main loop: API call → tool execution → repeat
- `services/api/claude.ts` handles the actual API streaming

**state/AppStateStore.ts ↔ components/**
- `AppState` is the single source of truth for reactive UI state
- Components read from AppState via selectors or direct access
- State updates flow through `setAppState()` functional updates

**commands.ts ↔ skills/ ↔ plugins/**
- Commands are loaded from multiple sources: built-in, skills, plugins
- `getCommands()` merges all sources with availability filtering
- Skills and plugins can dynamically add commands at runtime

### Circular Dependency Management

The codebase uses several patterns to manage circular dependencies:
1. **Lazy requires**: `require()` inside functions to break cycles
2. **Type-only imports**: `import type` for type references
3. **Centralized type definitions**: Shared types in `types/` directory
4. **Feature-gated imports**: Conditional requires based on `feature()` flags

### Import DAG Enforcement

The `bootstrap/` directory is treated as a DAG leaf — it cannot import from higher-level modules. This is enforced by the `bootstrap-isolation` ESLint rule. Bootstrap state uses explicit getter/setter functions rather than importing complex types from the application layer.
