# Remote Commands

## Purpose

Documents the CLI commands that manage remote session connectivity, environment configuration, and remote control (bridge) functionality. These commands enable Claude Code to operate in remote and distributed environments.

---

## /web-setup (Remote Session Setup)

### Purpose
Sets up Claude Code on the web (claude.ai/code) by connecting the user's GitHub account. Used for first-time remote session configuration.

### Location
`restored-src/src/commands/remote-setup/index.ts`
`restored-src/src/commands/remote-setup/remote-setup.tsx`
`restored-src/src/commands/remote-setup/api.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `web-setup` |
| Type | `local-jsx` |
| Description | Setup Claude Code on the web (requires connecting your GitHub account) |
| Availability | `['claude-ai']` |
| Enabled | When `tengu_cobalt_lantern` feature flag is on AND `allow_remote_sessions` policy is allowed |
| Hidden | When `allow_remote_sessions` policy is not allowed |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; renders the `Web` setup component
- `checkLoginState()`: Checks GitHub authentication status and token availability
- `importGithubToken(token)`: POSTs a GitHub token to the CCR backend for validation and storage
- `createDefaultEnvironment()`: Best-effort default environment creation for first-time users
- `isSignedIn()`: Checks if user has valid Claude OAuth credentials
- `getCodeWebUrl()`: Returns the claude.ai/code URL

#### Types
- `CheckResult`: Discriminated union for login state (`not_signed_in`, `has_gh_token`, `gh_not_installed`, `gh_not_authenticated`)
- `ImportTokenResult`: `{ github_username: string }`
- `ImportTokenError`: Discriminated union (`not_signed_in`, `invalid_token`, `server`, `network`)

#### Classes
- `RedactedGithubToken`: Wraps a raw GitHub token so that its string representation is redacted. `toString()`, `toJSON()`, and `inspect.custom` all return `[REDACTED:gh-token]`. Call `.reveal()` only at the point where the raw value is placed into an HTTP body.

### Dependencies

#### Internal
- `../../components/CustomSelect/index.js` — `Select` for user choices
- `../../components/design-system/Dialog.js` — `Dialog` container
- `../../components/design-system/LoadingState.js` — Loading state display
- `../../utils/browser.js` — `openBrowser` for opening URLs
- `../../utils/github/ghAuthStatus.js` — `getGhAuthStatus` for GitHub CLI auth check
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../constants/oauth.js` — `getOauthConfig` for API configuration
- `../../utils/teleport/api.js` — `getOAuthHeaders`, `prepareApiRequest`
- `../../utils/teleport/environments.js` — `fetchEnvironments` for environment checks
- `../../utils/debug.js` — `logForDebugging` for error logging
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `execa` — Process execution for `gh` CLI commands
- `axios` — HTTP client for API requests
- `react` — Component rendering with `useEffect`, `useState`

### Implementation Details

#### Setup Flow (Multi-step)
1. **Checking**: Verifies login state via `checkLoginState()`:
   - Checks Claude OAuth via `isSignedIn()` (calls `prepareApiRequest()`)
   - Checks GitHub CLI auth via `getGhAuthStatus()`
   - If authenticated, retrieves token via `gh auth token` (5s timeout)
   - Wraps token in `RedactedGithubToken` for safe handling

2. **Confirm**: Shows confirmation dialog explaining that:
   - Claude on the web requires GitHub connection for clone/push
   - Local credentials are used to authenticate with GitHub
   - User can Continue or Cancel

3. **Uploading**: Imports the GitHub token:
   - POSTs to `/v1/code/github/import-token` with the token
   - Server validates against GitHub's `/user` endpoint
   - Token is Fernet-encrypted and stored in `sync_user_tokens`
   - Creates default environment (best-effort, non-fatal)
   - Opens claude.ai/code in browser

#### Token Import API
The `importGithubToken` function:
- Uses `prepareApiRequest()` for OAuth authentication
- Sends POST to `{BASE_API_URL}/v1/code/github/import-token`
- Includes `anthropic-beta: ccr-byoc-2025-07-29` header
- Includes `x-organization-uuid` header
- Handles response codes: 200 (success), 400 (invalid token), 401 (not signed in)
- Network errors caught and reported without logging the token

#### Default Environment Creation
`createDefaultEnvironment()` creates a trusted network access environment:
- Checks for existing environments first (avoids duplicates)
- POSTs to `{BASE_API_URL}/v1/environment_providers/cloud/create`
- Environment config: `anthropic_cloud` type, Python 3.11 + Node 20, default host access
- Failures are non-fatal — the web onboarding falls back to env-setup

#### Security Considerations
- `RedactedGithubToken` prevents accidental token leakage in logs, errors, or JSON serialization
- Token is only revealed at the single point of HTTP body construction
- Error messages never include token data
- POST body is explicitly excluded from axios error logging

