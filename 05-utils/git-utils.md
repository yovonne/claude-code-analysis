# Git Utilities

**Files:** `restored-src/src/utils/git/gitignore.ts`, `gitFilesystem.ts`, `gitConfigParser.ts`

## Purpose

Provides git-related operations without spawning git subprocesses where possible. Covers gitignore management, filesystem-based git state reading, and .git/config parsing.

## gitignore.ts

### Functions

- `isPathGitignored(filePath, cwd)` - Checks if a path is ignored via `git check-ignore`
  - Exit code 0 = ignored, 1 = not ignored, 128 = not in git repo
  - Returns `false` for code 128 (fail open outside git repo)

- `getGlobalGitignorePath()` - Returns path to `~/.config/git/ignore`

- `addFileGlobRuleToGitignore(filename, cwd)` - Adds a pattern to global gitignore
  - Skips if already ignored by any gitignore source
  - Creates directory if needed
  - Appends `**/filename` pattern

## gitFilesystem.ts

### Purpose

Reads git state directly from `.git/` files instead of spawning git subprocesses. Verified against git source code for correctness.

### Core Functions

- `resolveGitDir(startPath)` - Resolves actual .git directory, handling worktrees/submodules
  - Caches results per path
  - Handles `.git` as file (worktree `gitdir:` pointer) or directory

- `readGitHead(gitDir)` - Parses `.git/HEAD` to determine branch or detached SHA
  - Returns `{ type: 'branch', name }` or `{ type: 'detached', sha }`
  - Validates ref names and SHAs for safety

- `resolveRef(gitDir, ref)` - Resolves a git ref to commit SHA
  - Checks loose ref files first, then packed-refs
  - Follows symrefs
  - Handles worktrees via commonDir

- `getCommonDir(gitDir)` - Reads `commondir` file for worktree shared directory

- `readRawSymref(gitDir, refPath, branchPrefix)` - Reads symref and extracts branch name

### Cached Watcher Functions

- `getCachedBranch()` - Cached current branch name via `GitFileWatcher`
- `getCachedHead()` - Cached HEAD SHA via `GitFileWatcher`
- `getCachedRemoteUrl()` - Cached remote origin URL via `GitFileWatcher`
- `getCachedDefaultBranch()` - Cached default branch (main/master) via `GitFileWatcher`

### GitFileWatcher Class

Watches git files for changes and caches derived values:
- Watches `.git/HEAD`, `.git/config`, and current branch ref file
- Uses `fs.watchFile` with 1s poll interval (10ms in tests)
- Invalidates cache on file changes
- Handles branch switches by updating ref watcher
- Defers file I/O until scroll idle to avoid event loop contention

### Utility Functions

- `getHeadForDir(cwd)` - Reads HEAD SHA for arbitrary directory
- `readWorktreeHeadSha(worktreePath)` - Reads HEAD for a worktree directly
- `getRemoteUrlForDir(cwd)` - Reads remote URL for arbitrary directory
- `isShallowClone()` - Checks for shallow clone via `shallow` file existence
- `getWorktreeCountFromFs()` - Counts worktrees via `worktrees/` directory

### Safety Validations

- `isSafeRefName(name)` - Validates ref names against path traversal and shell injection
  - Allowlist: ASCII alphanumerics, `/`, `.`, `_`, `+`, `-`, `@`
  - Rejects `..`, leading `-`, empty path components, shell metacharacters

- `isValidGitSha(s)` - Validates SHA as 40 hex (SHA-1) or 64 hex (SHA-256) chars

## gitConfigParser.ts

### Purpose

Lightweight parser for `.git/config` files. Verified against git's `config.c`.

### Functions

- `parseGitConfigValue(gitDir, section, subsection, key)` - Reads a single config value
  - Case-insensitive section and key matching
  - Case-sensitive subsection matching
  - Handles quoted subsections with backslash escapes

- `parseConfigString(config, section, subsection, key)` - Parses from in-memory string

### Parser Features

- Section headers: `[section]` or `[section "subsection"]`
- Key-value pairs: `key = value`
- Quoted values with escape sequences (`\n`, `\t`, `\\`, `\"`)
- Inline comments (`#` or `;`) outside quotes
- Backslash line continuation

## Dependencies

- `cwd.js` - Current working directory
- `execFileNoThrow.js` - Git subprocess execution (gitignore only)
- `cleanupRegistry.js` - Watcher cleanup on exit
- `debug.js` / `log.js` - Error logging
