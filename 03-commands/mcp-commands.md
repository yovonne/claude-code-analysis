# MCP & Plugin Commands

## Purpose

Documents the CLI commands that manage MCP (Model Context Protocol) servers, plugins, and plugin lifecycle. These commands allow users to add, configure, enable/disable MCP servers, browse and manage plugins, and refresh plugin state without restarting.

---

## /mcp (MCP Server Management)

### Purpose
Manages MCP servers — adding, enabling, disabling, reconnecting, and configuring OAuth/XAA authentication. Also provides a full MCP settings UI.

### Location
`restored-src/src/commands/mcp/index.ts`
`restored-src/src/commands/mcp/mcp.tsx`
`restored-src/src/commands/mcp/addCommand.ts`
`restored-src/src/commands/mcp/xaaIdpCommand.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `mcp` |
| Type | `local-jsx` |
| Description | Manage MCP servers |
| Immediate | `true` |
| Argument Hint | `[enable\|disable [server-name]]` |

### Subcommands (via Commander.js)

#### `mcp add <name> <commandOrUrl> [args...]`
Adds an MCP server configuration to Claude Code.

| Option | Description |
|--------|-------------|
| `-s, --scope <scope>` | Configuration scope: `local`, `user`, or `project` (default: `local`) |
| `-t, --transport <transport>` | Transport type: `stdio`, `sse`, or `http` (default: `stdio`) |
| `-e, --env <env...>` | Set environment variables (e.g. `-e KEY=value`) |
| `-H, --header <header...>` | Set WebSocket headers (e.g. `-H "Authorization: Bearer ..."`) |
| `--client-id <clientId>` | OAuth client ID for HTTP/SSE servers |
| `--client-secret` | Prompt for OAuth client secret |
| `--callback-port <port>` | Fixed port for OAuth callback |
| `--xaa` | Enable XAA (SEP-990) for this server (hidden unless XAA enabled) |

**Transport handling:**
- **stdio**: Interprets `commandOrUrl` as an executable command with optional args. Warns if the argument looks like a URL but `--transport` wasn't explicitly set.
- **sse/http**: Interprets `commandOrUrl` as a URL. Supports headers, OAuth client ID/secret, and callback port configuration.

#### `mcp xaa` — XAA (SEP-990) IdP Connection Management
Manages the XAA Identity Provider connection used for MCP server authentication.

| Subcommand | Description |
|------------|-------------|
| `mcp xaa setup` | Configure the IdP connection (one-time setup). Requires `--issuer <url>` and `--client-id <id>`. Optional: `--client-secret`, `--callback-port` |
| `mcp xaa login` | Cache an IdP id_token for silent authentication. Default: runs OIDC browser login. With `--id-token <jwt>`: writes a pre-obtained JWT directly. `--force` ignores cached token |
| `mcp xaa show` | Display current IdP configuration (issuer, client ID, callback port, secret status, login status) |
| `mcp xaa clear` | Remove IdP configuration and cached tokens from settings and keychain |

#### In-REPL Subcommands (via JSX args)
| Usage | Behavior |
|-------|----------|
| `/mcp` | Opens MCP settings UI (or redirects to plugin manage tab for Ant users) |
| `/mcp no-redirect` | Opens MCP settings UI bypassing the Ant redirect (for testing) |
| `/mcp reconnect <server-name>` | Reconnects a specific MCP server |
| `/mcp enable [server-name]` | Enables all MCP servers or a specific one |
| `/mcp disable [server-name]` | Disables all MCP servers or a specific one |

### Key Exports

#### Functions
- `registerMcpAddCommand(mcp: Command)`: Registers the `mcp add` subcommand on a Commander command
- `registerMcpXaaIdpCommand(mcp: Command)`: Registers the `mcp xaa` subcommand tree
- `call(onDone, _context, args?)`: Entry point; routes to settings, reconnect, enable/disable, or redirect

### Dependencies

#### Internal
- `../../services/mcp/config.js` — `addMcpConfig` for persisting server configs
- `../../services/mcp/auth.js` — `readClientSecret`, `saveMcpClientSecret` for OAuth secrets
- `../../services/mcp/utils.js` — `ensureConfigScope`, `ensureTransport`, `parseHeaders`, `describeMcpConfigFilePath`
- `../../services/mcp/xaaIdpLogin.js` — Full XAA IdP login flow (setup, login, token cache)
- `../../services/mcp/MCPConnectionManager.js` — `useMcpToggleEnabled` hook
- `../../components/mcp/index.js` — `MCPSettings` UI component
- `../../components/mcp/MCPReconnect.js` — `MCPReconnect` UI component
- `../../utils/envUtils.js` — `parseEnvVars` for environment variable parsing
- `../../utils/settings/settings.js` — `updateSettingsForSource` for persisting XAA config

#### External
- `@commander-js/extra-typings` — CLI argument parsing for subcommands
- `react` — Component rendering with `useEffect`, `useRef`

### Implementation Details

#### MCP Toggle Logic
The `MCPToggle` React component:
1. Reads MCP clients from app state (excluding the `ide` client)
2. Filters clients to toggle based on action (enable vs disable) and target (all vs specific server)
3. Calls `toggleMcpServer(name)` for each matching server
4. Reports result via `onComplete` callback

#### XAA Security Model
- The IdP connection is user-level: configure once, all XAA-enabled servers reuse it
- Stored in `settings.xaaIdp` (non-secret) + keychain slot keyed by issuer (secret)
- Separate trust domain from per-server AS (Authorization Server) secrets
- Validates issuer URL must be HTTPS (or HTTP for loopback only)
- Validates callback port is a positive integer
- Clears stale keychain slots when issuer or client ID changes

#### Edge Cases
- URL-like arguments without `--transport` emit a warning suggesting correct usage
- OAuth options (`--client-id`, `--client-secret`, `--callback-port`, `--xaa`) are only valid for HTTP/SSE transports
- XAA setup validates ALL inputs before writing any state to prevent partial configuration
- Ant users are redirected to the plugin manage tab when running `/mcp` without args

### Data Flow
```
User runs: claude mcp add my-server -- npx my-mcp
  → ensureConfigScope('local') → ensureTransport(undefined → 'stdio')
  → parseEnvVars(options.env)
  → addMcpConfig(name, { type: 'stdio', command, args, env }, scope)
  → Write to MCP config file
  → Output: "Added stdio MCP server my-server with command: npx my-mcp to local config"
