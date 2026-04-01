# Settings Commands

## Overview

Settings commands manage runtime behavior including effort levels, fast mode, tool permissions, sandboxing, usage limits, and rate limit handling. These commands provide both interactive UI and direct argument-based control.

---

## `/effort`

**Purpose:** Set the effort level for model usage, controlling how deeply the model reasons and works on tasks.

**Source:** `restored-src/src/commands/effort/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with dynamic immediate mode |
| `effort.tsx` | Effort level logic, validation, and state management |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'effort',
  description: 'Set effort level for model usage',
  argumentHint: '[low|medium|high|max|auto]',
  immediate: shouldInferenceConfigCommandBeImmediate(),
  load: () => import('./effort.js'),
}
```

**Effort levels:**

| Level | Description |
|-------|-------------|
| `low` | Quick, straightforward implementation |
| `medium` | Balanced approach with standard testing |
| `high` | Comprehensive implementation with extensive testing |
| `max` | Maximum capability with deepest reasoning (Opus 4.6 only) |
| `auto` | Use the default effort level for your model |

**Usage modes:**

| Invocation | Behavior |
|------------|----------|
| `/effort` | Shows current effort level |
| `/effort current` | Shows current effort level |
| `/effort status` | Shows current effort level |
| `/effort [level]` | Sets effort to the specified level |
| `/effort auto` | Resets effort to auto/default |
| `/effort unset` | Same as `auto` — clears effort setting |
| `/effort --help` | Shows full help with all level descriptions |

**Setting logic (`setEffortValue`):**
1. Converts the effort value to a persistable form via `toPersistableEffort`
2. Updates user settings via `updateSettingsForSource('userSettings', ...)`
3. Checks for environment variable override (`CLAUDE_CODE_EFFORT_LEVEL`)
4. If env var conflicts, warns the user that the env var takes precedence
5. Logs `tengu_effort_command` analytics event
6. Returns a message with description and session-only suffix if not persistable

**Display logic (`showCurrentEffort`):**
- Resolves effective effort: env override > app state > model default
- Shows `auto (currently <level>)` when no explicit level is set
- Shows `Current effort level: <value> (<description>)` when set

**Unsetting logic (`unsetEffortLevel`):**
- Clears effort from user settings
- Warns if `CLAUDE_CODE_EFFORT_LEVEL` env var still controls the session

**Validation (`executeEffort`):**
- Normalizes input to lowercase
- Accepts `auto` or `unset` to clear
- Validates against known effort levels via `isEffortLevel`
- Returns error message for invalid arguments

---

## `/fast`

**Purpose:** Toggle fast mode — a high-speed research preview mode with separate rate limits and premium billing.

**Source:** `restored-src/src/commands/fast/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with dynamic availability |
| `fast.tsx` | Fast mode picker, toggle logic, and state management |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'fast',
  description: `Toggle fast mode (${FAST_MODE_MODEL_DISPLAY} only)`,
  availability: ['claude-ai', 'console'],
  isEnabled: () => isFastModeEnabled(),
  isHidden: !isFastModeEnabled(),
  argumentHint: '[on|off]',
  immediate: shouldInferenceConfigCommandBeImmediate(),
  load: () => import('./fast.js'),
}
```

**Availability:** Only visible when fast mode is enabled (`isFastModeEnabled()`). Available in both `claude-ai` and `console` contexts.

**Usage modes:**

| Invocation | Behavior |
|------------|----------|
| `/fast` | Opens the interactive `FastModePicker` dialog |
| `/fast on` | Enables fast mode directly |
| `/fast off` | Disables fast mode directly |

**Fast mode picker (`FastModePicker`):**
- Displays current fast mode state with toggle control
- Shows unavailable reason if fast mode cannot be enabled
- Shows cooldown status with reset timer when rate-limited
- Displays pricing information for the fast mode model tier
- On confirm: applies the setting, logs analytics, shows result message
- On cancel: preserves original state
- Keybindings: `Enter` to confirm, `Tab`/arrows to toggle, `Esc` to cancel

