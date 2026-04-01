# Effort and Fast Mode

## Purpose

Two related but independent feature utilities: **Effort** controls the reasoning depth/computation level sent to the API (low/medium/high/max), while **Fast Mode** ("penguin mode") is a speed optimization that uses a faster response path. Both are subscription-gated and model-dependent features with complex availability logic.

## Location

- `restored-src/src/utils/effort.ts` — Effort level management, model support, defaults, persistence
- `restored-src/src/utils/fastMode.ts` — Fast mode availability, runtime state, cooldown, org status, prefetch

## Key Exports

### Effort (`effort.ts`)

#### Types
- `EffortLevel`: `'low' | 'medium' | 'high' | 'max'`
- `EffortValue`: `EffortLevel | number` — numeric values are model-default only and not persisted
- `OpusDefaultEffortConfig`: `{ enabled, dialogTitle, dialogDescription }` — remote config for Opus default effort dialog

#### Constants
- `EFFORT_LEVELS`: `['low', 'medium', 'high', 'max']`

#### Model Support
- `modelSupportsEffort(model)`: Check if a model supports the effort parameter. Opus 4.6 and Sonnet 4.6 support it; haiku and older variants don't. Unknown models default to true on first-party, false on third-party.
- `modelSupportsMaxEffort(model)`: Check if a model supports `'max'` effort. Opus 4.6 only for public models.

#### Parsing and Conversion
- `isEffortLevel(value)`: Type guard for valid effort level strings
- `parseEffortValue(value)`: Parse unknown input into EffortValue (handles strings, numbers, null)
- `isValidNumericEffort(value)`: Check if a number is a valid numeric effort (must be integer)
- `convertEffortValueToLevel(value)`: Convert EffortValue to EffortLevel. Numeric values map to ranges (≤50→low, ≤85→medium, ≤100→high, >100→max) for ants only.
- `toPersistableEffort(value)`: Convert to persistable form. Only `'low'`, `'medium'`, `'high'` are persistable for external users; ants can also persist `'max'`.

#### Resolution
- `getInitialEffortSetting()`: Get effort from initial settings, filtered through `toPersistableEffort`
- `resolveAppliedEffort(model, appStateEffortValue)`: Resolve the effort value actually sent to the API. Precedence: env `CLAUDE_CODE_EFFORT_LEVEL` → appState.effortValue → model default. Downgrades `'max'` to `'high'` on non-Opus-4.6 models.
- `getDisplayedEffortLevel(model, appStateEffort)`: Resolve effort level to show user. Wraps `resolveAppliedEffort` with `'high'` fallback (what the API uses when no effort param is sent).
- `getEffortSuffix(model, effortValue)`: Build the ` with {level} effort` suffix shown in Logo/Spinner. Empty string if no effort set.
- `getDefaultEffortForModel(model)`: Get default effort for a model. Opus 4.6 defaults to `'medium'` for Pro/Max/Team subscribers; ultrathink-enabled models default to `'medium'`.
- `resolvePickerEffortPersistence(picked, modelDefault, priorPersisted, toggledInPicker)`: Decide what effort to persist when user selects a model in ModelPicker. Keeps explicit choices sticky while letting defaults fall through.

#### Environment Overrides
- `getEffortEnvOverride()`: Read `CLAUDE_CODE_EFFORT_LEVEL` env var. `'unset'` or `'auto'` returns `null`.

#### Descriptions
- `getEffortLevelDescription(level)`: Human-readable description for each effort level
- `getEffortValueDescription(value)`: Description for both string and numeric effort values
- `getOpusDefaultEffortConfig()`: Get remote config for the Opus default effort recommendation dialog

### Fast Mode (`fastMode.ts`)

#### Types
- `FastModeRuntimeState`: `{ status: 'active' } | { status: 'cooldown', resetAt, reason }`
- `FastModeDisabledReason`: `'free' | 'preference' | 'extra_usage_disabled' | 'network_error' | 'unknown'`
- `CooldownReason`: `'rate_limit' | 'overloaded'`

#### Constants
- `FAST_MODE_MODEL_DISPLAY`: `'Opus 4.6'`

#### Availability Checks
- `isFastModeEnabled()`: Check if fast mode is enabled via env var (`CLAUDE_CODE_DISABLE_FAST_MODE`)
- `isFastModeAvailable()`: Check if fast mode is available (enabled + no unavailable reason)
- `getFastModeUnavailableReason()`: Get the reason fast mode is unavailable, or null. Checks: Statsig gate, bundled mode requirement, SDK session restriction, first-party provider requirement, org status.
- `isFastModeSupportedByModel(modelSetting)`: Check if the current model supports fast mode (Opus 4.6 only)
- `getFastModeModel()`: Get the fast mode model string (includes `[1m]` suffix if Opus 1m merge is enabled)
- `getInitialFastModeModel(model)`: Get the initial fast mode setting based on model, availability, and user preferences

#### Runtime State
- `getFastModeRuntimeState()`: Get current runtime state. Auto-resets if cooldown has expired.
- `triggerFastModeCooldown(resetTimestamp, reason)`: Enter cooldown after rate limit or overload. Emits analytics and notifies listeners.
- `clearFastModeCooldown()`: Manually clear cooldown.
- `isFastModeCooldown()`: Check if currently in cooldown.
- `getFastModeState(model, fastModeUserEnabled)`: Get aggregate state: `'off' | 'cooldown' | 'on'`

#### API Rejection Handlers
- `handleFastModeRejectedByAPI()`: Called when API rejects fast mode (e.g., 400 "not enabled for your organization"). Permanently disables fast mode.
- `handleFastModeOverageRejection(reason)`: Called when 429 indicates extra usage is not available. Permanently disables unless out of credits.
- `onFastModeOverageRejection`: Subscribe to overage rejection events
- `onOrgFastModeChanged`: Subscribe to org-level fast mode status changes
- `onCooldownTriggered`, `onCooldownExpired`: Subscribe to cooldown lifecycle events

#### Org Status Prefetch
- `prefetchFastModeStatus()`: Fetch org fast mode status from API with 30s minimum interval. Handles auth errors with token refresh. Falls back to cache on failure.
- `resolveFastModeStatusFromCache()`: Resolve orgStatus from persisted cache without network calls (used when startup prefetches are throttled)

#### Overage Messages
- `getOverageDisabledMessage(reason)`: Get user-facing message for various overage rejection reasons

## Design Notes

- **Effort precedence chain**: env `CLAUDE_CODE_EFFORT_LEVEL` → `appState.effortValue` → model default. The env override can explicitly unset effort with `'unset'` or `'auto'`.
- **Max effort is session-scoped** for external users — it cannot be persisted to settings.json. Only internal users (ants) can persist `'max'`.
- **Numeric effort values** are model-default only and never persisted. They're converted to named levels via `convertEffortValueToLevel` for display.
- **Fast mode cooldown** is tracked separately from user preference — it reflects the actual operational state (actively sending fast speed vs. in cooldown after rate limit).
- **Fast mode org status** is fetched from the API and cached. On failure, ants default to enabled; external users fall back to cached value or disable with `network_error` reason.
- **Fast mode prefetch** uses a 30-second minimum interval to avoid excessive API calls. In-flight requests are deduplicated.
- **Overage vs org disabling** are distinct: overage means extra usage billing isn't available; org disabling means the organization has turned off fast mode entirely.
