# ConfigTool

## Purpose

ConfigTool provides Claude with the ability to read and modify its own configuration settings at runtime. It supports both global settings (stored in `~/.claude.json`) and project settings (stored in `.claude/settings.json`). The tool handles type coercion, async validation (e.g., model API checks), feature-flag-gated settings (voice, bridge mode, Kairos), and immediate AppState synchronization for real-time UI effect. Reading settings is auto-allowed; writing requires user permission.

## Location

- `restored-src/src/tools/ConfigTool/ConfigTool.ts` — Main tool definition and read/write logic (468 lines)
- `restored-src/src/tools/ConfigTool/prompt.ts` — Dynamic prompt generation from settings registry (94 lines)
- `restored-src/src/tools/ConfigTool/supportedSettings.ts` — Settings registry with types, validation, and options (212 lines)
- `restored-src/src/tools/ConfigTool/constants.ts` — Tool name constant (2 lines)
- `restored-src/src/tools/ConfigTool/UI.tsx` — Terminal UI rendering (38 lines)

## Key Exports

### From ConfigTool.ts

| Export | Description |
|--------|-------------|
| `ConfigTool` | The complete tool definition built via `buildTool()` |
| `Input` | Zod-inferred input type: `{ setting, value? }` |
| `Output` | Zod-inferred output type: `{ success, operation?, setting?, value?, previousValue?, newValue?, error? }` |

### From supportedSettings.ts

| Export | Description |
|--------|-------------|
| `SUPPORTED_SETTINGS` | Registry of all configurable settings with type, source, description, and validation |
| `isSupported(key)` | Checks if a setting key is in the registry |
| `getConfig(key)` | Returns the SettingConfig for a key |
| `getAllKeys()` | Returns all setting keys |
| `getOptionsForSetting(key)` | Returns valid option values for a setting |
| `getPath(key)` | Returns the nested path array for a setting |
| `SettingConfig` | Type: `{ source, type, description, path?, options?, getOptions?, appStateKey?, validateOnWrite?, formatOnRead? }` |

## Input/Output Schemas

### Input Schema

```typescript
{
  setting: string,   // Required: The setting key (e.g., "theme", "model", "permissions.defaultMode")
  value?: string | boolean | number,  // Optional: New value. Omit to get current value.
}
```

### Output Schema

```typescript
{
  success: boolean,
  operation?: 'get' | 'set',
  setting?: string,
  value?: unknown,           // Current value (for get operations)
  previousValue?: unknown,   // Value before change (for set operations)
  newValue?: unknown,        // Value after change (for set operations)
  error?: string,            // Error message if operation failed
}
```

## Supported Settings

### Global Settings (stored in `~/.claude.json`)

| Setting | Type | Options | Description |
|---------|------|---------|-------------|
| `theme` | string | Theme names (or theme settings with AUTO_THEME) | Color theme for the UI |
| `editorMode` | string | `EDITOR_MODES` | Key binding mode (e.g., emacs, vim) |
| `verbose` | boolean | true/false | Show detailed debug output |
| `preferredNotifChannel` | string | Notification channels | Preferred notification channel |
| `autoCompactEnabled` | boolean | true/false | Auto-compact when context is full |
| `fileCheckpointingEnabled` | boolean | true/false | Enable file checkpointing for code rewind |
| `showTurnDuration` | boolean | true/false | Show turn duration after responses |
| `terminalProgressBarEnabled` | boolean | true/false | Show OSC 9;4 progress indicator |
| `todoFeatureEnabled` | boolean | true/false | Enable todo/task tracking |
| `teammateMode` | string | `TEAMMATE_MODES` | How to spawn teammates (tmux, in-process, auto) |

### Project Settings (stored in `.claude/settings.json`)

