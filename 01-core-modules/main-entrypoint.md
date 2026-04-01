# Main Entrypoint (main.tsx)

## Purpose
The primary CLI entry point for Claude Code. This file orchestrates the entire application lifecycle: startup profiling, CLI argument parsing with Commander, feature flag resolution, authentication, MCP server initialization, session management (continue/resume/teleport/SSH), and Ink React rendering. It is the central routing hub that dispatches to the correct execution path based on command-line flags, environment variables, and build-time feature toggles.

## Location
`restored-src/src/main.tsx`

## File Size
~4,683 lines (~800KB)

---

## Functional Sections

### 1. Startup Profiling & Early Initialization (Lines 1-210)

This section runs **before all other imports** via top-level side effects to minimize startup latency:

```typescript
// 1. Mark entry point before heavy module evaluation
profileCheckpoint('main_tsx_entry');

// 2. Fire MDM subprocesses (plutil/reg query) in parallel with remaining imports
startMdmRawRead();

// 3. Fire macOS keychain reads (OAuth + legacy API key) in parallel
//    Otherwise isRemoteManagedSettingsEligible() reads them sequentially (~65ms)
startKeychainPrefetch();
```

**Key optimizations:**
- **`profileCheckpoint`** from `startupProfiler.ts` marks timing checkpoints using Node.js `performance.mark()`. Two modes: sampled Statsig logging (100% ant, 0.5% external) and detailed profiling (`CLAUDE_CODE_PROFILE_STARTUP=1`).
- **`startMdmRawRead()`** launches MDM (Mobile Device Management) subprocesses early so they complete during the ~135ms of remaining imports.
- **`startKeychainPrefetch()`** launches both macOS keychain reads in parallel, avoiding the ~65ms sequential penalty.
- **`ensureKeychainPrefetchCompleted()`** and **`ensureMdmSettingsLoaded()`** are awaited in the `preAction` hook before `init()`.

**Debug detection** (`isBeingDebugged()`, lines 232-263): Checks for `--inspect`, `--debug`, `NODE_OPTIONS` inspect flags, and `inspector.url()`. Exits immediately if detected to prevent debugger attachment in production builds.

**Early input capture**: `seedEarlyInput()` captures user typing during startup (the "user is typing" window), so input isn't lost while modules load.

### 2. CLI Setup with Commander (Lines 884-1008)

The `run()` function builds the Commander CLI program:

```typescript
const program = new CommanderCommand()
  .configureHelp(createSortedHelpConfig())
  .enablePositionalOptions()
  .name('claude')
  .description('Claude Code - starts an interactive session by default...')
  .argument('[prompt]', 'Your prompt', String)
  .helpOption('-h, --help', 'Display help for command')
```

**Global options include:**
- `-d, --debug [filter]` — Debug mode with category filtering
- `-p, --print` — Print response and exit (headless/SDK mode)
- `--bare` — Minimal mode: skips hooks, LSP, plugin sync, attribution, background prefetches, keychain reads, CLAUDE.md auto-discovery
- `--init`, `--init-only`, `--maintenance` — Hook lifecycle triggers
- `--output-format` — `text`, `json`, or `stream-json`
- `--max-budget-usd`, `--task-budget` — Cost controls
- `--allowed-tools`, `--disallowed-tools`, `--tools` — Tool allowlists
- `--mcp-config` — Load MCP servers from JSON files/strings
- `--permission-mode` — Permission mode selection
- `-c, --continue` — Continue most recent conversation
- `-r, --resume [value]` — Resume by session ID or interactive picker
- `--model`, `--effort`, `--agent`, `--betas` — Model/session configuration
- `--settings`, `--setting-sources` — Settings file/source control
- `--chrome` / `--no-chrome` — Claude in Chrome integration
- `--plugin-dir` — Load plugins from directories
- `--file` — Download file resources at startup

**Sorted help config** (lines 890-901): Options are sorted alphabetically by long option name for readability.

### 3. Feature Flags (Lines 21-82, throughout)

Feature flags use `feature('FLAG_NAME')` from `bun:bundle` for build-time dead code elimination:

