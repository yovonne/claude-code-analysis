# Settings Utilities

**Directory:** `restored-src/src/utils/settings/`

## Purpose

Manages Claude Code settings files across multiple sources with priority-based merging, schema validation, MDM/profile enforcement, and change detection.

## Settings Sources (Priority: Low to High)

1. **pluginSettings** - Plugin-provided base settings
2. **userSettings** - `~/.claude/settings.json` (or `cowork_settings.json`)
3. **projectSettings** - `.claude/settings.json` in project root
4. **localSettings** - `.claude/settings.local.json` in project root
5. **flagSettings** - SDK inline settings + flag file
6. **policySettings** - Managed settings (remote > MDM > file > HKCU)

## settings.ts

### Core Functions

- `getSettingsForSource(source)` - Returns settings for a specific source (cached)
- `getInitialSettings()` - Returns merged settings from all sources
- `getSettings_DEPRECATED()` - Alias for getInitialSettings (backward compat)
- `getSettingsWithSources()` - Returns effective settings plus per-source breakdown
- `getSettingsWithErrors()` - Returns merged settings with validation errors
- `updateSettingsForSource(source, settings)` - Updates settings for editable source
- `getSettingsFilePathForSource(source)` - Returns file path for a source
- `getSettingsRootPathForSource(source)` - Returns root directory for a source

### File Parsing

- `parseSettingsFile(path)` - Parses and validates a settings file (cached)
- `loadManagedFileSettings()` - Loads managed-settings.json + drop-in directory
- `getManagedFileSettingsPresence()` - Checks which managed file sources exist

### Merging

- `settingsMergeCustomizer(objValue, srcValue)` - lodash mergeWith customizer
  - Arrays: concatenate and deduplicate
  - Objects: deep merge
  - Undefined: delete key

### Security Checks

- `hasSkipDangerousModePermissionPrompt()` - Checks trusted sources for bypass acceptance
- `hasAutoModeOptIn()` - Checks trusted sources for auto-mode opt-in
- `getUseAutoModeDuringPlan()` - Checks if plan mode uses auto-mode semantics
- `getAutoModeConfig()` - Returns merged auto-mode classifier config
- `rawSettingsContainsKey(key)` - Checks if any raw settings file contains a key

### Managed Settings Keys

- `getManagedSettingsKeysForLogging(settings)` - Extracts keys for logging display
  - Expands permissions, sandbox, hooks to one level deep

## settingsCache.ts

### Caching Layers

1. **sessionSettingsCache** - Merged settings for entire session
2. **perSourceCache** - Per-source settings (Map<SettingSource, SettingsJson>)
3. **parseFileCache** - Per-file parsed results (Map<path, ParsedSettings>)
4. **pluginSettingsBase** - Plugin settings base layer

- `resetSettingsCache()` - Clears all caches (called on settings write)
- `getPluginSettingsBase()` / `setPluginSettingsBase()` - Plugin base layer access

## types.ts

### SettingsSchema

Comprehensive Zod v4 schema covering all settings fields:

- **Auth**: apiKeyHelper, awsCredentialExport, awsAuthRefresh, gcpAuthRefresh, xaaIdp
- **Model**: model, availableModels, modelOverrides, effortLevel, advisorModel
- **Permissions**: permissions (allow/deny/ask/defaultMode/disableBypassPermissionsMode/disableAutoMode/additionalDirectories)
- **MCP**: enabledMcpjsonServers, disabledMcpjsonServers, allowedMcpServers, deniedMcpServers, enableAllProjectMcpServers
- **Plugins**: enabledPlugins, extraKnownMarketplaces, strictKnownMarketplaces, blockedMarketplaces
- **Hooks**: hooks (PreToolUse, PostToolUse, Notification, etc.), disableAllHooks, allowManagedHooksOnly
- **Sandbox**: sandbox settings
- **Environment**: env variables
- **Git**: worktree (symlinkDirectories, sparsePaths), respectGitignore
- **UI**: outputStyle, language, terminalTitleFromRename, syntaxHighlightingDisabled
- **Features**: agent, fastMode, alwaysThinkingEnabled, promptSuggestionEnabled
- **Remote**: remote.defaultEnvironmentId, sshConfigs
- **Auto-mode**: autoMode (allow, soft_deny, environment), skipAutoPermissionPrompt
- **Enterprise**: strictPluginOnlyCustomization, allowManagedMcpServersOnly, allowedHttpHookUrls
- **Other**: cleanupPeriodDays, attribution, plansDirectory, voiceEnabled, assistant

### Sub-schemas

