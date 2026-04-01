# Plugins System

## Purpose

The plugins system provides a modular extension architecture for Claude Code. Built-in plugins ship with the CLI and can be enabled/disabled by users via the `/plugin` UI. They can provide multiple components including skills, hooks, and MCP servers. The system distinguishes between built-in plugins (shipped with CLI) and marketplace plugins (downloaded from external sources).

## Location

- `restored-src/src/plugins/builtinPlugins.ts` — Built-in plugin registry and management
- `restored-src/src/plugins/bundled/index.ts` — Built-in plugin initialization scaffolding

## Key Exports

### Functions (builtinPlugins.ts)

- `registerBuiltinPlugin(definition)`: Registers a built-in plugin definition in the internal map. Called during startup from `initBuiltinPlugins()`.
- `isBuiltinPluginId(pluginId)`: Checks if a plugin ID ends with `@builtin` suffix.
- `getBuiltinPluginDefinition(name)`: Retrieves a specific plugin definition by name for UI display without marketplace lookup.
- `getBuiltinPlugins()`: Returns `{ enabled: LoadedPlugin[], disabled: LoadedPlugin[] }` split by user settings and `defaultEnabled` fallback. Plugins with `isAvailable() === false` are omitted entirely.
- `getBuiltinPluginSkillCommands()`: Extracts skill commands from enabled built-in plugins as `Command[]` objects.
- `clearBuiltinPlugins()`: Clears the registry for test isolation.

### Functions (bundled/index.ts)

- `initBuiltinPlugins()`: Initialization function called at CLI startup. Currently empty — scaffolding for future migration of toggleable skills to built-in plugins.

### Constants

- `BUILTIN_MARKETPLACE_NAME`: `'builtin'` — sentinel value used in plugin IDs (`{name}@builtin`) and path fields.

### Types (referenced)

- `BuiltinPluginDefinition`: Plugin definition with skills, hooks, MCP servers, availability check, and default enabled state.
- `LoadedPlugin`: Runtime plugin object with manifest, enabled state, hooks config, and MCP server list.

## Dependencies

### Internal Dependencies

- `../skills/bundledSkills.ts` — `BundledSkillDefinition` type for plugin-provided skills
- `../types/plugin.js` — `BuiltinPluginDefinition` and `LoadedPlugin` types
- `../utils/settings/settings.js` — `getSettings_DEPRECATED()` for user plugin preferences
- `../commands.js` — `Command` type for skill command conversion

## Implementation Details

### Core Logic

The built-in plugin system operates as a write-once registry with runtime enable/disable evaluation:

1. **Registration phase** — At startup, `initBuiltinPlugins()` calls `registerBuiltinPlugin()` for each built-in plugin. Currently no plugins are registered (empty scaffolding).

2. **Resolution phase** — `getBuiltinPlugins()` iterates the registry, evaluates `isAvailable()` checks, reads user preferences from settings, and classifies each plugin as enabled or disabled.

3. **Skill extraction** — `getBuiltinPluginSkillCommands()` converts enabled plugin skill definitions into `Command` objects compatible with the command system.

### Plugin ID Convention

Built-in plugins use the format `{name}@builtin` to distinguish them from marketplace plugins (`{name}@{marketplace}`). The `@builtin` suffix is checked by `isBuiltinPluginId()` and used as a sentinel in the `LoadedPlugin.path` field (since built-in plugins have no filesystem path).

### Enabled State Resolution

```
User setting exists?
    ├── Yes → use user setting (true/false)
    └── No → use definition.defaultEnabled ?? true
```

The default is `true` — built-in plugins are enabled unless explicitly disabled by the user or the plugin specifies `defaultEnabled: false`.

### Availability Check

Plugins can define an `isAvailable()` callback that returns `false` to hide the plugin entirely (not just disable it). This is checked before enabled-state resolution, so unavailable plugins don't appear in either the enabled or disabled list.

### Skill-to-Command Conversion