#### Edge Cases
- GitHub CLI not installed: Directs user to install from cli.github.com
- GitHub CLI not authenticated: Directs user to run `gh auth login`
- Not signed in to Claude: Prompts to run `/login` first
- Token import fails: Shows specific error based on failure type
- Environment creation fails: Non-fatal, web UI handles it

### Data Flow
```
User runs /web-setup
  → Log analytics: tengu_remote_setup_started
  → checkLoginState()
  │
  ├─ not_signed_in → "Not signed in to Claude. Run /login first."
  ├─ gh_not_installed → Open onboarding URL → Show install instructions
  ├─ gh_not_authenticated → Open onboarding URL → Show auth instructions
  └─ has_gh_token → Show confirmation dialog
                     → User confirms → importGithubToken(token)
                     │                  → POST to backend → Validate & store
                     │                  → createDefaultEnvironment() (best-effort)
                     │                  → openBrowser(claude.ai/code)
                     │                  → "Connected as <username>. Opened <url>"
                     └─ User cancels → Close
```

---

## /remote-env (Remote Environment Configuration)

### Purpose
Configures the default remote environment for teleport (remote) sessions.

### Location
`restored-src/src/commands/remote-env/index.ts`
`restored-src/src/commands/remote-env/remote-env.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `remote-env` |
| Type | `local-jsx` |
| Description | Configure the default remote environment for teleport sessions |
| Enabled | When user is a claude.ai subscriber AND `allow_remote_sessions` policy is allowed |
| Hidden | When not a subscriber OR policy not allowed |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; renders `RemoteEnvironmentDialog`

### Dependencies

#### Internal
- `../../components/RemoteEnvironmentDialog.js` — Remote environment configuration UI
- `../../utils/auth.js` — `isClaudeAISubscriber` for subscriber check
- `../../services/policyLimits/index.js` — `isPolicyAllowed` for policy check
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Renders `RemoteEnvironmentDialog` component with done callback
2. The dialog allows users to configure the default remote environment used when starting teleport sessions
3. Only available to claude.ai subscribers with the `allow_remote_sessions` policy enabled

### Data Flow
```
User runs /remote-env
  → [Subscriber? AND Policy allowed?]
  │
  ├─ Yes → Render RemoteEnvironmentDialog(onDone)
  │         → User configures remote environment
  │         → User exits → onDone() closes
  └─ No → Command hidden
```

---

## /remote-control (Bridge / Remote Control)

### Purpose
Connects the current terminal for remote-control sessions, enabling bidirectional communication between the CLI and claude.ai via a bridge connection.

### Location
`restored-src/src/commands/bridge/index.ts`
`restored-src/src/commands/bridge/bridge.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `remote-control` |
| Aliases | `rc` |
| Type | `local-jsx` |
| Description | Connect this terminal for remote-control sessions |
| Argument Hint | `[name]` |
| Immediate | `true` |
| Enabled | When `BRIDGE_MODE` feature is on AND bridge is enabled |
| Hidden | When bridge is not enabled |

### Key Exports

#### Functions
- `call(onDone, _context, args)`: Entry point; renders `BridgeToggle` with optional session name
- `checkBridgePrerequisites()`: Validates all bridge prerequisites before connecting

#### Components
- `BridgeToggle`: Manages the bridge connection lifecycle
- `BridgeDisconnectDialog`: Shown when already connected; offers disconnect, QR code, or continue

### Dependencies

#### Internal
- `../../bridge/bridgeConfig.js` — `getBridgeAccessToken` for auth token
- `../../bridge/bridgeEnabled.js` — `checkBridgeMinVersion`, `getBridgeDisabledReason`, `isEnvLessBridgeEnabled`, `isBridgeEnabled`
- `../../bridge/envLessBridgeConfig.js` — `checkEnvLessBridgeMinVersion`
- `../../bridge/types.js` — `BRIDGE_LOGIN_INSTRUCTION`, `REMOTE_CONTROL_DISCONNECTED_MSG`
- `../../components/design-system/Dialog.js` — `Dialog` container
- `../../components/design-system/ListItem.js` — `ListItem` for menu items
- `../../components/RemoteCallout.js` — `shouldShowRemoteCallout` for onboarding callout
- `../../context/overlayContext.js` — `useRegisterOverlay` for overlay management
- `../../keybindings/useKeybinding.js` — `useKeybindings` for keyboard navigation
- `../../state/AppState.js` — `useAppState`, `useSetAppState` for state management
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../services/policyLimits/index.js` — `isPolicyAllowed`, `waitForPolicyLimitsToLoad` (dynamic import)
- `../../utils/debug.js` — `logForDebugging` for debug logging
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../Tool.js` — `ToolUseContext` type
- `../../commands.js` — `LocalJSXCommandContext`, `LocalJSXCommandOnDone` types

#### External
- `bun:bundle` — `feature` flags for `BRIDGE_MODE`, `KAIROS`
- `qrcode` — QR code generation via `toString`
- `react` — Component rendering with React compiler, `useEffect`, `useState`

### Implementation Details