```

```
User runs: claude mcp xaa setup --issuer https://idp.example.com --client-id my-client
  → Validate issuer URL (must be HTTPS or loopback HTTP)
  → updateSettingsForSource('userSettings', { xaaIdp: { issuer, clientId, callbackPort } })
  → Clear stale keychain slots if issuer changed
  → Save client secret to keychain if --client-secret provided
  → Output: "XAA IdP connection configured for https://idp.example.com"
```

---

## /plugin (Plugin Management)

### Purpose
Provides a full interactive UI for browsing, installing, configuring, and managing Claude Code plugins and marketplaces.

### Location
`restored-src/src/commands/plugin/index.tsx`
`restored-src/src/commands/plugin/plugin.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `plugin` |
| Aliases | `plugins`, `marketplace` |
| Type | `local-jsx` |
| Description | Manage Claude Code plugins |
| Immediate | `true` |

### Key Exports

#### Functions
- `call(onDone, _context, args?)`: Entry point; renders `PluginSettings` with optional args

### Dependencies

#### Internal
- `./PluginSettings.js` — Main plugin settings UI component
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering

### Implementation Details

#### Plugin Settings UI
The `PluginSettings` component supports multiple tabs/modes via the `args` parameter:
- **Installed**: View and manage currently installed plugins
- **Discover**: Browse available plugins from marketplaces
- **Manage**: Full plugin management (includes MCP redirect for Ant users)
- **Marketplaces**: Configure plugin marketplace sources

