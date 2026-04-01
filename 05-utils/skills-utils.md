# Skills Utilities

## Purpose

The skills utilities module provides file watching and change detection for skill and command directories. It monitors user and project skill/command directories for changes (add, modify, delete) and triggers cache invalidation and reload when files change, ensuring Claude Code always uses the latest skill definitions.

## Location

- `restored-src/src/utils/skills/skillChangeDetector.ts` — File watcher and change detection for skills/commands

## Key Exports

### Module Object
- `skillChangeDetector`: Singleton object exposing `initialize`, `dispose`, `subscribe`, and `resetForTesting` methods

### Functions

#### `initialize(): Promise<void>`
Initializes file watching for skill and command directories. Watches user skills (`~/.claude/skills`), user commands (`~/.claude/commands`), project skills (`.claude/skills`), project commands (`.claude/commands`), and additional directories from `--add-dir` flags. Uses chokidar for file watching with Bun-specific polling workaround to avoid a PathWatcherManager deadlock. Registers a callback for dynamic skills loading.

#### `dispose(): Promise<void>`
Cleans up the file watcher, clears pending reload timers, and resets internal state.

#### `subscribe`: Signal subscription function for listening to skill change events.

#### `resetForTesting(overrides?): Promise<void>`
Resets internal state for testing. Accepts optional timing overrides for stability threshold, poll interval, reload debounce, and chokidar interval.

### Internal Constants
- `FILE_STABILITY_THRESHOLD_MS` (1000): Wait time for file writes to stabilize
- `FILE_STABILITY_POLL_INTERVAL_MS` (500): Polling interval for checking file stability
- `RELOAD_DEBOUNCE_MS` (300): Debounce rapid skill change events into a single reload
- `POLLING_INTERVAL_MS` (2000): Chokidar polling interval when `usePolling` is enabled
- `USE_POLLING`: Enabled when running under Bun to avoid FSWatcher deadlocks

## Dependencies

- `chokidar` — File watching library
- `../../skills/loadSkillsDir.js` — Skill cache management (`clearSkillCaches`, `getSkillsPath`, `onDynamicSkillsLoaded`)
- `../../commands.js` — Command cache management
- `../attachments.js` — Reset sent skill names
- `../cleanupRegistry.js` — Cleanup registration
- `../hooks.js` — Config change hook execution
- `../signal.js` — Event signaling

## Design Notes

- **Bun deadlock workaround**: Under Bun, chokidar uses stat() polling instead of native FSWatchers to avoid a PathWatcherManager deadlock (oven-sh/bun#27469) that occurs when closing watchers while events are being delivered.
- **Debouncing**: Rapid file changes (e.g., during git operations or auto-updates) are debounced into a single reload to prevent cascading cache clears that could deadlock the Bun event loop.
- **Batched hooks**: ConfigChange hooks fire once per batch of changed paths rather than per-file, avoiding redundant hook matcher invocations.
- **Dynamic skills**: When dynamic skills are loaded, memoization caches are cleared (but not the full command cache, which would wipe the just-loaded dynamic skills).