#### Bridge Connection Architecture
When enabled, the bridge:
1. Sets `replBridgeEnabled` in AppState
2. Triggers `useReplBridge` in REPL.tsx to initialize the connection
3. Registers an environment with the backend
4. Creates a session with the current conversation
5. Polls for work from the backend
6. Connects an ingress WebSocket for bidirectional messaging between CLI and claude.ai

#### Prerequisite Checks (`checkBridgePrerequisites`)
1. **Organization policy**: Waits for policy limits to load, checks `allow_remote_control`
2. **Bridge disabled reason**: Checks `getBridgeDisabledReason()` for any blocking conditions
3. **Version compatibility**:
   - Determines v1 vs v2 path based on `isEnvLessBridgeEnabled()` and `KAIROS` feature
   - In assistant mode (KAIROS), forces v1 path regardless of env-less flag
   - Checks minimum version for the appropriate path
4. **Access token**: Verifies bridge access token exists via `getBridgeAccessToken()`

#### Connection Flow
1. Checks if already connected (`replBridgeConnected` or `replBridgeEnabled` without `replBridgeOutboundOnly`)
2. If connected: Shows `BridgeDisconnectDialog` with session URL and options
3. If not connected:
   - Runs prerequisite checks
   - If `shouldShowRemoteCallout()`: Shows onboarding callout instead of connecting
   - Otherwise: Sets `replBridgeEnabled: true`, `replBridgeExplicit: true`, `replBridgeOutboundOnly: false`
   - Shows "Remote Control connecting…" message

#### Disconnect Dialog
When already connected, shows:
- Session URL (or connect URL if session not yet active)
- Three options:
  1. **Disconnect**: Terminates the bridge connection
  2. **Show/Hide QR code**: Generates QR code for the session URL
  3. **Continue**: Dismisses dialog and keeps the connection
- Keyboard navigation: Up/Down to select, Enter to accept, Esc to continue

#### State Management
Bridge state in AppState:
| Property | Purpose |
|----------|---------|
| `replBridgeEnabled` | Whether bridge is enabled |
| `replBridgeConnected` | Whether bridge is actively connected |
| `replBridgeOutboundOnly` | Outbound-only mode (no inbound control) |
| `replBridgeExplicit` | Whether user explicitly enabled bridge |
| `replBridgeSessionUrl` | Active session URL |
| `replBridgeConnectUrl` | Connection URL (before session active) |
| `replBridgeSessionActive` | Whether session is active |
| `replBridgeInitialName` | Initial session name |

#### Env-Less Bridge (v2) vs Traditional (v1)
- **v2 (env-less)**: Used when `isEnvLessBridgeEnabled()` is true AND session is not perpetual
- **v1 (traditional)**: Used in assistant mode (KAIROS sets `perpetual=true`) or when env-less flag is off
- Different minimum version checks apply to each path

#### Edge Cases
- Already connected: Shows disconnect dialog instead of re-connecting
- Prerequisite failure: Reports specific error message
- Connection timeout: 35-second timeout with cleanup
- Cancelled connection: Async operation tracks `cancelled` flag to avoid state updates after unmount
- QR code generation: Async with error handling (empty string on failure)

### Data Flow
```
User runs /remote-control [name]
  → BridgeToggle component mounts
  → [Already connected?]
  │
  ├─ Yes → Show BridgeDisconnectDialog
  │         → User: Disconnect → setAppState(replBridgeEnabled: false, ...)
  │         → User: Show QR → qrToString(sessionUrl) → display QR
  │         → User: Continue → onDone(undefined, { display: 'skip' })
  │
  └─ No → checkBridgePrerequisites()
           → [Failed?] → onDone(error message)
           → [Show remote callout?] → setAppState(showRemoteCallout: true)
           → [All clear?] → setAppState(replBridgeEnabled: true, replBridgeExplicit: true, ...)
                           → "Remote Control connecting…"
                           → useReplBridge initializes connection (in REPL.tsx)
```

---

## Remote Commands Overview

These three commands manage the remote session ecosystem:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  /web-setup  │     │ /remote-env  │     │ /remote-control│
│ (connect     │     │ (configure   │     │ (bridge CLI   │
│  GitHub to   │     │  remote      │     │  to claude.ai)│
│  claude.ai)  │     │  environment)│     │               │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │    Remote Session         │
              │    Infrastructure         │
              │  (teleport, bridge,       │
              │   environments)           │
              └───────────────────────────┘
```

### Prerequisites Chain
1. **Policy**: `allow_remote_sessions` and `allow_remote_control` must be allowed
2. **Authentication**: User must be signed in to Claude (OAuth)
3. **Feature flags**: `tengu_cobalt_lantern` for web-setup, `BRIDGE_MODE` for remote-control
4. **GitHub**: Connected via web-setup for claude.ai/code functionality
5. **Environment**: Configured via remote-env for teleport sessions
6. **Bridge**: Activated via remote-control for bidirectional CLI-web communication