| Flag | Purpose |
|------|---------|
| `COORDINATOR_MODE` | Multi-agent coordinator mode; conditionally imports `coordinatorModeModule` |
| `KAIROS` | Assistant mode (Agent SDK); conditionally imports `assistantModule` and `kairosGate` |
| `DIRECT_CONNECT` | Session server connectivity (cc:// URLs, `claude server`) |
| `SSH_REMOTE` | SSH remote execution (`claude ssh <host>`) |
| `LODESTONE` | Deep link URI handling (`--handle-uri`) |
| `BRIDGE_MODE` | Remote Control (`--remote-control` / `--rc`) |
| `TRANSCRIPT_CLASSIFIER` | Auto mode classifier |
| `PROACTIVE` | Autonomous proactive mode |
| `KAIROS_BRIEF` | Brief mode (SendUserMessage tool) |
| `KAIROS_CHANNELS` | MCP channel notifications |
| `UDS_INBOX` | Unix domain socket messaging |
| `CHICAGO_MCP` | Computer Use MCP (macOS) |
| `BG_SESSIONS` | Background sessions |
| `HARD_FAIL` | Crash on errors instead of logging |
| `AGENT_MEMORY_SNAPSHOT` | Agent memory persistence |
| `CCR_MIRROR` | Claude Code Remote mirroring |
| `WEB_BROWSER_TOOL` | Web browser integration |
| `UPLOAD_USER_SETTINGS` | Settings sync to cloud |
| `TRANSCRIPT_CLASSIFIER` | Auto mode + transcript classification |

**Conditional imports pattern:**
```typescript
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null;
```

**Build variant detection:** `"external" === 'ant'` checks if this is the internal Anthropic build vs. the public external build. This gates ant-only features (tasks, log/error commands, rollback, computer use, etc.).

### 4. Command Registration (Lines 3892-4512)

Subcommands are registered **after** the main command's action handler. In print mode (`-p`), subcommand registration is skipped entirely (~65ms savings):

```typescript
const isPrintMode = process.argv.includes('-p') || process.argv.includes('--print');
if (isPrintMode && !isCcUrl) {
  await program.parseAsync(process.argv);
  return program;
}
```

**Registered subcommand groups:**

| Command | Description | Feature Gate |
|---------|-------------|-------------|
| `claude mcp serve` | Start MCP server | Always |
| `claude mcp add/remove/list/get` | MCP server management | Always |
| `claude mcp add-json` | Add MCP from JSON string | Always |
| `claude mcp add-from-claude-desktop` | Import from Claude Desktop | Always |
| `claude mcp reset-project-choices` | Reset project MCP approvals | Always |
| `claude server` | Start session server | `DIRECT_CONNECT` |
| `claude ssh <host> [dir]` | SSH remote execution | `SSH_REMOTE` |
| `claude open <cc-url>` | Connect to server (headless) | `DIRECT_CONNECT` |
| `claude auth login/status/logout` | Authentication management | Always |
| `claude plugin validate/list/install/uninstall/enable/disable/update` | Plugin management | Always |
| `claude plugin marketplace add/list/remove/update` | Marketplace management | Always |
| `claude setup-token` | Long-lived auth token setup | Always |
| `claude agents` | List configured agents | Always |
| `claude auto-mode defaults/config/critique` | Auto mode inspection | `TRANSCRIPT_CLASSIFIER` |
| `claude remote-control / rc` | Remote Control bridge | `BRIDGE_MODE` |
| `claude assistant [sessionId]` | Attach to bridge session | `KAIROS` |
| `claude doctor` | Health check | Always |
| `claude update / upgrade` | Check for updates | Always |
| `claude install` | Install native build | Always |
| `claude up` | Run CLAUDE.md setup | ant-only |
| `claude rollback` | Roll back releases | ant-only |
| `claude log / error / export` | Conversation log management | ant-only |
| `claude task create/list/get/update/dir` | Task management | ant-only |
| `claude completion <shell>` | Shell completion script | ant-only, hidden |

**Pre-action hook** (lines 907-967): Runs before any command execution:
1. Awaits MDM and keychain prefetch completion
2. Calls `init()` (config, env vars, graceful shutdown, telemetry)
3. Sets `process.title = 'claude'`
4. Attaches logging sinks
5. Wires `--plugin-dir` to inline plugins
6. Runs migrations
7. Starts remote managed settings and policy limits loading (non-blocking)
8. Uploads user settings (if `UPLOAD_USER_SETTINGS` flag)

### 5. Authentication Flow

Authentication is handled **after trust is established** (or implicitly in non-interactive mode):

**Pre-auth setup:**
- `startKeychainPrefetch()` fires at module load (line 20)
- `ensureKeychainPrefetchCompleted()` awaited in preAction hook (line 914)
- `populateOAuthAccountInfoIfNeeded()` called during `init()`

**Post-onboarding auth refresh** (lines 2279-2297):
```typescript
if (onboardingShown) {
  void refreshRemoteManagedSettings();
  void refreshPolicyLimits();
  resetUserCache();
  refreshGrowthBookAfterAuthChange();
  void import('./bridge/trustedDevice.js').then(m => {
    m.clearTrustedDeviceToken();
    return m.enrollTrustedDevice();
  });
}
```

**Org validation** (lines 2299-2305): `validateForceLoginOrg()` checks if the active token's org matches the `forceLoginOrgUUID` from managed settings.

**Session ingress auth** (lines 1309-1330): `getSessionIngressAuthToken()` retrieves tokens for file downloads in remote sessions.

### 6. MCP Initialization (Lines 1413-1630, 2380-2430, 2691-2808)

MCP initialization happens in **three phases**:

**Phase 1 — Config loading** (started early, lines 1799-1814):
```typescript
const mcpConfigPromise = (strictMcpConfig || isBareMode()
  ? Promise.resolve({ servers: {} })
  : getClaudeCodeMcpConfigs(dynamicMcpConfig)
).then(result => { ... });
```

**Phase 2 — Dynamic MCP config parsing** (lines 1413-1523):
- Parses `--mcp-config` arguments as JSON strings or file paths
- Validates against reserved names (Claude in Chrome, Computer Use)
- Applies enterprise policy filtering (`filterMcpServersByPolicy`)
- Adds `scope: 'dynamic'` to all configs

**Phase 3 — Resource prefetching** (lines 2404-2430):
```typescript
const localMcpPromise = isNonInteractiveSession
  ? Promise.resolve({ clients: [], tools: [], commands: [] })
  : prefetchAllMcpResources(regularMcpConfigs);
```
- Interactive mode: connects to all MCP servers before REPL renders
- Non-interactive mode: per-server incremental push into headlessStore
- Claude.ai connectors are fetched in parallel with a 5-second timeout

**Special MCP integrations:**
- **Claude in Chrome** (lines 1525-1577): Sets up MCP config, allowed tools, and system prompt for browser integration
- **Chicago MCP / Computer Use** (lines 1597-1630): macOS-only, GrowthBook-gated screen capture
- **Enterprise MCP config** (lines 1582-1595): Blocks dynamic MCP when enterprise config exists
- **Strict MCP config** (line 1580): `--strict-mcp-config` ignores all MCP except `--mcp-config`

### 7. Rendering Pipeline (Lines 2213-2242, 2926-3036, 3760-3807)

**Ink root creation** (lines 2226-2229):
```typescript
const { createRoot } = await import('./ink.js');
root = await createRoot(ctx.renderOptions);
```

**Setup screens** (lines 2239-2242):
```typescript
const onboardingShown = await showSetupScreens(
  root, permissionMode, allowDangerouslySkipPermissions,
  commands, enableClaudeInChrome, devChannels
);
```
Shows trust dialog, OAuth login, onboarding, and resume picker as needed.

**Initial AppState construction** (lines 2926-3036): A comprehensive state object containing:
- Settings, tasks, agent registry, verbose mode
- Model configuration, brief mode state
- MCP state (clients, tools, commands, resources)
- Plugin state (enabled, disabled, errors)
- Remote control / bridge state (replBridge fields)
- Notifications, elicitation queue, todos
- File history, attribution, thinking state
- Prompt suggestion, speculation state
- Initial user message from CLI prompt

**REPL launch** (lines 3798-3807):
```typescript
await launchRepl(root, {
  getFpsMetrics, stats, initialState
}, {
  ...sessionConfig,
  initialMessages,
  pendingHookMessages
}, renderAndRun);
```

**`renderAndRun`** (from `interactiveHelpers.tsx`): Renders the React tree, starts deferred prefetches, waits for exit, then performs graceful shutdown.

### 8. Session Management (Lines 3101-3807)

The main action handler branches into multiple session paths:

**Continue** (`-c`, lines 3101-3155):
- Loads most recent conversation from disk
- Clears session caches for fresh file/skill discovery
- Processes resumed conversation with attribution
- Launches REPL with restored messages and agent state

**Direct Connect** (`cc://` URLs, lines 3156-3192):
- Creates `DirectConnectSession` with server URL and auth token
- Sets working directory from server config
- Launches REPL with direct connect config (no local MCP/tools)

**SSH Remote** (`claude ssh`, lines 3193-3258):
- Probes remote host, deploys binary if needed
- Creates SSH session with unix-socket auth tunnel
- `--local` flag skips probe/deploy for e2e testing
- Progress output with `\r` + erase for in-place updates

**Assistant Mode** (`claude assistant`, lines 3259-3354):
- Discovers bridge sessions or uses provided session ID
- Installs assistant if none found (wizard + daemon wait)
- Sets `kairosActive` and `userMsgOptIn` for brief mode
- Launches REPL as a pure viewer client

**Resume** (`-r`, `--from-pr`, lines 3355-3759):
- Supports UUID, custom title search, ccshare URLs, file paths
- `--fork-session` creates a new session from resume point
- `--from-pr` filters sessions by linked PR
- Interactive picker with worktree support when no specific session given
- Cross-worktree resume via `matchedLog`

**Teleport** (`--teleport`, lines 3504-3579):
- Interactive mode: shows task selector, checks out branch, processes messages
- Direct mode: fetches session, validates repository, handles repo mismatch
- Uses progress UI component for visual feedback

**Remote** (`--remote`, lines 3409-3503):
- Creates remote session via CCR (Claude Code Remote)
- TUI mode check: requires description unless `tengu_remote_backend` gate is on
- Original behavior: prints session info and exits
- New behavior: starts local TUI with CCR engine

**Fresh session** (lines 3760-3807):
- Default path when no resume/continue/teleport flags
- Passes unresolved hooks promise to REPL for non-blocking render
- Deep link banner if launched via protocol handler
- Persists mode for future resumes

### 9. REPL Launcher (Lines 2213-2242, 3134-3807)

The REPL is launched via `launchRepl()` from `replLauncher.tsx`:

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>
): Promise<void>
```

**Launch paths and their configurations:**

| Path | Initial Messages | Tools | MCP |
|------|-----------------|-------|-----|
| Fresh session | Hook messages + deep link banner | Full tool set | Connected |
| Continue | Restored messages | Full tool set | Connected |
| Direct Connect | Connection info message | Empty | None (remote) |
| SSH | SSH info message | Empty | None (remote) |
| Assistant | Session attached message | Empty | None (remote) |
| Resume | Restored messages | Full tool set | Connected |
| Teleport | Teleported messages | Full tool set | Connected |
| Remote | Remote info + user message | Filtered (remote-safe) | None (remote) |

---

## Key Exports

| Export | Type | Description |
|--------|------|-------------|
| `main()` | `async function` | Top-level entry point called from CLI bootstrap |
| `startDeferredPrefetches()` | `async function` | Called after first render to warm caches |

## Dependencies

### Internal
- `./bootstrap/state.ts` — Global mutable state (cwd, session ID, client type, etc.)
- `./entrypoints/init.ts` — `init()` function (config, env vars, telemetry, graceful shutdown)
- `./setup.ts` — `setup()` function (worktree, UDS, git, memory, hooks)
- `./replLauncher.tsx` — `launchRepl()` for Ink rendering
- `./interactiveHelpers.tsx` — `showSetupScreens()`, `exitWithError()`, `renderAndRun()`
- `./commands.ts` — `getCommands()`, `filterCommandsForRemoteMode()`
- `./services/mcp/client.ts` — `getMcpToolsCommandsAndResources()`, `prefetchAllMcpResources()`
- `./services/mcp/config.ts` — `getClaudeCodeMcpConfigs()`, `parseMcpConfig()`, policy filtering
- `./services/analytics/growthbook.ts` — Feature flag initialization and refresh
- `./state/AppStateStore.ts` — `getDefaultAppState()`, `AppState` type
- `./state/store.ts` — `createStore()` for state management
- `./utils/startupProfiler.ts` — `profileCheckpoint()`, `profileReport()`
- `./utils/permissions/permissionSetup.ts` — Tool permission context initialization
- `./utils/sessionRestore.ts` — `processResumedConversation()`
- `./utils/teleport.ts` — Teleport session management
- `./server/createDirectConnectSession.ts` — Direct connect sessions
- `./ssh/createSSHSession.ts` — SSH remote sessions
- `./bridge/bridgeEnabled.js` — Remote Control bridge gating
- `./dialogLaunchers.ts` — Resume chooser, teleport wrapper, assistant installer

### External
- `@commander-js/extra-typings` — Typed CLI argument parsing
- `bun:bundle` — `feature()` for build-time feature flags
- `react` — UI rendering with Ink
- `chalk` — Terminal color output
- `lodash-es` — `mapValues`, `pickBy`, `uniqBy`

## Data Flow

```
CLI invocation
    │
    ▼
