# Plugins Service

## Purpose
Manages plugin lifecycle operations (install, uninstall, enable, disable, update) and background marketplace installations. Provides both CLI command wrappers and pure library functions usable by the interactive UI.

## Location
`restored-src/src/services/plugins/`

### Key Files
| File | Description |
|------|-------------|
| `pluginOperations.ts` | Core plugin operations (install, uninstall, enable, disable, update) — pure library functions |
| `PluginInstallationManager.ts` | Background plugin and marketplace installation manager for startup |
| `pluginCliCommands.ts` | CLI command wrappers that handle console output and process exit |

## Key Exports

### Functions (pluginOperations.ts)
- `installPluginOp(plugin, scope)`: Install a plugin from marketplace to specified scope (user/project/local)
- `uninstallPluginOp(plugin, scope, deleteDataDir)`: Uninstall a plugin from specified scope
- `enablePluginOp(plugin, scope)`: Enable a plugin at specified scope
- `disablePluginOp(plugin, scope)`: Disable a plugin at specified scope
- `disableAllPluginsOp()`: Disable all enabled plugins
- `updatePluginOp(plugin, scope)`: Update a plugin to latest version
- `setPluginEnabledOp(plugin, enabled, scope)`: Core enable/disable with full scope resolution
- `getPluginInstallationFromV2(pluginId)`: Get most relevant installation for a plugin from V2 data
- `isPluginEnabledAtProjectScope(pluginId)`: Check if plugin is enabled in project-level settings
- `assertInstallableScope(scope)`: Runtime scope validation
- `isInstallableScope(scope)`: Type guard for installable scopes
- `getProjectPathForScope(scope)`: Get project path for project/local scopes

### Functions (PluginInstallationManager.ts)
- `performBackgroundPluginInstallations(setAppState)`: Background startup checks and installations

### Functions (pluginCliCommands.ts)
- `installPlugin(plugin, scope)`: CLI install command with console output and telemetry
- `uninstallPlugin(plugin, scope, keepData)`: CLI uninstall command
- `enablePlugin(plugin, scope)`: CLI enable command
- `disablePlugin(plugin, scope)`: CLI disable command
- `disableAllPlugins()`: CLI disable-all command
- `updatePluginCli(plugin, scope)`: CLI update command

### Types
- `PluginScope`: `'user' | 'project' | 'local' | 'managed'`
- `InstallableScope`: `'user' | 'project' | 'local'` (excludes 'managed')
- `PluginOperationResult`: Result object with success, message, pluginId, scope, reverseDependents
- `PluginUpdateResult`: Result object with success, message, oldVersion, newVersion, alreadyUpToDate

### Constants
- `VALID_INSTALLABLE_SCOPES`: `['user', 'project', 'local']`
- `VALID_UPDATE_SCOPES`: `['user', 'project', 'local', 'managed']`

## Dependencies

### Internal Dependencies
- `marketplaceManager` — Plugin marketplace loading and lookup
- `pluginLoader` — Plugin caching and versioned cache management
- `installedPluginsManager` — V2 installation tracking on disk
- `pluginPolicy` — Organization policy enforcement
- `pluginIdentifier` — Plugin name/marketplace parsing
- `settings` — Reading/writing enabledPlugins across scopes
- `cacheUtils` — Cache clearing and version orphaning
- `dependencyResolver` — Reverse dependency detection

### External Dependencies
- `path` — File path manipulation

## Implementation Details

### Core Logic
The plugin service follows a **settings-first** architecture:

1. **Install**: Search marketplace → write settings (declares intent) → cache plugin + record version
2. **Uninstall**: Remove from settings → remove from installed_plugins_v2.json → clean up caches and data
3. **Enable/Disable**: Resolve plugin ID and scope from settings → write enabledPlugins → clear caches
4. **Update**: Non-inplace — download to temp → calculate version → copy to new versioned cache → update disk JSON

### Scope Resolution
Scopes follow precedence: `local > project > user > managed`. When auto-detecting scope, the most specific scope where the plugin is mentioned wins. The 'managed' scope is special — plugins can only be installed from managed-settings.json, not directly.

### Plugin Identifier Format
Plugins are identified as `name@marketplace`. Bare names are resolved by searching all editable scopes for matching keys.

### Reverse Dependencies
On uninstall/disable, the system warns (but does not block) if other enabled plugins depend on the target. Blocking would create tombstones with delisted plugins.

### Versioned Caching
Plugins are stored in versioned cache directories. Updates create new versioned paths; old versions are orphaned when no longer referenced.

## Data Flow

```
CLI/UI → pluginCliCommands.ts → pluginOperations.ts
                                    ↓
                           marketplaceManager (find plugin)
                                    ↓
                           settings (write enabledPlugins)
                                    ↓
                           pluginLoader (cache plugin)
                                    ↓
                           installedPluginsManager (record on disk)
```

### Background Installation Flow
```
Startup → PluginInstallationManager
              ↓
         diffMarketplaces (declared vs materialized)
              ↓
         reconcileMarketplaces (install missing/changed)
              ↓
         refreshActivePlugins (if new installs)
         OR set needsRefresh (if updates only)
```

## Integration Points
- **CLI**: `pluginCliCommands.ts` wraps operations with console output, telemetry, and process.exit
- **UI**: `pluginOperations.ts` functions are used directly by ManagePlugins.tsx
- **Startup**: `PluginInstallationManager.ts` runs background marketplace reconciliation
- **Policy**: `isPluginBlockedByPolicy` gates enable/install operations
- **Settings**: enabledPlugins in settings.json at each scope level

## Configuration
- Scopes: `user` (~/.claude/), `project` (.claude/), `local` (.claude/ local override), `managed` (remote-managed)
- Plugin data directories are cleaned on uninstall (configurable via `deleteDataDir` flag)

## Error Handling
- Operations return `PluginOperationResult` objects — no process.exit() or console writes in core ops
- CLI wrappers catch errors, log telemetry (`tengu_plugin_command_failed`), and exit with code 1
- Policy-blocked plugins return descriptive error messages
- Delisted plugins are handled via fallback to installed_plugins.json

## Testing
- `_reset*ForTesting` helpers are not present — tests use the pure operation functions directly
- Module-level state is minimal; most state lives in disk files

## Related Modules
- [Remote Managed Settings](./remote-managed-settings.md) — managed scope plugin installation
- [Policy Limits](./policy-limits-service.md) — organization policy enforcement
- [Analytics](./analytics-service.md) — plugin telemetry events

## Notes
- Core operations deliberately avoid side effects (no console output, no process.exit) for UI reuse
- The V2 installed_plugins.json format tracks installations independently of marketplace state
- Marketplace reconciliation at startup handles declared-but-not-materialized marketplaces
- Plugin options and secrets are cleaned up when the last scope is uninstalled
