# UI Commands

## Purpose

Documents the CLI commands that control the user interface, display information, and provide usage/cost reporting. These commands help users navigate the application, customize appearance, and monitor their usage.

---

## /help (Help & Commands)

### Purpose
Displays the help screen with all available commands and their descriptions.

### Location
`restored-src/src/commands/help/index.ts`
`restored-src/src/commands/help/help.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `help` |
| Type | `local-jsx` |
| Description | Show help and available commands |

### Key Exports

#### Functions
- `call(onDone, { options: { commands } })`: Entry point; renders `HelpV2` with command list

### Dependencies

#### Internal
- `../../components/HelpV2/HelpV2.js` — Help UI component
- `../../types/command.js` — `LocalJSXCommandCall` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Receives the full command list from `context.options.commands`
2. Renders `HelpV2` component with commands and close callback
3. The help component displays all registered slash commands with descriptions

### Data Flow
```
User runs /help
  → Render HelpV2(commands, onClose)
  → User browses available commands
  → User closes → onDone() closes
```

---

## /color (Session Color)

### Purpose
Sets the prompt bar color for the current session. Used primarily in swarm/multi-agent contexts to visually distinguish different agents.

### Location
`restored-src/src/commands/color/index.ts`
`restored-src/src/commands/color/color.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `color` |
| Type | `local-jsx` |
| Description | Set the prompt bar color for this session |
| Immediate | `true` |
| Argument Hint | `<color|default>` |

### Key Exports

#### Functions
- `call(onDone, context, args)`: Entry point; validates and applies color change

### Dependencies

#### Internal
- `../../tools/AgentTool/agentColorManager.js` — `AGENT_COLORS` list, `AgentColorName` type
- `../../utils/sessionStorage.js` — `getTranscriptPath`, `saveAgentColor` for persistence
- `../../bootstrap/state.js` — `getSessionId` for session identification
- `../../utils/teammate.js` — `isTeammate` for swarm session check
- `../../Tool.js` — `ToolUseContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone`, `LocalJSXCommandContext` types

#### External
- `crypto` — `UUID` type

### Implementation Details

#### Core Logic
1. **Teammate guard**: If session is a swarm teammate, blocks color change with message explaining that teammate colors are assigned by the team leader
2. **No argument**: Shows available colors list (`AGENT_COLORS.join(', ')` plus `default`)
3. **Reset aliases**: `default`, `reset`, `none`, `gray`, `grey` all reset to default color
   - Saves `'default'` sentinel to session storage (truthy for persistence guards)
   - Sets `standaloneAgentContext.color` to `undefined` in app state
4. **Valid color**: Validates against `AGENT_COLORS` list
   - Saves color to transcript via `saveAgentColor(sessionId, colorArg, fullPath)`
   - Updates `standaloneAgentContext.color` in app state for immediate effect
5. **Invalid color**: Shows error with available colors list

#### Color Persistence
Colors are saved to the transcript file for persistence across session restarts. The `'default'` sentinel (not empty string) ensures truthiness guards in `sessionStorage.ts` work correctly.

### Data Flow
```
User runs /color <color>
  → isTeammate() check → Block if true
  → [No arg?] → Show available colors list
  → [Reset alias?] → saveAgentColor(sessionId, 'default', path)
  │                   → setAppState(color: undefined)
  │                   → "Session color reset to default"
  → [Valid color?] → saveAgentColor(sessionId, colorArg, path)
  │                   → setAppState(color: colorArg)
  │                   → "Session color set to: <color>"
  → [Invalid color?] → Show error with available colors
```

---

## /stats (Usage Statistics)

### Purpose
Shows Claude Code usage statistics and activity overview.

### Location
`restored-src/src/commands/stats/index.ts`
`restored-src/src/commands/stats/stats.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `stats` |
| Type | `local-jsx` |
| Description | Show your Claude Code usage statistics and activity |

### Key Exports

#### Functions
- `call(onDone)`: Entry point; renders `Stats` component

### Dependencies

#### Internal
- `../../components/Stats.js` — Statistics display component
- `../../types/command.js` — `LocalJSXCommandCall` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Renders the `Stats` component with close callback
2. The component fetches and displays usage statistics from the user's account

### Data Flow
```
User runs /stats
  → Render Stats(onClose)
  → User views usage statistics
  → User closes → onDone() closes
