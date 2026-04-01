# Integration Commands

## Purpose

Documents the CLI commands that integrate Claude Code with external platforms and environments: GitHub Actions, Slack, Chrome browser, Desktop app, IDEs, and mobile devices. These commands extend Claude Code's reach beyond the terminal.

---

## /install-github-app (GitHub Actions Setup)

### Purpose
Sets up Claude GitHub Actions for a repository through a multi-step interactive wizard. Installs the Claude GitHub App, configures API key secrets, and creates workflow files.

### Location
`restored-src/src/commands/install-github-app/index.ts`
`restored-src/src/commands/install-github-app/install-github-app.tsx`

Plus step components:
- `ApiKeyStep.tsx`, `CheckExistingSecretStep.tsx`, `CheckGitHubStep.tsx`
- `ChooseRepoStep.tsx`, `CreatingStep.tsx`, `ErrorStep.tsx`
- `ExistingWorkflowStep.tsx`, `InstallAppStep.tsx`, `OAuthFlowStep.tsx`
- `SuccessStep.tsx`, `WarningsStep.tsx`, `setupGitHubActions.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `install-github-app` |
| Type | `local-jsx` |
| Description | Set up Claude GitHub Actions for a repository |
| Availability | `['claude-ai', 'console']` |
| Enabled | Unless `DISABLE_INSTALL_GITHUB_APP_COMMAND` env var is truthy |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; renders `InstallGitHubApp` component

#### Components
- `InstallGitHubApp`: Multi-step wizard managing the entire GitHub App installation flow

### Dependencies

#### Internal
- Step components (see Location above)
- `../../components/WorkflowMultiselectDialog.js` — Workflow selection UI
- `../../constants/github-app.js` — `GITHUB_ACTION_SETUP_DOCS_URL`
- `../../hooks/useExitOnCtrlCDWithKeybindings.js` — Ctrl+C exit handling
- `../../utils/auth.js` — `getAnthropicApiKey`, `isAnthropicAuthEnabled`
- `../../utils/browser.js` — `openBrowser` for opening URLs
- `../../utils/execFileNoThrow.js` — `execFileNoThrow` for `gh` CLI commands
- `../../utils/git.js` — `getGithubRepo` for detecting current repo
- `../../utils/stringUtils.js` — `plural` for count formatting
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../ink.js` — `Box` for UI rendering
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `execa` — Process execution for `gh` CLI commands
- `react` — Component rendering with `useState`, `useCallback`, `useEffect`

### Implementation Details

#### Multi-Step Flow
The wizard progresses through these steps:

| Step | Description |
|------|-------------|
| `check-gh` | Checks GitHub CLI installation, authentication, and required scopes (`repo`, `workflow`) |
| `warnings` | Displays any warnings (missing CLI, not authenticated) with instructions |
| `choose-repo` | Select target repository (current repo or manual entry) |
| `install-app` | Opens GitHub App installation URL in browser |
| `check-existing-workflow` | Checks if Claude workflow file already exists |
| `select-workflows` | Multi-select dialog for choosing which workflows to install (`claude`, `claude-review`) |
| `check-existing-secret` | Checks if `ANTHROPIC_API_KEY` secret already exists in the repo |
| `api-key` | Enter API key or create OAuth token |
| `oauth-flow` | OAuth authentication flow (when OAuth is enabled) |
| `creating` | Progress display while setting up GitHub Actions |
| `success` | Completion confirmation |
| `error` | Error display with instructions |

#### GitHub CLI Checks
The `checkGitHubCLI` function:
1. Runs `gh --version` — warns if not found
2. Runs `gh auth status -a` — warns if not authenticated
3. Parses token scopes from stdout — errors if `repo` or `workflow` scopes missing
4. Detects current repo via `getGithubRepo()`
5. Sets `useCurrentRepo: true` if repo detected

#### Repository Validation
- Accepts `owner/repo` format or full GitHub URLs
- Extracts repo name from URLs via regex: `/github\.com[:/]([^/]+\/[^/]+)(\.git)?$/`
- Checks admin permissions via `gh api repos/{repo} --jq '.permissions.admin'`
- Checks existing workflow file via `gh api repos/{repo}/contents/.github/workflows/claude.yml`
- Checks existing secrets via `gh secret list --app actions --repo {repo}`