| Setting | Type | Options | Description |
|---------|------|---------|-------------|
| `autoMemoryEnabled` | boolean | true/false | Enable auto-memory |
| `autoDreamEnabled` | boolean | true/false | Enable background memory consolidation |
| `model` | string | Model options (sonnet, opus, haiku, etc.) | Override the default model |
| `alwaysThinkingEnabled` | boolean | true/false | Enable extended thinking |
| `permissions.defaultMode` | string | `default`, `plan`, `acceptEdits`, `dontAsk`, `auto` | Default permission mode |
| `language` | string | Any language name | Preferred language for responses and voice dictation |

### Feature-Flag-Gated Settings

| Setting | Feature Flag | Type | Description |
|---------|-------------|------|-------------|
| `classifierPermissionsEnabled` | `TRANSCRIPT_CLASSIFIER` (ant only) | boolean | Enable AI-based Bash permission classification |
| `voiceEnabled` | `VOICE_MODE` | boolean | Enable voice dictation (hold-to-talk) |
| `remoteControlAtStartup` | `BRIDGE_MODE` | boolean | Enable Remote Control for all sessions |
| `taskCompleteNotifEnabled` | `KAIROS` or `KAIROS_PUSH_NOTIFICATION` | boolean | Push to mobile device when idle after completion |
| `inputNeededNotifEnabled` | `KAIROS` or `KAIROS_PUSH_NOTIFICATION` | boolean | Push to mobile device when waiting for input |
| `agentPushNotifEnabled` | `KAIROS` or `KAIROS_PUSH_NOTIFICATION` | boolean | Allow Claude to push when it deems appropriate |

## Read/Write Pipeline

### Execution Flow

```
1. INPUT RECEIVED
   - Model returns Config tool_use with { setting, value? }

2. SUPPORT CHECK
   - isSupported(setting): checks against SUPPORTED_SETTINGS registry
   - Voice settings: additional runtime gate via GrowthBook feature flag
   - Unknown setting → return error

3. GET OPERATION (value is undefined)
   a. Look up config for setting
   b. Get current value from source (global or settings)
   c. Apply formatOnRead if defined (e.g., model null → "default")
   d. Return { success: true, operation: 'get', setting, value }

4. SET OPERATION (value is provided)
   a. Look up config for setting
   b. Handle "default" special case (remoteControlAtStartup)
   c. Coerce and validate type (boolean string → true/false)
   d. Check against allowed options if defined
   e. Run async validation if defined (e.g., validateModel API check)
   f. Pre-flight checks for voice mode:
      - GrowthBook enabled check
      - Anthropic auth check
      - Voice stream availability
      - Recording availability
      - Voice dependencies (arecord tool)
      - Microphone permission
   g. Write to storage (global config or settings file)
   h. Notify change detector (for voice settings)
   i. Sync to AppState if appStateKey is defined
   j. Log analytics event
   k. Return { success: true, operation: 'set', setting, previousValue, newValue }
```

### Storage Sources

| Source | Storage | Access |
|--------|---------|--------|
| `global` | `~/.claude.json` via `getGlobalConfig()` / `saveGlobalConfig()` | Single-level keys |
| `settings` | `.claude/settings.json` via `updateSettingsForSource()` | Nested path support |

### Type Coercion

Boolean settings accept string values and coerce them:

| Input | Result |
|-------|--------|
| `"true"` (case-insensitive) | `true` |
| `"false"` (case-insensitive) | `false` |
| Other string | Error: "requires true or false" |
| Actual boolean | Passed through |

### Async Validation

Some settings have `validateOnWrite` functions that run asynchronously:

| Setting | Validation |
|---------|-----------|
| `model` | `validateModel()` — checks model API availability |

### AppState Synchronization

Settings with an `appStateKey` are synced to AppState immediately after writing:

| Setting | AppState Key |
|---------|-------------|
| `verbose` | `verbose` |
| `model` | `mainLoopModel` |
| `alwaysThinkingEnabled` | `thinkingEnabled` |

The `remoteControlAtStartup` setting uses a custom sync path because the config key differs from the AppState field name.