```

---

## /status (System Status)

### Purpose
Shows Claude Code system status including version, model, account, API connectivity, and tool statuses.

### Location
`restored-src/src/commands/status/index.ts`
`restored-src/src/commands/status/status.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `status` |
| Type | `local-jsx` |
| Description | Show Claude Code status including version, model, account, API connectivity, and tool statuses |
| Immediate | `true` |

### Key Exports

#### Functions
- `call(onDone, context)`: Entry point; renders `Settings` component on the "Status" tab

### Dependencies

#### Internal
- `../../components/Settings/Settings.js` — Settings UI component with tab navigation
- `../../commands.js` — `LocalJSXCommandContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Renders the `Settings` component with `defaultTab="Status"`
2. The Settings component provides a tabbed interface; the Status tab shows:
   - Version information
   - Current model
   - Account details
   - API connectivity status
   - Tool statuses

### Data Flow
```
User runs /status
  → Render Settings(onClose, context, defaultTab="Status")
  → User views status information
  → User closes → onDone() closes
```

---

## /usage (Plan Usage)

### Purpose
Shows plan usage limits for claude.ai subscribers.

### Location
`restored-src/src/commands/usage/index.ts`
`restored-src/src/commands/usage/usage.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `usage` |
| Type | `local-jsx` |
| Description | Show plan usage limits |
| Availability | `['claude-ai']` |

### Key Exports

#### Functions
- `call(onDone, context)`: Entry point; renders `Settings` component on the "Usage" tab

### Dependencies

#### Internal
- `../../components/Settings/Settings.js` — Settings UI component with tab navigation
- `../../types/command.js` — `LocalJSXCommandCall` type

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic
1. Renders the `Settings` component with `defaultTab="Usage"`
2. Only available for `claude-ai` availability context
3. The Usage tab displays plan limits and current consumption

### Data Flow
```
User runs /usage
  → Render Settings(onClose, context, defaultTab="Usage")
  → User views plan usage limits
  → User closes → onDone() closes
```

---

## /extra-usage (Extra Usage Configuration)

### Purpose
Configures extra usage (overage) to keep working when plan limits are hit. For team/enterprise users without billing access, sends a limit increase request to admins.

### Location
`restored-src/src/commands/extra-usage/index.ts`
`restored-src/src/commands/extra-usage/extra-usage.tsx`
`restored-src/src/commands/extra-usage/extra-usage-core.ts`
`restored-src/src/commands/extra-usage/extra-usage-noninteractive.ts`

### Command Definition

**Interactive version:**
| Property | Value |
|----------|-------|
| Name | `extra-usage` |
| Type | `local-jsx` |
| Description | Configure extra usage to keep working when limits are hit |
| Enabled | When extra usage is allowed AND not in non-interactive session |

**Non-interactive version:**
| Property | Value |
|----------|-------|
| Name | `extra-usage` |
| Type | `local` |
| Description | Configure extra usage to keep working when limits are hit |
| Non-interactive | `true` |
| Enabled | When extra usage is allowed AND in non-interactive session |
| Hidden | When NOT in non-interactive session |

### Key Exports

#### Functions
- `call(onDone, context)` (interactive): Runs extra usage flow, may trigger login
- `runExtraUsage()`: Core extra usage logic (shared between interactive and non-interactive)

### Dependencies

#### Internal
- `../../services/api/adminRequests.js` — `checkAdminRequestEligibility`, `createAdminRequest`, `getMyAdminRequests`
- `../../services/api/overageCreditGrant.js` — `invalidateOverageCreditGrantCache`
- `../../services/api/usage.js` — `fetchUtilization`, `ExtraUsage` type
- `../../utils/auth.js` — `getSubscriptionType`, `hasClaudeAiBillingAccess`, `isOverageProvisioningAllowed`
- `../../utils/billing.js` — `hasClaudeAiBillingAccess`
- `../../utils/browser.js` — `openBrowser` for opening settings URLs
- `../../utils/config.js` — `getGlobalConfig`, `saveGlobalConfig` for visited flag
- `../../utils/log.js` — `logError` for error logging
- `../login/login.js` — `Login` component for re-authentication

#### External
- `react` — Component rendering

### Implementation Details

#### Core Logic (`runExtraUsage`)
1. Sets `hasVisitedExtraUsage` flag in global config (first visit tracking)
2. Invalidates overage credit grant cache to force fresh state
3. Determines subscription type and billing access

**Team/Enterprise without billing access:**
1. Checks utilization — if extra usage is enabled with no monthly limit, returns "already unlimited"
2. Checks admin request eligibility — if not allowed, directs user to contact admin
3. Checks for pending/dismissed admin requests — if found, informs user request already submitted
4. Creates a new admin request (`limit_increase`) — returns success message
5. Falls back to "contact your admin" message on any failure