#### API Key / OAuth Options
Three authentication methods:
1. **Existing API key**: Uses locally configured `ANTHROPIC_API_KEY`
2. **New API key**: User enters API key manually
3. **OAuth token**: Creates OAuth token via interactive flow (when `isAnthropicAuthEnabled()`)

When OAuth is used, the secret name changes to `CLAUDE_CODE_OAUTH_TOKEN` and `authType` is set to `oauth_token`.

#### Workflow Installation
- Supports two workflows: `claude` and `claude-review`
- If workflow file already exists, user can update or skip
- Installation progress tracked via `currentWorkflowInstallStep` counter
- Uses `setupGitHubActions()` for the actual setup

#### Error Handling
Each error state includes:
- `error`: Human-readable error message
- `errorReason`: Short reason label
- `errorInstructions`: Array of actionable steps
- Links to documentation for manual setup

#### Analytics Events
| Event | When |
|-------|------|
| `tengu_install_github_app_started` | Command launched |
| `tengu_install_github_app_step_completed` | Each step completes |
| `tengu_install_github_app_error` | Error occurs (with reason) |
| `tengu_install_github_app_completed` | Successful completion |

### Data Flow
```
User runs /install-github-app
  → check-gh: Verify gh CLI, auth, scopes, current repo
  → [Warnings?] → Show warnings → User continues
  → choose-repo: Select or enter repository
  → Validate repo: permissions, existing workflow
  → install-app: Open GitHub App installation URL
  → [Existing workflow?] → select-workflows: Choose workflows
  → check-existing-secret: Check for ANTHROPIC_API_KEY
  → [No local key?] → api-key: Enter key or create OAuth token
  → creating: Run setupGitHubActions()
  → success: Show completion message
```

---

## /install-slack-app (Slack App Installation)

### Purpose
Opens the Slack app marketplace page for installing the Claude Slack app integration.

### Location
`restored-src/src/commands/install-slack-app/index.ts`
`restored-src/src/commands/install-slack-app/install-slack-app.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `install-slack-app` |
| Type | `local` |
| Description | Install the Claude Slack app |
| Availability | `['claude-ai']` |
| Non-interactive | `false` |

### Key Exports

#### Functions
- `call()`: Opens browser to Slack app marketplace and tracks installation attempt

### Dependencies

#### Internal
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../utils/browser.js` — `openBrowser` for opening the installation URL
- `../../utils/config.js` — `saveGlobalConfig` for tracking install attempts
- `../../commands.js` — `LocalCommandResult` type

### Implementation Details

#### Core Logic
1. Logs analytics event `tengu_install_slack_app_clicked`
2. Increments `slackAppInstallCount` in global config (tracks how many times user attempted install)
3. Opens browser to `https://slack.com/marketplace/A08SF47R6P4-claude`
4. Returns success or fallback URL message

### Data Flow
```
User runs /install-slack-app
  → logEvent('tengu_install_slack_app_clicked')
  → saveGlobalConfig(slackAppInstallCount: count + 1)
  → openBrowser(SLACK_APP_URL)
  │
  ├─ Success → "Opening Slack app installation page in browser…"
  └─ Failed → "Couldn't open browser. Visit: <SLACK_APP_URL>"
```

---

## /chrome (Claude in Chrome)

### Purpose
Manages the Claude in Chrome (Beta) integration — a Chrome extension that lets Claude Code control the browser directly for web automation, form filling, screenshots, and debugging.