## Voice Mode Pre-Flight Checks

When enabling `voiceEnabled`, a comprehensive validation chain runs:

```
1. GrowthBook voice mode enabled?
   → No: error "Voice mode requires Claude.ai account" or "not available"

2. Voice stream available (isVoiceStreamAvailable)?
   → No: error "requires Claude.ai account"

3. Recording available (checkRecordingAvailability)?
   → No: error with reason

4. Voice dependencies (checkVoiceDependencies)?
   → No: error with install command if available

5. Microphone permission (requestMicrophonePermission)?
   → No: platform-specific guidance (Windows/Linux/macOS)
```

## Permission Handling

| Operation | Permission Decision |
|-----------|-------------------|
| GET (value omitted) | Auto-allow |
| SET (value provided) | Ask (user must approve) |

## Prompt Generation

The prompt is dynamically generated from the settings registry:

1. Iterates over `SUPPORTED_SETTINGS`
2. Skips `model` (has its own section with dynamic options)
3. Skips feature-flag-gated settings when the flag is off
4. Groups settings by source (global vs project)
5. Generates model section with current available options from `getModelOptions()`
6. Includes usage examples

## Error Handling

| Error Condition | Response |
|----------------|----------|
| Unknown setting | `{ success: false, error: 'Unknown setting: "<setting>"' }` |
| Invalid boolean value | `{ success: false, error: '<setting> requires true or false.' }` |
| Invalid option value | `{ success: false, error: 'Invalid value "<value>". Options: <list>' }` |
| Async validation failure | `{ success: false, error: <validation error> }` |
| Voice pre-flight failure | `{ success: false, error: <specific voice error> }` |
| Write error | `{ success: false, error: <error message> }` |

## Configuration

### Feature Flags

| Flag | Gated Settings |
|------|---------------|
| `VOICE_MODE` | `voiceEnabled` |
| `BRIDGE_MODE` | `remoteControlAtStartup` |
| `KAIROS` / `KAIROS_PUSH_NOTIFICATION` | `taskCompleteNotifEnabled`, `inputNeededNotifEnabled`, `agentPushNotifEnabled` |
| `TRANSCRIPT_CLASSIFIER` (ant only) | `classifierPermissionsEnabled` |
| `AUTO_THEME` | Extended theme options |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/config.js` | Global config get/save |
| `utils/settings/settings.js` | Settings management |
| `utils/model/modelOptions.js` | Model option listing |
| `utils/model/validateModel.js` | Model validation |
| `utils/theme.js` | Theme constants |
| `utils/configConstants.js` | Editor modes, notification channels, teammate modes |
| `voice/voiceModeEnabled.js` | Voice feature gate |
| `services/voiceStreamSTT.js` | Voice stream availability |
| `services/voice.js` | Recording and dependency checks |
| `utils/auth.js` | Auth status check |
| `services/analytics/` | Event logging |

### External

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |
| `bun:bundle` | Feature flag API |

## Data Flow

```
Model Request
    |
    v
ConfigTool Input { setting, value? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ SUPPORT CHECK                                                   │
│ - isSupported(setting)                                          │
│ - Feature flag gates (voice, bridge, kairos)                    │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  GET → auto-allow                                               │
│  SET → ask user                                                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ OPERATION                                                       │
│                                                                 │
│  GET:                                                           │
│    - Read from source (global/settings)                         │
│    - Apply formatOnRead                                         │
│    - Return value                                               │
│                                                                 │
│  SET:                                                           │
│    - Coerce type (boolean strings)                              │
│    - Validate against options                                  │
│    - Async validation (model API check)                        │
│    - Voice pre-flight checks (if voiceEnabled)                 │
│    - Write to storage                                          │
│    - Sync to AppState                                          │
│    - Log analytics event                                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { success, operation, setting, value/previousValue/newValue, error? }
    |
    v
Model Response (tool_result block)
```
