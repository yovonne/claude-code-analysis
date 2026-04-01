# Sandbox Utilities

## Purpose

The sandbox utilities module provides an adapter layer between Claude Code and the `@anthropic-ai/sandbox-runtime` package. It converts Claude Code settings into sandbox runtime configuration, manages sandbox lifecycle, handles path resolution for permission rules, and provides UI utilities for sandbox violation display.

## Location

- `restored-src/src/utils/sandbox/sandbox-adapter.ts` — Main sandbox manager adapter
- `restored-src/src/utils/sandbox/sandbox-ui-utils.ts` — UI helper for violation tag removal

## Key Exports

### Sandbox Adapter (`sandbox-adapter.ts`)

#### Path Resolution
- `resolvePathPatternForSandbox(pattern, source)`: Resolve Claude Code-specific path patterns for sandbox-runtime:
  - `//path` → absolute from filesystem root (`/path`)
  - `/path` → relative to settings file directory
  - `~/path`, `./path`, `path` → passed through for sandbox-runtime to handle
- `resolveSandboxFilesystemPath(pattern, source)`: Resolve paths from `sandbox.filesystem.*` settings with standard path semantics (`/path` = absolute, not settings-relative)

#### Settings Conversion
- `convertToSandboxRuntimeConfig(settings)`: Convert Claude Code settings to `SandboxRuntimeConfig` format:
  - Extracts network domains from WebFetch permission rules
  - Builds filesystem allow/deny lists from Edit/Read rules and `sandbox.filesystem.*` settings
  - Always includes current directory and Claude temp directory as writable
  - Denies writes to settings.json files, `.claude/skills`, and bare git repo files
  - Includes `--add-dir` directories in allowWrite
  - Configures ripgrep from settings or bundled binary

#### Policy Checks
- `shouldAllowManagedSandboxDomainsOnly()`: Check if only managed domains should be used
- `areSandboxSettingsLockedByPolicy()`: Check if sandbox settings are overridden by policy/flag settings

#### Platform Support
- `isSupportedPlatform()`: Memoized check for platform support (macOS, Linux, WSL2+)
- `isPlatformInEnabledList()`: Check if current platform is in `enabledPlatforms` list (undocumented enterprise setting)
- `getLinuxGlobPatternWarnings()`: Return warnings for glob patterns that won't work on Linux/WSL (bubblewrap limitation)

#### Sandbox Manager (`SandboxManager` object)
Implements `ISandboxManager` interface:

**Lifecycle**
- `initialize(sandboxAskCallback?)`: Initialize sandbox with config, subscribe to settings changes, detect git worktrees
- `reset()`: Clear state, memoization caches, and reset base manager
- `refreshConfig()`: Immediately update sandbox config from current settings

**Status Checks**
- `isSandboxingEnabled()`: Check platform support, dependencies, enabledPlatforms, and enabled setting
- `isSandboxEnabledInSettings()`: Check `sandbox.enabled` setting directly
- `isSandboxRequired()`: Check if sandbox is enabled AND `failIfUnavailable` is true
- `getSandboxUnavailableReason()`: Return human-readable reason why sandbox can't run (when explicitly enabled)
- `checkDependencies()`: Memoized dependency check
- `isAutoAllowBashIfSandboxedEnabled()`: Check `autoAllowBashIfSandboxed` setting
- `areUnsandboxedCommandsAllowed()`: Check `allowUnsandboxedCommands` setting

**Configuration**
- `setSandboxSettings(options)`: Update local sandbox settings (enabled, autoAllowBashIfSandboxed, allowUnsandboxedCommands)
- `getExcludedCommands()`: Get commands excluded from sandboxing
- `addToExcludedCommands(command, permissionUpdates?)`: Add command to excluded list with pattern extraction from permission suggestions

**Execution**
- `wrapWithSandbox(command, binShell?, customConfig?, abortSignal?)`: Wrap command with sandbox enforcement

**Delegated to Base Manager**
- `getFsReadConfig()`, `getFsWriteConfig()`, `getNetworkRestrictionConfig()`, `getIgnoreViolations()`, `getAllowUnixSockets()`, `getAllowLocalBinding()`, `getEnableWeakerNestedSandbox()`, `getProxyPort()`, `getSocksProxyPort()`, `getLinuxHttpSocketPath()`, `getLinuxSocksSocketPath()`, `waitForNetworkInitialization()`, `getSandboxViolationStore()`, `annotateStderrWithSandboxFailures()`, `cleanupAfterCommand()`

#### Security: Bare Git Repo Scrubbing
- `scrubBareGitRepoFiles()`: Delete bare-repo files (HEAD, objects, refs, hooks, config) planted at cwd during sandboxed commands, before unsandboxed git runs. Prevents sandbox escape via git's `is_git_directory()` detection.

#### Worktree Detection
- `detectWorktreeMainRepoPath(cwd)`: Detect if cwd is a git worktree and resolve the main repo path. Cached for the session.

### UI Utilities (`sandbox-ui-utils.ts`)

#### Functions
- `removeSandboxViolationTags(text)`: Remove `<sandbox_violations>...</sandbox_violations>` XML blocks from text for clean UI display

## Dependencies

- `@anthropic-ai/sandbox-runtime` — Core sandbox runtime package
- `../../bootstrap/state.js` — CWD, original CWD, additional directories
- `../settings/` — Settings system (reading, updating, change detection)
- `../permissions/` — Permission rules
- `../platform.js` — Platform detection
- `../path.js` — Path expansion
- `../ripgrep.js` — Ripgrep command resolution
- `../errors.js`, `../debug.js` — Error handling and logging

## Design Notes

- **Settings-to-config conversion**: The `convertToSandboxRuntimeConfig` function is the central translation layer, mapping Claude Code's permission rules, network settings, and filesystem policies into the sandbox-runtime's configuration format. It iterates through all setting sources (policy, project, user, local) to resolve paths correctly.
- **Security hardening**: Multiple defense layers prevent sandbox escape:
  - Settings files are always write-denied
  - `.claude/skills` directories are write-denied (same privilege as commands/agents)
  - Bare git repo files are scrubbed post-command to prevent git-based escapes
  - Git worktree main repo paths are allowed for write access (needed for index.lock)
- **Dynamic config updates**: Settings changes trigger automatic sandbox config updates via `settingsChangeDetector.subscribe()`. The `refreshConfig()` function provides synchronous immediate updates after permission changes.
- **Platform-specific behavior**: Linux/WSL doesn't support glob patterns in bubblewrap, so `getLinuxGlobPatternWarnings()` surfaces this limitation to users. The `enabledPlatforms` setting allows enterprises to restrict sandbox to specific platforms during rollout.
- **Memoization strategy**: `checkDependencies()` and `isSupportedPlatform()` are memoized using the settings object as cache key — new settings object = cache miss, ensuring auto-invalidation on config changes.
- **Network policy enforcement**: The `wrappedCallback` in `initialize()` enforces `allowManagedDomainsOnly` at the ask level, ensuring all code paths (REPL, print/SDK) respect the policy.