- `PermissionsSchema` - Permission rules and mode configuration
- `EnvironmentVariablesSchema` - Environment variable records
- `AllowedMcpServerEntrySchema` - MCP server allowlist entries
- `DeniedMcpServerEntrySchema` - MCP server denylist entries
- `ExtraKnownMarketplaceSchema` - Custom marketplace definitions

### Type Guards

- `isMcpServerNameEntry()` / `isMcpServerCommandEntry()` / `isMcpServerUrlEntry()`

## validation.ts

### Validation Functions

- `formatZodError(error, filePath)` - Converts Zod errors to ValidationError array
- `validateSettingsFileContent(content)` - Validates settings string content
- `filterInvalidPermissionRules(data, filePath)` - Removes invalid permission rules before schema validation
- `getValidationTip()` - Returns fix suggestions for common errors

### ValidationError Type

```typescript
{
  file?: string
  path: FieldPath  // dot notation
  message: string
  expected?: string
  invalidValue?: unknown
  suggestion?: string
  docLink?: string
  mcpErrorMetadata?: { scope, serverName?, severity? }
}
```

## managedPath.ts

### Platform-Specific Paths

- `getManagedFilePath()` - Returns managed settings directory:
  - macOS: `/Library/Application Support/ClaudeCode`
  - Windows: `C:\Program Files\ClaudeCode`
  - Linux: `/etc/claude-code`
- `getManagedSettingsDropInDir()` - Returns `managed-settings.d` subdirectory

## mdm/settings.ts

### MDM Settings Management

- `startMdmSettingsLoad()` - Kicks off async MDM/HKCU reads at startup
- `ensureMdmSettingsLoaded()` - Awaits in-flight MDM load
- `getMdmSettings()` - Returns admin-controlled MDM settings (cached)
- `getHkcuSettings()` - Returns HKCU registry settings (Windows, cached)
- `refreshMdmSettings()` - Fires fresh subprocess read
- `clearMdmSettingsCache()` - Clears MDM and HKCU caches
- `setMdmSettingsCache(mdm, hkcu)` - Updates session caches

### Policy Priority (First Source Wins)

1. Remote managed settings (highest)
2. Admin MDM (HKLM / macOS plist)
3. managed-settings.json + drop-ins
4. HKCU registry (lowest, user-writable)

## mdm/constants.ts

### MDM Constants

- `MACOS_PREFERENCE_DOMAIN` - `com.anthropic.claudecode`
- `WINDOWS_REGISTRY_KEY_PATH_HKLM` - `HKLM\SOFTWARE\Policies\ClaudeCode`
- `WINDOWS_REGISTRY_KEY_PATH_HKCU` - `HKCU\SOFTWARE\Policies\ClaudeCode`
- `WINDOWS_REGISTRY_VALUE_NAME` - `Settings`
- `PLUTIL_PATH` - `/usr/bin/plutil`
- `MDM_SUBPROCESS_TIMEOUT_MS` - 5000ms
- `getMacOSPlistPaths()` - Returns plist paths in priority order

## mdm/rawRead.ts

### Subprocess I/O

- `fireRawRead()` - Fires fresh subprocess reads for MDM settings
  - macOS: plutil for each plist path in parallel
  - Windows: reg query for HKLM and HKCU in parallel
  - Linux: returns empty
- `startMdmRawRead()` - Fires once at startup
- `getMdmRawReadPromise()` - Gets startup promise

## constants.ts

- `CLAUDE_CODE_SETTINGS_SCHEMA_URL` - JSON Schema reference URL
- `SettingSource` type union
- `EditableSettingSource` type (excludes policySettings, flagSettings)
- `getEnabledSettingSources()` - Returns active sources based on context

## changeDetector.ts

- Detects settings file changes on disk
- 30-minute poll interval for MDM settings refresh
- Triggers cache invalidation on change

## applySettingsChange.ts

- Applies settings changes to running session
- Handles context updates after settings modification

## allErrors.ts

- Aggregates all settings errors for display
- Used by /status and diagnostics

## validationTips.ts

- Provides human-readable tips for fixing validation errors
- Maps error codes to suggestions and documentation links

## schemaOutput.ts

- Generates JSON Schema output from SettingsSchema
- Used for validation error reporting

## internalWrites.ts

- Tracks internal file writes to prevent change detector false positives
- `markInternalWrite(path)` - Marks a path as internally written

## permissionValidation.ts

- `validatePermissionRule(ruleString)` - Validates a single permission rule string
- `PermissionRuleSchema` - Zod schema for permission rule validation

## toolValidationConfig.ts / validateEditTool.ts

- Tool-specific validation configuration
- Edit tool validation helpers

## pluginOnlyPolicy.ts

- Enforces strictPluginOnlyCustomization policy
- Blocks non-plugin customization sources for locked surfaces
