# Configuration Commands

## Overview

Configuration commands provide interactive interfaces for managing Claude Code's persistent settings, environment variables, model selection, theme, output style, and privacy controls. These commands render JSX-based UI components within the terminal.

---

## `/config` (alias: `/settings`)

**Purpose:** Open the main configuration panel for managing all settings.

**Source:** `restored-src/src/commands/config/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with alias `settings` |
| `config.tsx` | Entry point that renders the `Settings` component |

**Behavior:**
- Opens an interactive settings panel via the `Settings` component
- Defaults to the "Config" tab
- Accepts an `onClose` callback for dismissal

**Registration:**
```typescript
{
  aliases: ['settings'],
  type: 'local-jsx',
  name: 'config',
  description: 'Open config panel',
  load: () => import('./config.js'),
}
```

---

## `/env`

**Purpose:** Environment variable management.

**Source:** `restored-src/src/commands/env/`

| File | Purpose |
|------|---------|
| `index.js` | Stub implementation (disabled and hidden) |

**Status:** This command is currently a stub — `isEnabled: () => false` and `isHidden: true`. It is not functional in the current codebase.

---

## `/model`

**Purpose:** Set the AI model for Claude Code with an interactive picker or direct argument.

**Source:** `restored-src/src/commands/model/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with dynamic description showing current model |
| `model.tsx` | Model selection logic, validation, and picker UI |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'model',
  description: `Set the AI model for Claude Code (currently ${renderModelName(getMainLoopModel())})`,
  argumentHint: '[model]',
  immediate: shouldInferenceConfigCommandBeImmediate(),
  load: () => import('./model.js'),
}
```

**Usage modes:**

| Invocation | Behavior |
|------------|----------|
| `/model` | Opens the `ModelPicker` interactive UI |
| `/model [modelName]` | Sets the model directly without UI |
| `/model default` | Resets to the default model |
| `/model --help` | Shows inline help text |
| `/model -h` / `/model help` | Shows usage information |

**Model selection flow (`SetModelAndClose`):**
1. Checks if the model is allowed by organization policy (`isModelAllowed`)
2. Checks 1M context access for Opus and Sonnet models
3. Skips validation for the default model (`null`)
4. Recognizes known aliases without validation (`isKnownAlias`)
5. Validates custom models via `validateModel`
6. Updates app state (`mainLoopModel`, clears `mainLoopModelForSession`)
7. Handles fast mode compatibility — auto-disables if the new model doesn't support it
8. Displays billing notices for extra usage models

**Model picker flow (`ModelPickerWrapper`):**
- Renders `ModelPicker` component with current model and session override
- On select: updates model, handles fast mode toggling, logs analytics
- On cancel: displays current model and dismisses
- Shows fast mode notice when applicable

**Display logic (`ShowModelAndClose`):**
- Shows current model with effort level
- If a session override exists (from plan mode), shows both session and base model

**Model validation helpers:**
- `isKnownAlias(model)` — checks against `MODEL_ALIASES`
- `isOpus1mUnavailable(model)` — checks Opus 4.6 1M context access
- `isSonnet1mUnavailable(model)` — checks Sonnet 4.6 1M context access
- `renderModelLabel(model)` — formats model name with "(default)" suffix

---

## `/theme`

**Purpose:** Change the terminal theme via an interactive picker.

**Source:** `restored-src/src/commands/theme/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration |
| `theme.tsx` | Theme picker UI wrapper |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'theme',
  description: 'Change the theme',
  load: () => import('./theme.js'),
}
```

**Behavior:**
- Renders `ThemePicker` inside a `Pane` component with `color="permission"`
- On theme select: applies the theme via `setTheme()` and confirms
- On cancel: dismisses with "Theme picker dismissed" message
- Uses `skipExitHandling={true}` to manage its own exit behavior

---

## `/output-style`

**Purpose:** Deprecated — users should use `/config` instead.

**Source:** `restored-src/src/commands/output-style/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration (hidden) |
| `output-style.tsx` | Deprecation notice |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'output-style',
  description: 'Deprecated: use /config to change output style',
  isHidden: true,
  load: () => import('./output-style.js'),
}
```

**Behavior:**
- Displays a deprecation message directing users to `/config` or the settings file
- Changes take effect on the next session

---

## `/privacy-settings`

**Purpose:** View and update privacy settings (Grove data sharing controls).

**Source:** `restored-src/src/commands/privacy-settings/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration (consumer subscribers only) |
| `privacy-settings.tsx` | Privacy settings dialog logic |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'privacy-settings',
  description: 'View and update your privacy settings',
  isEnabled: () => isConsumerSubscriber(),
  load: () => import('./privacy-settings.js'),
}
```

**Availability:** Only enabled for consumer subscribers (`isConsumerSubscriber()`).

**Behavior:**
1. Checks if the user is qualified for Grove (`isQualifiedForGrove`)
2. If not qualified, shows a fallback message directing to `https://claude.ai/settings/data-privacy-controls`
3. Fetches current Grove settings and notice config in parallel
4. If the user has already accepted terms (`grove_enabled !== null`), shows `PrivacySettingsDialog` directly
5. If not, shows `GroveDialog` for first-time terms acceptance
6. On completion, fetches updated settings and reports the new state
7. Logs `tengu_grove_policy_toggled` analytics event when the setting changes

**Components used:**
- `GroveDialog` — terms acceptance flow for new users
- `PrivacySettingsDialog` — settings management for returning users

---

## Component Dependencies

Configuration commands rely on these shared components and utilities:

| Component/Utility | Source | Used By |
|-------------------|--------|---------|
| `Settings` | `components/Settings/Settings.js` | `/config` |
| `ModelPicker` | `components/ModelPicker.js` | `/model` |
| `ThemePicker` | `components/ThemePicker.js` | `/theme` |
| `Pane` | `components/design-system/Pane.js` | `/theme` |
| `GroveDialog`, `PrivacySettingsDialog` | `components/grove/Grove.js` | `/privacy-settings` |
| `useAppState`, `useSetAppState` | `state/AppState.js` | `/model`, `/privacy-settings` |
| `validateModel` | `utils/model/validateModel.js` | `/model` |
| `MODEL_ALIASES` | `utils/model/aliases.js` | `/model` |
| `isFastModeEnabled`, `isFastModeSupportedByModel` | `utils/fastMode.js` | `/model` |
| `getGroveSettings`, `isQualifiedForGrove` | `services/api/grove.js` | `/privacy-settings` |