**Users with billing access:**
1. Opens browser to the appropriate URL:
   - Team/Enterprise: `https://claude.ai/admin-settings/usage`
   - Individual: `https://claude.ai/settings/usage`
2. Returns browser open status or fallback URL message

#### Interactive Flow
If `runExtraUsage` returns `browser-opened`, the interactive version renders a `Login` component for new account setup following the extra-usage flow.

#### Edge Cases
- Disabled via `DISABLE_EXTRA_USAGE_COMMAND` env var
- Already unlimited: Returns informative message without opening browser
- Admin request already pending: Prevents duplicate requests
- Browser fails to open: Provides URL for manual visit
- Network errors: Logged and handled gracefully

### Data Flow
```
User runs /extra-usage
  → saveGlobalConfig(hasVisitedExtraUsage: true)
  → invalidateOverageCreditGrantCache()
  → getSubscriptionType() → hasClaudeAiBillingAccess()
  │
  ├─ Team/Enterprise + No billing access
  │   → fetchUtilization() → [Unlimited?] → "Already unlimited"
  │   → checkAdminRequestEligibility() → [Not allowed?] → "Contact admin"
  │   → getMyAdminRequests() → [Pending?] → "Request already submitted"
  │   → createAdminRequest('limit_increase') → "Request sent to admin"
  │
  └─ Has billing access
      → openBrowser(appropriate URL)
      → [Success?] → "Opening browser..."
      → [Failed?] → "Visit <URL> manually"
```

---

## /cost (Session Cost)

### Purpose
Shows the total cost and duration of the current session. Hidden for claude.ai subscribers (except Ant users who always see cost breakdowns).

### Location
`restored-src/src/commands/cost/index.ts`
`restored-src/src/commands/cost/cost.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `cost` |
| Type | `local` |
| Description | Show the total cost and duration of the current session |
| Non-interactive | `true` |
| Hidden | For claude.ai subscribers (except when `USER_TYPE === 'ant'`) |

### Key Exports

#### Functions
- `call()`: Returns current session cost information

### Dependencies

#### Internal
- `../../cost-tracker.js` — `formatTotalCost` for cost formatting
- `../../services/claudeAiLimits.js` — `currentLimits` for overage status
- `../../utils/auth.js` — `isClaudeAISubscriber` for subscriber check
- `../../types/command.js` — `LocalCommandCall` type

### Implementation Details

#### Core Logic
1. **Subscriber check**: If user is a claude.ai subscriber:
   - Shows overage status message:
     - Using overage: "You are currently using your overages to power your Claude Code usage..."
     - Using subscription: "You are currently using your subscription to power your Claude Code usage"
   - **Ant override**: Appends cost breakdown anyway with `[ANT-ONLY] Showing cost anyway:` prefix
2. **Non-subscribers**: Returns formatted total cost via `formatTotalCost()`

#### Visibility
- Hidden for subscribers via `isHidden` getter (unless `USER_TYPE === 'ant'`)
- Always visible for Ant users

### Data Flow
```
User runs /cost
  → isClaudeAISubscriber()
  │
  ├─ Yes (subscriber) → [Using overage?] → Overage message
  │                     → [Using subscription?] → Subscription message
  │                     → [USER_TYPE === 'ant'?] → Append cost breakdown
  │
  └─ No (non-subscriber) → formatTotalCost() → Return cost string
```

---

## UI Commands Overview

These commands provide the primary interface for information display and customization:

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  /help   │  │ /status  │  │  /stats  │  │ /color   │
│ (commands│  │ (system  │  │ (usage   │  │ (session │
│  & help) │  │  status) │  │  stats)  │  │  color)  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘

┌──────────┐  ┌──────────────┐  ┌──────────────┐
│ /usage   │  │ /extra-usage │  │    /cost     │
│ (plan    │  │ (overage     │  │ (session     │
│  limits) │  │  config)     │  │  cost)       │
└──────────┘  └──────────────┘  └──────────────┘
```

### Visibility Rules
| Command | Always Visible | Conditionally Hidden |
|---------|---------------|---------------------|
| `/help` | Yes | — |
| `/status` | Yes | — |
| `/stats` | Yes | — |
| `/color` | Yes | — |
| `/usage` | No | Only for `claude-ai` availability |
| `/extra-usage` | No | Disabled by env var or policy |
| `/cost` | No | Hidden for subscribers (except Ants) |
