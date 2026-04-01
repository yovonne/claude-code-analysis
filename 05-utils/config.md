# Config Utility

**File:** `restored-src/src/utils/config.ts`

## Purpose

Manages global and project-level configuration storage, including user preferences, project metadata, API key responses, and feature flags. Provides file-based persistence with locking, caching, and backup mechanisms.

## Key Types

### GlobalConfig
Persisted in `~/.claude.json`. Contains:
- User preferences (theme, editor mode, auto-updates)
- Notification settings
- API key responses (approved/rejected)
- OAuth account info
- Feature flags and experiment tracking
- Usage statistics (memory count, prompt queue, etc.)
- Model and subscription caches
- Terminal and IDE configuration

### ProjectConfig
Per-project settings keyed by normalized path:
- `allowedTools` - Tools allowed for this project
- `mcpServers` - MCP server configurations
- Trust dialog acceptance (`hasTrustDialogAccepted`)
- Session metrics (cost, duration, tokens)
- Worktree session management
- Remote control spawn mode

## Key Functions

### Global Config

- `getGlobalConfig()` - Returns cached global config with freshness watcher
- `saveGlobalConfig(updater)` - Thread-safe update via lock file with backup
- `getCustomApiKeyStatus(truncatedKey)` - Returns approved/rejected/new status
- `checkHasTrustDialogAccepted()` - Checks trust acceptance for cwd and parents
- `isPathTrusted(dir)` - Checks trust for an arbitrary directory
- `enableConfigs()` - Enables config reading after bootstrap
- `getGlobalConfigWriteCount()` - Returns session write count for diagnostics

### Trust Management

- `checkHasTrustDialogAccepted()` - Latch-once true check, re-evaluates false
- `computeTrustDialogAccepted()` - Walks cwd parents and project path for trust
- `isPathTrusted(dir)` - Walks up from arbitrary directory for trust

### File Operations

- `saveConfigWithLock()` - Acquires file lock, reads current, merges, writes with backup
- `saveConfig()` - Writes filtered config (removes defaults) with 0o600 permissions
- `getConfig()` - Reads and parses config file with default fallback
- `writeThroughGlobalConfigCache()` - Updates cache after write to skip re-read

## Caching Strategy

- **In-memory cache**: `globalConfigCache` with mtime tracking
- **Freshness watcher**: `fs.watchFile` polls every 1s for external changes
- **Write-through**: Own writes update cache immediately with overshoot mtime
- **Auth-loss guard**: Refuses to write default config over cached config with auth state

## Security Considerations

- Config files written with 0o600 permissions (owner read/write only)
- Lock file prevents concurrent write corruption
- Timestamped backups (max 5) in `~/.claude/backups/` with 60s minimum interval
- Auth-loss prevention: detects when re-read returns defaults that would wipe OAuth/onboarding state
- Trust dialog: project settings cannot auto-accept trust (only user/local/flag/policy sources)

## Constants

- `GLOBAL_CONFIG_KEYS` - List of all top-level global config keys
- `PROJECT_CONFIG_KEYS` - List of editable project config keys
- `CONFIG_WRITE_DISPLAY_THRESHOLD` - 20 writes triggers warning
- `CONFIG_FRESHNESS_POLL_MS` - 1000ms poll interval for file watcher

## Dependencies

- `lockfile.ts` - File locking for concurrent access
- `envUtils.ts` - Config directory paths
- `fsOperations.ts` - File system abstraction
- `git.ts` - Git root detection for project path
- `settings/settings.ts` - Settings integration