#### Component Architecture
The plugin command is backed by a rich component hierarchy:
- `PluginSettings` — Main container with tab navigation
- `ManagePlugins` — Installed plugin list with enable/disable/remove
- `DiscoverPlugins` — Plugin discovery and installation
- `BrowseMarketplace` — Marketplace browsing
- `ManageMarketplaces` — Add/remove marketplace sources
- `AddMarketplace` — Add a new marketplace source
- `PluginOptionsDialog` / `PluginOptionsFlow` — Plugin configuration wizards
- `PluginTrustWarning` — Security warnings for untrusted plugins
- `PluginErrors` — Error display for failed plugin operations
- `PluginDetailsHelpers` — Plugin metadata rendering utilities
- `UnifiedInstalledCell` — Unified display for installed plugin entries
- `ValidatePlugin` — Plugin validation logic
- `usePagination` — Pagination hook for large plugin lists

### Data Flow
```
User runs /plugin
  → PluginSettings component mounts
  → Loads installed plugins, marketplaces, and discoverable plugins
  → User navigates tabs, installs/configures/removes plugins
  → Changes applied to plugin configuration
  → User exits → onDone() closes
```

---

## /reload-plugins (Plugin Refresh)

### Purpose
Activates pending plugin changes in the current session without restarting. Refreshes plugins, skills, agents, hooks, MCP servers, and LSP servers from disk.

### Location
`restored-src/src/commands/reload-plugins/index.ts`
`restored-src/src/commands/reload-plugins/reload-plugins.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `reload-plugins` |
| Type | `local` |
| Description | Activate pending plugin changes in the current session |
| Non-interactive | `false` (interactive only) |

### Key Exports

#### Functions
- `call(_args, context)`: Refreshes all plugin sources and reports counts

### Dependencies

#### Internal
- `../../utils/plugins/refresh.js` — `refreshActivePlugins` for reloading all plugin types
- `../../services/settingsSync/index.js` — `redownloadUserSettings` for CCR mode
- `../../utils/settings/changeDetector.js` — `settingsChangeDetector` for file watcher notification
- `../../bootstrap/state.js` — `getIsRemoteMode` for remote mode detection
- `../../utils/envUtils.js` — `isEnvTruthy` for feature flag checks
- `../../utils/stringUtils.js` — `plural` for count formatting

#### External
- `bun:bundle` — `feature` flag for `DOWNLOAD_USER_SETTINGS`

### Implementation Details

#### Core Logic
1. **CCR mode**: If `DOWNLOAD_USER_SETTINGS` feature is enabled and in remote mode, re-downloads user settings from the remote backend. Triggers `settingsChangeDetector.notifyChange('userSettings')` to apply changes mid-session.
2. **Plugin refresh**: Calls `refreshActivePlugins(context.setAppState)` which reloads:
   - Enabled plugins
   - Command skills
   - Agents
   - Hooks
   - Plugin MCP servers
   - Plugin LSP servers
3. **Result formatting**: Builds a summary string with counts for each category using `n(count, noun)` helper with proper pluralization
4. **Error reporting**: If any plugins failed to load, appends a note directing the user to `/doctor` for details

#### Managed Settings Behavior
Managed settings are NOT re-fetched during reload — they already poll hourly (`POLLING_INTERVAL_MS`). Policy enforcement is eventually-consistent by design with stale-cache fallback on fetch failure.

#### Edge Cases
- No retry on settings download failure — user-initiated command, one attempt + fail-open
- Non-CCR headless mode (e.g., VSCode SDK subprocess) shares disk with settings writers — file watcher delivers changes, no re-pull needed
- "plugin MCP/LSP" label disambiguates from user-config/built-in servers (which `/reload-plugins` doesn't touch)

### Data Flow
```
User runs /reload-plugins
  → [CCR mode?] → redownloadUserSettings() → notifyChange('userSettings')
  → refreshActivePlugins(setAppState)
    → Reload plugins, skills, agents, hooks, MCP servers, LSP servers
  → Format result: "Reloaded: 3 plugins · 5 skills · 2 agents · 1 hook · 0 plugin MCP servers · 0 plugin LSP servers"
  → [Errors?] → Append "1 error during load. Run /doctor for details."
  → Return text result
```
