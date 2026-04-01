# Shell Utilities

**Directory:** `restored-src/src/utils/shell/`

## Purpose

Provides shell abstraction for command execution, including shell provider interfaces, platform-specific shell implementations, output limits, and read-only command validation.

## shellProvider.ts

### ShellProvider Interface

```typescript
{
  type: ShellType  // 'bash' | 'powershell'
  shellPath: string
  detached: boolean

  buildExecCommand(command, opts): Promise<{ commandString, cwdFilePath }>
  getSpawnArgs(commandString): string[]
  getEnvironmentOverrides(command): Promise<Record<string, string>>
}
```

### Shell Types

- `SHELL_TYPES` - `['bash', 'powershell']`
- `DEFAULT_HOOK_SHELL` - `'bash'`
- `SHELL_TOOL_NAMES` - Array of shell tool names (BashTool, PowerShellTool)

### Key Functions

- `isPowerShellToolEnabled()` - Runtime gate for PowerShellTool
  - Windows-only, ant defaults on (opt-out), external defaults off (opt-in)
  - Checks `CLAUDE_CODE_USE_POWERSHELL_TOOL` env var

## bashProvider.ts

### Bash Shell Provider

- `createBashShellProvider(shellPath, options?)` - Creates a Bash ShellProvider
  - Creates shell snapshot for environment capture
  - Builds execution command with:
    1. Source snapshot file (environment state)
    2. Source session environment script
    3. Disable extended glob patterns (security)
    4. Eval-wrapped command
    5. PWD tracking via `pwd -P`
  - Handles sandbox mode with POSIX tmpdir paths
  - Applies CLAUDE_CODE_SHELL_PREFIX if set
  - TMUX socket isolation for ant users

### Security Features

- `getDisableExtglobCommand(shellPath)` - Disables bash extglob and zsh EXTENDED_GLOB
  - Prevents malicious filename exploitation
  - Handles shell prefix wrapper (dual-shell mode)

### Environment Overrides

- TMUX env var override for isolated socket
- TMPDIR/CLAUDE_CODE_TMPDIR for sandbox mode
- TMPPREFIX for zsh heredoc support
- Session env vars from /env command

## powershellProvider.ts

### PowerShell Shell Provider

- `createPowerShellProvider(shellPath)` - Creates a PowerShell ShellProvider
  - Uses UTF-16LE base64 encoding for -EncodedCommand
  - Avoids shell-quoting corruption issues
  - Exit code capture: prefers $LASTEXITCODE, falls back to $?

### Command Building

- `buildPowerShellArgs(cmd)` - Returns `['-NoProfile', '-NonInteractive', '-Command', cmd]`
- `encodePowerShellCommand(psCommand)` - Base64 encodes UTF-16LE string
- Sandbox mode: self-invokes pwsh with full flag set
- Non-sandbox: returns bare command, flags added by getSpawnArgs

### CWD Tracking

- Appends exit-code capture and path tracking to command
- Handles sandbox tmpdir for cwd file placement

## resolveDefaultShell.ts

### Default Shell Resolution

- `resolveDefaultShell()` - Returns `'bash'` or `'powershell'`
  - Checks `settings.defaultShell` first
  - Falls back to `'bash'` (no auto-flip for Windows)
  - Consistent across all platforms

## readOnlyCommandValidation.ts

### Read-Only Command Validation

Comprehensive command validation maps for shell tools:

#### GIT_READ_ONLY_COMMANDS

Safe git subcommands with allowed flags:
- `git diff` - Display and comparison flags
- `git log` - Display, traversal, format, pickaxe flags
- `git show` - Display and diff flags
- `git status` - Output format, untracked, ignore options
- `git blame` - Line range, output format, ignore options
- `git ls-files` - File selection, output format, exclude patterns
- `git config --get` - Config scope and type flags
- `git branch` - List mode only (creation blocked by callback)
- `git tag` - List mode only (creation blocked by callback)
- `git remote` / `git remote show` - Display only
- `git ls-remote` - Branch/tag filtering (server-option excluded for security)
- `git rev-parse` - SHA resolution, path queries, boolean queries
- `git rev-list` - Commit enumeration, traversal, formatting
- `git describe` - Tag-based description
- `git cat-file` - Object inspection (batch-check only)
- `git for-each-ref` - Ref iteration with formatting
- `git grep` - Pattern search in tracked files
- `git stash list/show` - Stash display
- `git worktree list` - Worktree listing
- `git merge-base` - Common ancestor finding
- `git shortlog` / `git reflog` - Log summaries

Each command has:
- `safeFlags` - Allowed flags with argument types (none, number, string, char)
- `additionalCommandIsDangerousCallback` - Dynamic danger checks
- `respectsDoubleDash` - Whether `--` terminates flag parsing

#### GH_READ_ONLY_COMMANDS (Ant-only)

Safe GitHub CLI commands:
- `gh pr view/list/diff/checks/status`
- `gh issue view/list/status`
- `gh repo view`
- `gh run list/view`
- `gh auth status`
- `gh release list/view`
- `gh workflow list/view`
- `gh label list`
- `gh search repos/issues/prs/commits/code`

All use `ghIsDangerousCallback` to prevent network exfiltration:
- Rejects HOST/OWNER/REPO format (3+ segments)
- Rejects URLs (://) and SSH-style (@)

#### Other Read-Only Commands

- `DOCKER_READ_ONLY_COMMANDS` - docker logs, docker inspect
- `EXTERNAL_READONLY_COMMANDS` - Cross-shell safe commands (cat, ls, echo, etc.)

### UNC Path Detection

- `containsVulnerableUncPath(path)` - Detects UNC paths that could leak credentials

## outputLimits.ts

### Output Length Limits

- `BASH_MAX_OUTPUT_UPPER_LIMIT` - 150,000 characters
- `BASH_MAX_OUTPUT_DEFAULT` - 30,000 characters
- `getMaxOutputLength()` - Returns effective limit from `BASH_MAX_OUTPUT_LENGTH` env var

## prefix.ts / specPrefix.ts

### Command Prefix Extraction

- `getCommandPrefixStatic(command)` - Extracts permission rule prefix from command
  - Uses tree-sitter parser for AST-based extraction
  - Handles wrapper commands (nice, sudo, timeout)
  - Recursively processes nested commands
  - Includes environment variable prefix

- `getCompoundCommandPrefixesStatic(command, excludeSubcommand?)` - Prefixes for compound commands
  - Splits on && / || / ;
  - Collapses prefixes sharing same root via word-aligned LCP

- `longestCommonPrefix(strings)` - Word-aligned LCP computation

## shellToolUtils.ts

### Shell Tool Utilities

- `SHELL_TOOL_NAMES` - Array of shell tool names
- `isPowerShellToolEnabled()` - PowerShell availability check

## Dependencies

- `bash/` - Bash parsing and analysis
- `envUtils.js` - Environment variable checks
- `platform.js` - Platform detection
- `sessionEnvVars.js` - Session environment variables
- `tmuxSocket.js` - TMUX socket isolation (ant-only)
- `settings/settings.js` - Default shell resolution