main.tsx module load
    ├── profileCheckpoint('main_tsx_entry')
    ├── startMdmRawRead()          ← parallel with imports
    ├── startKeychainPrefetch()    ← parallel with imports
    └── Import all modules (~135ms)
    │
    ▼
profileCheckpoint('main_tsx_imports_loaded')
    │
    ▼
main() function
    ├── Set NoDefaultCurrentDirectoryInExePath (Windows security)
    ├── Initialize warning handler
    ├── Parse special URLs (cc://, deep links, ssh, assistant)
    ├── Determine isNonInteractive (-p, --print, --init-only, no TTY)
    ├── Set clientType (cli, sdk, remote, github-action, etc.)
    ├── eagerLoadSettings() — parse --settings and --setting-sources
    │
    ▼
run() — Commander program setup
    ├── Define program, name, description, all global options
    ├── Register preAction hook:
    │   ├── await ensureMdmSettingsLoaded()
    │   ├── await ensureKeychainPrefetchCompleted()
    │   ├── await init() — config, env, telemetry, graceful shutdown
    │   ├── Set process.title
    │   ├── Attach logging sinks
    │   ├── Wire --plugin-dir
    │   ├── runMigrations()
    │   ├── loadRemoteManagedSettings() (non-blocking)
    │   └── loadPolicyLimits() (non-blocking)
    │
    ▼
    ├── Skip subcommands in print mode → parseAsync → exit
    │
    ▼
    └── Register all subcommands (mcp, auth, plugin, doctor, etc.)
    │
    ▼
    └── program.parseAsync(process.argv)
        │
        ▼
        Main action handler (default command)
            ├── Parse options (debug, tools, MCP, permissions, model, etc.)
            ├── maybeActivateProactive() / maybeActivateBrief()
            ├── initializeToolPermissionContext()
            ├── Start MCP config loading (parallel)
            ├── setup() — worktree, git, memory, hooks
            ├── Load commands + agent definitions (parallel with setup)
            ├── Compute effective model, advisor model, thinking config
            ├── Build initial AppState
            │
            ▼
            └── Branch by session path:
                ├── --continue     → loadConversationForResume → launchRepl
                ├── cc:// URL      → createDirectConnectSession → launchRepl
                ├── ssh <host>     → createSSHSession → launchRepl
                ├── assistant      → discover sessions → launchRepl (viewer)
                ├── --resume/-r    → loadConversationForResume → launchRepl
                ├── --teleport     → fetch session → teleportWithProgress → launchRepl
                ├── --remote       → teleportToRemote → launchRepl
                └── (default)      → launchRepl (fresh session)
                    │
                    ▼
                    launchRepl(root, appProps, replProps, renderAndRun)
                        ├── Dynamically import App + REPL components
                        └── renderAndRun(root, <App><REPL /></App>)
                            ├── root.render(element)
                            ├── startDeferredPrefetches()
                            └── Wait for exit → gracefulShutdown
```

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_PROFILE_STARTUP` | Enable detailed startup profiling |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | Exit after first render (benchmarking) |
| `CLAUDE_CODE_SIMPLE` | Set by `--bare`; disables hooks, LSP, plugins, etc. |
| `CLAUDE_CODE_ENTRYPOINT` | Identifies caller: `cli`, `sdk-cli`, `sdk-ts`, `sdk-py`, `remote`, `claude-vscode`, `claude-desktop`, `local-agent`, `mcp`, `claude-code-github-action` |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | `bridge` for remote-control sessions |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session ingress auth token |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | Remote session identifier |
| `CLAUDE_CODE_QUESTION_PREVIEW_FORMAT` | `markdown` or `html` |
| `CLAUDE_CODE_AGENT` | Agent override (set by `--agent` with `BG_SESSIONS`) |
| `CLAUDE_CODE_TASK_LIST_ID` | Task list ID for tasks mode |
| `CLAUDE_CODE_PROACTIVE` | Enable proactive mode |
| `CLAUDE_CODE_BRIEF` | Enable brief mode |
| `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` | Cloud provider selection |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` / `CLAUDE_CODE_SKIP_VERTEX_AUTH` | Skip credential prefetch |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | Skip setting process.title |
| `CLAUDE_CODE_REMOTE` | Remote mode (enables all hook events) |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | Enable partial message output |
| `CLAUDE_CODE_COORDINATOR_MODE` | Enable coordinator mode tool filtering |
| `CLAUDE_CODE_TERMINAL_RECORDING` | Enable asciicast recording (ant-only) |
| `CLAUDE_CODE_DISABLE_SESSION_DATA_UPLOAD` | Disable session data uploader |
| `CLAUDE_CODE_MESSAGING_SOCKET` | UDS messaging socket path |
| `NODE_EXTRA_CA_CERTS` | Additional CA certificates |
| `CLAUDE_CODE_CLIENT_CERT` | Client certificate for mTLS |
| `NoDefaultCurrentDirectoryInExePath` | Windows security (set to `1`) |
| `GITHUB_ACTIONS` | Detect GitHub Actions environment |
| `USER_TYPE` | `ant` for internal builds |

### Migration System

Current migration version: **11**. Migrations run synchronously on startup if `migrationVersion` in global config is stale:

1. `migrateAutoUpdatesToSettings` — Move auto-update config to settings
2. `migrateBypassPermissionsAcceptedToSettings` — Move permissions acceptance
3. `migrateEnableAllProjectMcpServersToSettings` — Move MCP server approvals
4. `resetProToOpusDefault` — Change default model from Pro to Opus
5. `migrateSonnet1mToSonnet45` — Model string migration
6. `migrateLegacyOpusToCurrent` — Legacy Opus → current model
7. `migrateSonnet45ToSonnet46` — Model string migration
8. `migrateOpusToOpus1m` — Model string migration
9. `migrateReplBridgeEnabledToRemoteControlAtStartup` — Bridge config migration
10. `resetAutoModeOptInForDefaultOffer` — Auto mode opt-in reset (TRANSCRIPT_CLASSIFIER)
11. `migrateFennecToOpus` — Internal model migration (ant-only)

Async migration: `migrateChangelogFromConfig` (fire-and-forget).

## Related Modules

- [Tool System](./tool-system.md)
- [Command System](./command-system.md)
- [State Management](./state-management.md)
- [MCP System](./mcp-system.md)
- [Session Management](./session-management.md)
- [Authentication](./authentication.md)
- [Feature Flags](./feature-flags.md)
- [Startup Performance](./startup-performance.md)