**Apply logic (`applyFastMode`):**
1. Clears fast mode cooldown timer
2. Updates user settings (`fastMode: true` or clears to `undefined`)
3. When enabling:
   - Checks if current model supports fast mode
   - If not, automatically switches to the fast mode model
   - Sets `fastMode: true` in app state
4. When disabling:
   - Sets `fastMode: false` in app state

**Shortcut handler (`handleFastModeShortcut`):**
- Used for direct `/fast on` and `/fast off` invocations
- Checks availability before applying
- Returns formatted result message with icon, model change notice, and pricing

**Pre-flight:**
- Calls `prefetchFastModeStatus()` before showing picker to check org-level fast mode status

---

## `/permissions` (alias: `/allowed-tools`)

**Purpose:** Manage allow and deny tool permission rules.

**Source:** `restored-src/src/commands/permissions/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with `allowed-tools` alias |
| `permissions.tsx` | Entry point rendering the permission rule list |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'permissions',
  aliases: ['allowed-tools'],
  description: 'Manage allow & deny tool permission rules',
  load: () => import('./permissions.js'),
}
```

**Behavior:**
- Renders `PermissionRuleList` component for managing tool permissions
- On exit: calls `onDone` to dismiss
- On retry denials: creates a retry message and adds it to the message queue via `context.setMessages`

**Components used:**
- `PermissionRuleList` — interactive list of permission rules with add/edit/delete capabilities

---

## `/sandbox`

**Purpose:** Configure sandboxing for command execution — controls whether shell commands run in an isolated environment.

