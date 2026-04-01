# Deep Link

## Purpose

Implements the `claude-cli://` custom URI scheme for launching Claude Code sessions from external sources (browsers, other apps). Covers URI parsing with security validation, OS-level protocol handler registration (macOS/Linux/Windows), terminal emulator detection and launching, and a security banner for deep-link-originated sessions.

## Location

- `restored-src/src/utils/deepLink/parseDeepLink.ts` — URI parsing and validation
- `restored-src/src/utils/deepLink/registerProtocol.ts` — OS-level protocol handler registration
- `restored-src/src/utils/deepLink/protocolHandler.ts` — Entry point for `claude --handle-uri`
- `restored-src/src/utils/deepLink/terminalLauncher.ts` — Terminal emulator detection and launching
- `restored-src/src/utils/deepLink/terminalPreference.ts` — Terminal preference capture for headless context
- `restored-src/src/utils/deepLink/banner.ts` — Security warning banner for deep-link sessions

## Key Exports

### URI Parser (`parseDeepLink.ts`)

#### Types
- `DeepLinkAction`: `{ query?, cwd?, repo? }` — parsed action from URI

#### Constants
- `DEEP_LINK_PROTOCOL`: `'claude-cli'`

#### Functions
- `parseDeepLink(uri)`: Parse a `claude-cli://open` URI into a structured action. Validates scheme, hostname, and all parameters.
- `buildDeepLink(action)`: Build a `claude-cli://` URL from an action object.

#### Validation Rules
- `cwd`: Must be absolute path, max 4096 chars, no ASCII control characters
- `repo`: Must match `owner/repo` pattern (alphanumerics, dots, hyphens, underscores, exactly one slash)
- `q` (query): Unicode-sanitized, no control characters, max 5000 chars
- Control characters (0x00-0x1F, 0x7F) rejected in all fields

### Protocol Handler Registration (`registerProtocol.ts`)

#### Constants
- `MACOS_BUNDLE_ID`: `'com.anthropic.claude-code-url-handler'`

#### Functions
- `registerProtocolHandler(claudePath?)`: Register the `claude-cli://` scheme with the OS
- `isProtocolHandlerCurrent(claudePath)`: Check if the OS registration points at the expected binary
- `ensureDeepLinkProtocolRegistered()`: Auto-register when missing or stale (runs every session, fire-and-forget)

#### Platform Details
- **macOS**: Creates a minimal `.app` bundle in `~/Applications` with `CFBundleURLTypes` in Info.plist; symlinks to the signed `claude` binary; re-registers with LaunchServices
- **Linux**: Creates a `.desktop` file in `$XDG_DATA_HOME/applications`; registers with `xdg-mime`
- **Windows**: Writes registry keys under `HKEY_CURRENT_USER\Software\Classes\claude-cli`

### Protocol Handler (`protocolHandler.ts`)

#### Functions
- `handleDeepLinkUri(uri)`: Parse URI, resolve cwd (explicit > repo MRU clone > home), read FETCH_HEAD age, launch in terminal. Returns exit code.
- `handleUrlSchemeLaunch()`: Handle macOS URL scheme launch via NAPI module. Returns exit code or null if not a URL launch.
- `resolveCwd(action)`: Resolve working directory: explicit cwd > repo lookup (MRU clone) > home directory

### Terminal Launcher (`terminalLauncher.ts`)

#### Types
- `TerminalInfo`: `{ name, command }`

#### Functions
- `detectTerminal()`: Detect user's preferred terminal emulator (platform-specific)
- `launchInTerminal(claudePath, action)`: Launch Claude Code in the detected terminal with appropriate arguments

#### Platform Support
- **macOS**: iTerm2, Ghostty, Kitty, Alacritty, WezTerm, Terminal.app (in preference order)
- **Linux**: ghostty, kitty, alacritty, wezterm, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm
- **Windows**: Windows Terminal (wt.exe), PowerShell, cmd.exe

#### Security: Pure argv vs Shell String
- Pure argv paths (no shell): Ghostty, Alacritty, Kitty, WezTerm, all Linux terminals, Windows Terminal — user input travels as distinct argv elements
- Shell string paths (shell-quoted): iTerm2, Terminal.app (AppleScript), PowerShell, cmd.exe — user input is shell-escaped via platform-specific quoting functions

### Terminal Preference (`terminalPreference.ts`)

#### Functions
- `updateDeepLinkTerminalPreference()`: Capture current terminal from `TERM_PROGRAM` env var and store in global config for the headless deep link handler to use later (macOS only)

### Deep Link Banner (`banner.ts`)

#### Types
- `DeepLinkBannerInfo`: `{ cwd, prefillLength?, repo?, lastFetch? }`

#### Functions
- `buildDeepLinkBanner(info)`: Build multi-line warning banner for deep-link-originated sessions. Shows working directory, repo slug, fetch age, and prefill review prompt.
- `readLastFetchTime(cwd)`: Read mtime of `.git/FETCH_HEAD` to determine repo freshness. Checks both worktree and common dir for worktree setups.

#### Security Design
- Banner requires user to press Enter before submitting, priming them to review the pre-filled prompt
- Switches from "review carefully" to "scroll to review the entire prompt" when prefill exceeds 1000 chars (doesn't fit on one screen)
- Shows repo fetch age so users can detect stale CLAUDE.md files

## Design Notes

- Protocol registration is idempotent — artifact check (symlink target, .desktop Exec line, registry value) makes it a no-op after first successful run
- Failure backoff: EACCES/ENOSPC errors are throttled to once per 24h via a marker file
- The deep link handler runs headless (no TTY) because the OS launches the binary directly
- Terminal detection on macOS checks stored preference first, then `TERM_PROGRAM`, then Spotlight/`/Applications` lookup
- Pure argv paths are preferred over shell strings for security — no shell interpretation of user input