`skillDefinitionToCommand()` maps a `BundledSkillDefinition` to a `Command` object:

- `source` and `loadedFrom` are set to `'bundled'` (not `'builtin'`) to keep skills visible in the Skill tool's listing and analytics
- `isBuiltin` tracking is on `LoadedPlugin.isBuiltin` instead
- `isEnabled` defaults to `() => true`
- `isHidden` is the inverse of `userInvocable`
- `contentLength` is 0 (not applicable for bundled skills)

## Data Flow

```
CLI Startup
    ↓
initBuiltinPlugins()
    ↓ registerBuiltinPlugin() for each plugin
BUILTIN_PLUGINS Map populated
    ↓
getBuiltinPlugins()
    ↓
├── Check isAvailable() → skip if false
├── Read user settings → enabledPlugins[pluginId]
├── Resolve enabled state (user > default > true)
└── Classify as enabled[] or disabled[]
    ↓
getBuiltinPluginSkillCommands()
    ↓
├── Filter to enabled plugins only
├── Extract skills from each plugin
└── Convert to Command[] via skillDefinitionToCommand()
    ↓
Commands available to model via Skill tool
```

## Integration Points

- **Command system** — Plugin skills become slash commands (`/skill-name`)
- **Settings system** — User preferences stored in `settings.enabledPlugins[pluginId]`
- **Plugin UI** — `/plugin` command displays built-in plugins under a "Built-in" section
- **Skill tool** — Plugin skills appear in the model's available tool listing
- **MCP system** — Plugins can provide MCP server configurations
- **Hooks system** — Plugins can define hook configurations

## Configuration

| Setting Path | Type | Purpose |
|---|---|---|
| `settings.enabledPlugins['{name}@builtin']` | `boolean \| undefined` | User's enable/disable preference for a built-in plugin |

| Plugin Definition Field | Type | Default | Purpose |
|---|---|---|---|
| `name` | `string` | — | Plugin identifier |
| `description` | `string` | — | User-facing description |
| `version` | `string` | — | Plugin version |
| `defaultEnabled` | `boolean` | `true` | Whether enabled by default |
| `isAvailable` | `() => boolean` | — | Runtime availability check |
| `skills` | `BundledSkillDefinition[]` | — | Skills provided by plugin |
| `hooks` | `HooksSettings` | — | Hook configurations |
| `mcpServers` | `MCPServer[]` | — | MCP server configurations |

## Error Handling

- Missing settings: `getSettings_DEPRECATED()` may return undefined; handled gracefully with optional chaining
- Invalid plugin definitions: Registry uses a Map for O(1) lookups; missing names return `undefined`
- Test isolation: `clearBuiltinPlugins()` resets the registry between test runs

## Testing

- `clearBuiltinPlugins()` provides test isolation for the registry
- `getBuiltinPluginDefinition()` allows direct access for unit testing individual plugins
- The enabled/disabled split is deterministic based on settings, making it straightforward to test with mocked settings

## Related Modules

- [Skills System](./skills-system.md) — Plugins provide skills as one of their components
- [Plugins Service](../04-services/plugins-service.md) — Service layer for marketplace plugin management
- [Settings Utils](../05-utils/settings-utils.md) — Plugin preference storage
- [Command System](../01-core-modules/command-system.md) — Plugin skills become commands

## Notes

- The built-in plugin system is currently scaffolding — `initBuiltinPlugins()` is empty and no plugins are registered
- The architecture is designed for future migration of bundled skills that should be user-toggleable
- Built-in plugins differ from bundled skills in that they appear in the `/plugin` UI and can be toggled on/off by users
- The distinction between `'bundled'` (source/loadedFrom) and `'builtin'` (LoadedPlugin.isBuiltin) is intentional: `'bundled'` keeps skills visible in tool listings while `'builtin'` tracks the user-toggleable aspect
- Plugin IDs use `@builtin` as the marketplace name, distinguishing them from actual marketplace sources like `claude-code-marketplace` or `claude-plugins-official`