**Source:** `restored-src/src/commands/sandbox-toggle/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with dynamic status description |
| `sandbox-toggle.tsx` | Sandbox settings UI and exclude subcommand |

**Registration:**
```typescript
{
  name: 'sandbox',
  description: `${icon} ${statusText} (⏎ to configure)`,
  argumentHint: 'exclude "command pattern"',
  isHidden: !isSupportedPlatform() || !isPlatformInEnabledList(),
  immediate: true,
  type: 'local-jsx',
  load: () => import('./sandbox-toggle.js'),
}
```

**Dynamic description** reflects current state:
- `⚠ sandbox disabled` — missing dependencies
- `✓ sandbox enabled` — sandboxing active
- `✓ sandbox enabled (auto-allow)` — sandboxing with auto-allowed bash
- `○ sandbox disabled` — sandboxing off
- `, fallback allowed` — appended when unsandboxed commands are permitted
- `(managed)` — appended when settings are locked by policy

**Availability:** Hidden on unsupported platforms or when the platform is not in the enabled list.

**Usage modes:**

| Invocation | Behavior |
|------------|----------|
| `/sandbox` | Opens the interactive `SandboxSettings` dialog |
| `/sandbox exclude "pattern"` | Adds a command pattern to the excluded commands list |

**Pre-flight checks (in order):**
1. Platform support — errors on non-macOS/Linux/WSL2 platforms
2. Dependency check — validates sandboxing dependencies are installed
3. Platform enabled list — checks enterprise `enabledPlatforms` setting
4. Policy lock — checks if sandbox settings are overridden by higher-priority config

**Exclude subcommand:**
- Parses `exclude "command pattern"` from arguments
- Strips surrounding quotes from the pattern
- Calls `addToExcludedCommands(pattern)` to persist
- Reports the relative path to the local settings file

**Components used:**
- `SandboxSettings` — interactive sandbox configuration dialog

---

## `/reset-limits`

**Purpose:** Reset usage limits.

**Source:** `restored-src/src/commands/reset-limits/`

| File | Purpose |
|------|---------|
| `index.js` | Stub implementation (disabled and hidden) |

**Status:** This command is currently a stub — `isEnabled: () => false` and `isHidden: true`. It is not functional in the current codebase. Two exports are defined (`resetLimits` and `resetLimitsNonInteractive`) but both point to the same stub.

---

## `/mock-limits`

**Purpose:** Mock usage limits (likely for testing).

**Source:** `restored-src/src/commands/mock-limits/`

| File | Purpose |
|------|---------|
| `index.js` | Stub implementation (disabled and hidden) |

**Status:** This command is currently a stub — `isEnabled: () => false` and `isHidden: true`. It is not functional in the current codebase.

---

## `/rate-limit-options`

**Purpose:** Show options when a rate limit is reached — upgrade plan, enable extra usage, or wait.

**Source:** `restored-src/src/commands/rate-limit-options/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration (Claude AI subscribers only, hidden) |
| `rate-limit-options.tsx` | Rate limit options menu with dynamic choices |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'rate-limit-options',
  description: 'Show options when rate limit is reached',
  isEnabled: () => isClaudeAISubscriber(),
  isHidden: true, // Hidden from help - only used internally
  load: () => import('./rate-limit-options.js'),
}
```

**Availability:** Only enabled for Claude AI subscribers. Hidden from help — triggered internally when rate limits are hit.

**Menu options (dynamically assembled):**

| Option | Condition | Value |
|--------|-----------|-------|
| Request extra usage / Switch to extra usage / Add funds | Extra usage enabled, billing access available | `extra-usage` |
| Request more | Team/enterprise without billing access, org spend cap depleted | `extra-usage` |
| Upgrade your plan | Not Max 20x, not team/enterprise, upgrade feature enabled | `upgrade` |
| Stop and wait for limit to reset | Always available | `cancel` |

**Option ordering:**
- Controlled by `buyFirst` feature flag (`tengu_jade_anvil_4`)
- When `true`: action options appear before cancel
- When `false`: cancel appears first

**Behavior (`RateLimitOptionsMenu`):**
1. Fetches subscription type, rate limit tier, and account info
2. Checks Claude AI limits for overage status and disabled reasons
3. Dynamically builds available options based on subscription and billing state
4. Renders a `Dialog` with a `Select` component for option choice
5. On select:
   - `upgrade` — launches the upgrade command JSX
   - `extra-usage` — launches the extra usage command JSX
   - `cancel` — dismisses with `display: "skip"`
6. Logs analytics events for menu cancel and each selection type

**State checks used:**
- `subscriptionType` — `"max"`, `"team"`, `"enterprise"`, etc.
- `rateLimitTier` — e.g., `"default_claude_max_20x"`
- `claudeAiLimits.overageStatus` — `"rejected"`, `"allowed_warning"`, etc.
- `claudeAiLimits.overageDisabledReason` — `"out_of_credits"`, `"org_level_disabled_until"`, etc.
- `hasClaudeAiBillingAccess()` — whether the user can manage billing
- `hasExtraUsageEnabled` — whether extra usage is already active

**Components used:**
- `Dialog` — modal container with title "What do you want to do?"
- `Select` — option picker with visible option count

---

## Component Dependencies

Settings commands rely on these shared components and utilities:

| Component/Utility | Source | Used By |
|-------------------|--------|---------|
| `PermissionRuleList` | `components/permissions/rules/PermissionRuleList.js` | `/permissions` |
| `SandboxSettings` | `components/sandbox/SandboxSettings.js` | `/sandbox` |
| `Dialog` | `components/design-system/Dialog.js` | `/fast`, `/rate-limit-options` |
| `Select` | `components/CustomSelect/select.js` | `/rate-limit-options` |
| `FastIcon`, `getFastIconString` | `components/FastIcon.js` | `/fast` |
| `useAppState`, `useSetAppState` | `state/AppState.js` | `/effort`, `/fast` |
| `useMainLoopModel` | `hooks/useMainLoopModel.js` | `/effort` |
| `SandboxManager` | `utils/sandbox/sandbox-adapter.js` | `/sandbox` |
| `isFastModeEnabled`, `isFastModeSupportedByModel`, `getFastModeModel` | `utils/fastMode.js` | `/fast` |
| `updateSettingsForSource` | `utils/settings/settings.js` | `/effort`, `/fast` |
| `getEffortEnvOverride`, `toPersistableEffort`, `isEffortLevel` | `utils/effort.js` | `/effort` |
| `useClaudeAiLimits` | `services/claudeAiLimitsHook.js` | `/rate-limit-options` |
| `getSubscriptionType`, `getRateLimitTier`, `getOauthAccountInfo` | `utils/auth.js` | `/rate-limit-options` |
| `extraUsage`, `upgrade` | `commands/extra-usage/`, `commands/upgrade/` | `/rate-limit-options` |