### Location
`restored-src/src/commands/chrome/index.ts`
`restored-src/src/commands/chrome/chrome.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `chrome` |
| Type | `local-jsx` |
| Description | Claude in Chrome (Beta) settings |
| Availability | `['claude-ai']` |
| Enabled | Only in interactive sessions |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; checks extension status and renders `ClaudeInChromeMenu`

### Dependencies

#### Internal
- `../../components/CustomSelect/select.js` — `Select` for menu options
- `../../components/design-system/Dialog.js` — `Dialog` container
- `../../utils/claudeInChrome/common.js` — `CLAUDE_IN_CHROME_MCP_SERVER_NAME`, `openInChrome`
- `../../utils/claudeInChrome/setup.js` — `isChromeExtensionInstalled`
- `../../utils/config.js` — `getGlobalConfig`, `saveGlobalConfig` for `claudeInChromeDefaultEnabled` preference
- `../../utils/auth.js` — `isClaudeAISubscriber` for subscription check
- `../../utils/browser.js` — `openBrowser` for opening URLs
- `../../utils/env.js` — `env.isWslEnvironment()` for WSL detection
- `../../utils/envUtils.js` — `isRunningOnHomespace`
- `../../state/AppState.js` — `useAppState` for reading MCP clients
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering with React compiler, `useState`

### Implementation Details

#### Menu Options
| Option | Action |
|--------|--------|
| Install Chrome extension | Opens extension URL in browser |
| Manage permissions | Opens permissions configuration URL |
| Reconnect extension | Opens reconnect URL |
| Enabled by default: Yes/No | Toggles `claudeInChromeDefaultEnabled` config |

#### URL Routing
- **Homespace**: Uses `openBrowser(url)` directly
- **Non-Homespace**: Uses `openInChrome(url)` to open URLs within the Chrome integration

#### Connection Status
The menu displays:
- **Status**: Enabled/Disabled based on MCP client connection state (`CLAUDE_IN_CHROME_MCP_SERVER_NAME`)
- **Extension**: Installed/Not detected based on `isChromeExtensionInstalled()` check

#### Prerequisites & Warnings
- WSL: Shows error "Claude in Chrome is not supported in WSL at this time"
- Subscription: Shows error "Claude in Chrome requires a claude.ai subscription"
- Extension not installed: Shows install hint and "requires extension" suffix on relevant options

#### Usage Hint
Displays CLI flags: `claude --chrome` or `claude --no-chrome`

#### Permissions Note
"Site-level permissions are inherited from the Chrome extension. Manage permissions in the Chrome extension settings to control which sites Claude can browse, click, and type on."

### Data Flow
```
User runs /chrome
  → isChromeExtensionInstalled() → getGlobalConfig() → isClaudeAISubscriber() → isWslEnvironment()
  → Render ClaudeInChromeMenu with all status info
  → User selects action
  │
  ├─ install-extension → openUrl(CHROME_EXTENSION_URL)
  ├─ reconnect → openUrl(CHROME_RECONNECT_URL) → recheck extension status
  ├─ manage-permissions → openUrl(CHROME_PERMISSIONS_URL)
  └─ toggle-default → saveGlobalConfig(claudeInChromeDefaultEnabled: !current)
```

---

## /desktop (Desktop App Handoff)

### Purpose
Continues the current session in the Claude Desktop application (macOS and Windows x64 only).

### Location
`restored-src/src/commands/desktop/index.ts`
`restored-src/src/commands/desktop/desktop.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `desktop` |
| Aliases | `app` |
| Type | `local-jsx` |
| Description | Continue the current session in Claude Desktop |
| Availability | `['claude-ai']` |
| Enabled | Only on macOS (`darwin`) or Windows x64 (`win32` + `x64`) |
| Hidden | On unsupported platforms |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; renders `DesktopHandoff` component

### Dependencies

#### Internal
- `../../components/DesktopHandoff.js` — Desktop handoff UI component
- `../../commands.js` — `CommandResultDisplay` type

#### External
- `react` — Component rendering

### Implementation Details

#### Platform Support
The `isSupportedPlatform()` function checks:
- macOS: `process.platform === 'darwin'` → supported
- Windows x64: `process.platform === 'win32' && process.arch === 'x64'` → supported
- All other platforms → not supported (command hidden)

#### Core Logic
1. Renders `DesktopHandoff` component with done callback
2. The component handles the session handoff from CLI to Desktop app

### Data Flow
```
User runs /desktop (or /app)
  → [Supported platform?]
  │
  ├─ Yes → Render DesktopHandoff(onDone)
  │         → Session handoff to Claude Desktop
  │         → User exits → onDone() closes
  └─ No → Command hidden
```

---

## /ide (IDE Integration)

### Purpose
Manages IDE integrations — detects running IDEs with Claude Code extensions/plugins, connects to them for integrated development features, and offers to install extensions on detected IDEs.

### Location
`restored-src/src/commands/ide/index.ts`
`restored-src/src/commands/ide/ide.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `ide` |
| Type | `local-jsx` |
| Description | Manage IDE integrations and show status |
| Argument Hint | `[open]` |

### Key Exports

#### Functions
- `call(onDone, context, args)`: Entry point; detects IDEs and renders appropriate UI
- `formatWorkspaceFolders(folders, maxLength?)`: Formats workspace folder paths for display
- `findCurrentIDE(availableIDEs, dynamicMcpConfig?)`: Finds the currently connected IDE

#### Components
- `IDEScreen`: Main IDE selection screen
- `IDEOpenSelection`: IDE selection for opening projects
- `RunningIDESelector`: Selector when multiple IDEs are running (for extension install)
- `InstallOnMount`: Auto-installs extension on mount
- `IDECommandFlow`: Main flow orchestrating IDE connection

### Dependencies

#### Internal
- `../../components/CustomSelect/index.js` — `Select` for IDE selection
- `../../components/design-system/Dialog.js` — `Dialog` container
- `../../components/IdeAutoConnectDialog.js` — Auto-connect and disable auto-connect dialogs
- `../../services/mcp/client.js` — `clearServerCache` for MCP cache management
- `../../state/AppState.js` — `useAppState`, `useSetAppState` for MCP client state
- `../../utils/ide.js` — `detectIDEs`, `detectRunningIDEs`, `DetectedIDEInfo`, `IdeType`, `isJetBrainsIde`, `isSupportedJetBrainsTerminal`, `isSupportedTerminal`, `toIDEDisplayName`
- `../../utils/cwd.js` — `getCwd` for current working directory
- `../../utils/execFileNoThrow.js` — `execFileNoThrow` for IDE CLI commands
- `../../utils/worktree.js` — `getCurrentWorktreeSession` for git worktree support
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../commands.js` — `CommandResultDisplay`, `LocalJSXCommandContext` types
- `../../services/mcp/types.js` — `ScopedMcpServerConfig` type

#### External
- `chalk` — Terminal text formatting
- `path` — `path.sep` for path separator
- `react` — Component rendering with React compiler, `useState`, `useEffect`, `useCallback`, `useRef`

### Implementation Details

#### Two Modes

**Mode 1: `/ide open` — Open project in IDE**
1. Gets current worktree session path (or cwd)
2. Detects available IDEs with Claude Code extensions
3. Shows IDE selection dialog
4. On selection:
   - VS Code-based IDEs (VS Code, Cursor, Windsurf): Opens via `code <path>`
   - JetBrains IDEs: Prompts user to open manually
5. Shows success or failure message

**Mode 2: `/ide` — Connect to IDE**
1. Detects available IDEs (with extensions) and unavailable IDEs (running but wrong workspace)
2. Finds currently connected IDE from dynamic MCP config
3. Renders `IDECommandFlow` which:
   - Shows available IDEs in a selection dialog
   - On selection: Updates dynamic MCP config to connect to the IDE
   - Monitors connection status with 35-second timeout
   - On disconnect: Clears MCP config and removes IDE client from state

#### IDE Detection
- **Available IDEs**: Running IDEs with Claude Code extension/plugin installed AND matching workspace directories
- **Unavailable IDEs**: Running IDEs whose workspace/project directories don't match current cwd
- When no IDEs with extensions detected, checks for running IDEs and offers to install extension

#### Auto-Connect
- Shows auto-connect dialog when user first selects an IDE
- Shows disable auto-connect dialog when user disconnects
- Auto-connect can be managed via `/config` or `--ide` flag

#### MCP Connection
IDEs connect via dynamic MCP server config:
- `sse-ide` type for SSE connections
- `ws-ide` type for WebSocket connections
- Config includes: URL, IDE name, auth token, Windows detection flag

#### Workspace Folder Display
`formatWorkspaceFolders`:
- Strips cwd prefix from paths
- Shows first 2 workspaces max
- Truncates long paths with ellipsis
- Normalizes to NFC for consistent comparison (macOS NFD paths)

#### VS Code Limitation
Warning displayed when multiple VS Code instances detected: "Only one Claude Code instance can be connected to VS Code at a time."

#### Connection Timeout
35-second timeout (slightly longer than the 30s MCP connection timeout) with automatic cleanup.

### Data Flow
```
User runs /ide [open]
  │
  ├─ "open" argument → detectIDEs() → Filter valid IDEs
  │                    → [None?] → "No IDEs detected"
  │                    → Show IDEOpenSelection → User selects IDE
  │                    → VS Code: execFileNoThrow('code', [path])
  │                    → JetBrains: Manual open instructions
  │
  └─ No argument → detectIDEs() → Split into available/unavailable
                   → findCurrentIDE() from dynamic MCP config
                   → [No extensions + running IDEs?] → Offer to install extension
                   → Render IDECommandFlow
                     → User selects IDE → Update dynamic MCP config
                     → Monitor connection (35s timeout)
                     → [Connected?] → "Connected to <IDE name>."
                     → [Failed?] → "Failed to connect to <IDE name>."
                     → [Timeout?] → "Connection to <IDE name> timed out."
```

---

## /mobile (Mobile App QR Code)

### Purpose
Displays a QR code for downloading the Claude mobile app from the App Store (iOS) or Google Play (Android).

### Location
`restored-src/src/commands/mobile/index.ts`
`restored-src/src/commands/mobile/mobile.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `mobile` |
| Aliases | `ios`, `android` |
| Type | `local-jsx` |
| Description | Show QR code to download the Claude mobile app |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; renders `MobileQRCode` component

### Dependencies

#### Internal
- `../../components/design-system/Pane.js` — `Pane` container
- `../../keybindings/useKeybinding.js` — `useKeybinding` for keyboard handling
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../ink/events/keyboard-event.js` — `KeyboardEvent` type
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `qrcode` — QR code generation via `toString`
- `react` — Component rendering with React compiler, `useState`, `useEffect`, `useCallback`

### Implementation Details

#### Platform URLs
| Platform | URL |
|----------|-----|
| iOS | `https://apps.apple.com/app/claude-by-anthropic/id6473753684` |
| Android | `https://play.google.com/store/apps/details?id=com.anthropic.claude` |

#### QR Code Generation
- Generates QR codes for BOTH platforms on mount (pre-generates both)
- Uses `qrcode.toString()` with UTF8 format and low error correction
- Stores both QR codes in state for instant platform switching

#### UI Structure
```
Pane
├── QR code lines (ASCII art)
├── "  " (spacing)
├── "  " (spacing)
├── iOS / Android (toggle, bold + underline on active)
├── "(tab to switch, esc to close)" (dim)
└── Platform URL (dim)
```

#### Keyboard Controls
| Key | Action |
|-----|--------|
| Tab / Left / Right | Switch between iOS and Android |
| q / Ctrl+C | Close |
| Esc | Close |

### Data Flow
```
User runs /mobile (or /ios, /android)
  → useEffect: generateQRCodes() → qrToString(ios URL) + qrToString(android URL)
  → Render MobileQRCode
  → User views QR code for current platform
  → User presses Tab → Switch platform
  → User presses Esc/q/Ctrl+C → onDone() closes
```

---

## Integration Commands Overview

These six commands connect Claude Code to external platforms:

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ /install-github  │  │ /install-slack   │  │    /chrome       │
│      -app        │  │      -app        │  │ (browser control │
│ (GitHub Actions) │  │  (Slack app)     │  │  via extension)  │
└──────────────────┘  └──────────────────┘  └──────────────────┘

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   /desktop       │  │      /ide        │  │    /mobile       │
│ (Desktop app     │  │ (IDE integration │  │ (mobile app QR   │
│  handoff)        │  │  VS Code, JB)    │  │  code)           │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### Platform Availability
| Command | Platforms | Requirements |
|---------|-----------|-------------|
| `/install-github-app` | All | GitHub CLI, `gh` auth with repo+workflow scopes |
| `/install-slack-app` | All | Browser access |
| `/chrome` | All (except WSL) | claude.ai subscription, Chrome extension |
| `/desktop` | macOS, Windows x64 | Claude Desktop installed |
| `/ide` | All | IDE with Claude Code extension/plugin |
| `/mobile` | All | QR code display capability |
