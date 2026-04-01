# Computer Use

## Purpose

Implements Claude Code's Computer Use feature (codename "Chicago") — an MCP-based system that allows Claude to control the user's macOS desktop via screenshots, mouse, and keyboard input. Covers the full stack: native module loading, CLI executor implementation, session context binding, permission dialogs, file-based locking, Escape hotkey abort, CFRunLoop pumping, app name sanitization, tool rendering, and turn-end cleanup.

## Location

Core infrastructure:
- `restored-src/src/utils/computerUse/common.ts` — Shared constants, terminal bundle detection, capabilities
- `restored-src/src/utils/computerUse/executor.ts` — CLI `ComputerExecutor` implementation (mouse, keyboard, screenshot, app management)
- `restored-src/src/utils/computerUse/wrapper.tsx` — `.call()` override adapter between ToolUseContext and the MCP package
- `restored-src/src/utils/computerUse/hostAdapter.ts` — Process-lifetime singleton host adapter
- `restored-src/src/utils/computerUse/setup.ts` — Dynamic MCP config and allowed tool names setup
- `restored-src/src/utils/computerUse/mcpServer.ts` — In-process MCP server for ListTools

Native module loaders:
- `restored-src/src/utils/computerUse/inputLoader.ts` — Lazy loader for `@ant/computer-use-input` (Rust/enigo — mouse, keyboard)
- `restored-src/src/utils/computerUse/swiftLoader.ts` — Lazy loader for `@ant/computer-use-swift` (SCContentFilter screenshots, NSWorkspace apps, TCC)

Lifecycle and safety:
- `restored-src/src/utils/computerUse/computerUseLock.ts` — File-based lock (O_EXCL) for cross-session mutual exclusion
- `restored-src/src/utils/computerUse/cleanup.ts` — Turn-end cleanup: auto-unhide apps, unregister Escape, release lock
- `restored-src/src/utils/computerUse/drainRunLoop.ts` — Shared CFRunLoop pump for @MainActor Swift methods under libuv
- `restored-src/src/utils/computerUse/escHotkey.ts` — Global Escape → abort via CGEventTap

UI and gating:
- `restored-src/src/utils/computerUse/toolRendering.tsx` — Rendering overrides for `mcp__computer-use__*` tools in terminal
- `restored-src/src/utils/computerUse/appNames.ts` — App name filtering and prompt-injection hardening
- `restored-src/src/utils/computerUse/gates.ts` — Feature gates, subscription checks, coordinate mode

## Key Exports

### Common (`common.ts`)

#### Constants
- `COMPUTER_USE_MCP_SERVER_NAME`: `'computer-use'`
- `CLI_HOST_BUNDLE_ID`: Sentinel bundle ID (`'com.anthropic.claude-code.cli-no-window'`) — never matches a real frontmost app
- `CLI_CU_CAPABILITIES`: `{ screenshotFiltering: 'native', platform: 'darwin' }`

#### Functions
- `getTerminalBundleId()`: Detect the terminal emulator's bundle ID from `__CFBundleIdentifier` env var or fallback table
- `isComputerUseMCPServer(name)`: Check if an MCP server name matches the computer-use server

### Executor (`executor.ts`)

The CLI `ComputerExecutor` implementation wrapping two native modules. Key differences from the Electron (Cowork) reference:
- No `withClickThrough` — terminal has no window
- Terminal as surrogate host — detected via `getTerminalBundleId()`, exempted from hide and screenshot exclusion
- Clipboard via `pbcopy`/`pbpaste` instead of Electron clipboard

#### Factory
- `createCliExecutor(opts)`: Create a `ComputerExecutor` instance. Throws on non-darwin platforms.

#### Executor Methods
- `prepareForAction(allowlistBundleIds, displayId?)`: Hide non-allowed apps, activate target app. Returns hidden app IDs
- `previewHideSet(allowlistBundleIds, displayId?)`: Preview which apps would be hidden
- `getDisplaySize(displayId?)`, `listDisplays()`, `findWindowDisplays(bundleIds)`: Display geometry queries
- `resolvePrepareCapture(opts)`: Resolve display, optionally hide, capture screenshot
- `screenshot(opts)`: Capture screenshot excluding terminal, pre-sized to API target dimensions
- `zoom(regionLogical, allowedBundleIds, displayId?)`: Capture a region of the screen
- `key(keySequence, repeat?)`: xdotool-style key sequence (e.g., "ctrl+shift+a")
- `holdKey(keyNames, durationMs)`: Press and hold keys for a duration
- `type(text, opts)`: Type text, optionally via clipboard (save → write → verify → Cmd+V → restore)
- `readClipboard()`, `writeClipboard(text)`: Clipboard via pbpaste/pbcopy
- `moveMouse(x, y)`, `click(x, y, button, count, modifiers?)`, `mouseDown()`, `mouseUp()`, `getCursorPosition()`: Mouse operations
- `drag(from?, to)`: Drag with animated move (ease-out-cubic at 60fps)
- `scroll(x, y, dx, dy)`: Scroll at position
- `getFrontmostApp()`, `appUnderPoint(x, y)`, `listInstalledApps()`, `getAppIcon(path)`, `listRunningApps()`, `openApp(bundleId)`: App management

#### Module Export
- `unhideComputerUseApps(bundleIds)`: Unhide apps hidden during a turn (called at turn-end)

### Wrapper (`wrapper.tsx`)

Binds the MCP package's `bindSessionContext` to Claude Code's `ToolUseContext`.

#### Functions
- `buildSessionContext()`: Build the `ComputerUseSessionContext` object with read/write callbacks for AppState, lock management, and permission dialogs
- `getOrBind()`: Get or create the cached binding (built once, reused for process lifetime)
- `getComputerUseMCPToolOverrides(toolName)`: Get the full override object (rendering + `.call()`) for an MCP tool
- `runPermissionDialog(req)`: Render approval dialog mid-call via `setToolJSX` + Promise, wait for user response

### Host Adapter (`hostAdapter.ts`)

#### Functions
- `getComputerUseHostAdapter()`: Process-lifetime singleton. Builds once on first CU tool call. Includes logger, executor, OS permission checks, sub-gates, and disabled check.

### Setup (`setup.ts`)

#### Functions
- `setupComputerUseMCP()`: Build dynamic MCP config + allowed tool names. Creates stdio server config that is intercepted by name (not actually spawned).

### MCP Server (`mcpServer.ts`)

#### Functions
- `createComputerUseMcpServerForCli()`: Construct in-process server with installed-app names in `request_access` description
- `runComputerUseMcpServer()`: Subprocess entrypoint for `--computer-use-mcp`. Stdio transport, exit on stdin close.

### Native Module Loaders

- `requireComputerUseInput()`: Lazy-load `@ant/computer-use-input` (Rust/enigo). Caches internally.
- `requireComputerUseSwift()`: Lazy-load `@ant/computer-use-swift`. Caches internally. macOS-only.

### Lock (`computerUseLock.ts`)

#### Types
- `AcquireResult`: `{ kind: 'acquired', fresh: boolean } | { kind: 'blocked', by: string }`
- `CheckResult`: `{ kind: 'free' } | { kind: 'held_by_self' } | { kind: 'blocked', by: }`

#### Functions
- `checkComputerUseLock()`: Check lock state without acquiring. Does stale-PID recovery.
- `tryAcquireComputerUseLock()`: Atomic test-and-set via O_EXCL (open 'wx'). Returns fresh/reentrant/blocked.
- `releaseComputerUseLock()`: Release lock if current session owns it. Returns true if actually unlinked.
- `isLockHeldLocally()`: Zero-syscall check — does this process believe it holds the lock?

### Cleanup (`cleanup.ts`)

#### Functions
- `cleanupComputerUseAfterTurn(ctx)`: Turn-end cleanup: auto-unhide hidden apps (5s timeout), unregister Escape hotkey, release file lock, send exit notification.

### Drain RunLoop (`drainRunLoop.ts`)

#### Functions
- `drainRunLoop(fn)`: Await `fn()` with shared CFRunLoop pump running. 30s timeout. Safe to nest.
- `retainPump`, `releasePump`: Manual pump retain/release for long-lived registrations (Escape handler).

### Escape Hotkey (`escHotkey.ts`)

#### Functions
- `registerEscHotkey(onEscape)`: Register global Escape → abort via CGEventTap. Returns false if CGEventTap creation failed (e.g., missing Accessibility permission).
- `unregisterEscHotkey()`: Unregister the Escape hotkey and release pump retain.
- `notifyExpectedEscape()`: Punch a hole for model-synthesized Escapes so they don't trigger the abort.

### App Names (`appNames.ts`)

#### Functions
- `filterAppsForDescription(installed, homeDir)`: Filter raw Spotlight results to user-facing apps, then sanitize. Always-keep apps (browsers, productivity, dev tools) bypass path/name filter.

#### Filtering
- Path allowlist: `/Applications/`, `/System/Applications/`, `~/Applications/`
- Name blocklist patterns: Helper, Agent, Service, Uninstaller, Updater, dot-prefixed
- Char allowlist: Unicode letters, marks, numbers, space, and limited punctuation
- Max length: 40 chars; Max count: 50 (with "… and N more" overflow)
- Always-keep bundle IDs: ~30 common apps (Safari, Chrome, VSCode, Slack, etc.)

### Tool Rendering (`toolRendering.tsx`)

#### Functions
- `getComputerUseMCPRenderingOverrides(toolName)`: Rendering overrides for `mcp__computer-use__*` tools — custom `userFacingName`, `renderToolUseMessage`, `renderToolResultMessage`.

### Gates (`gates.ts`)

#### Functions
- `getChicagoEnabled()`: Check if Computer Use is enabled (subscription check + GrowthBook config). Ants with monorepo access are excluded unless `ALLOW_ANT_COMPUTER_USE_MCP=1`.
- `getChicagoSubGates()`: Get sub-feature gates (pixelValidation, clipboardPasteMultiline, mouseAnimation, hideBeforeAction, autoTargetDisplay, clipboardGuard).
- `getChicagoCoordinateMode()`: Get coordinate mode ('pixels' or 'normalized'). Frozen at first read.

## Design Notes

- **CFRunLoop pumping**: Swift's `@MainActor` methods and enigo's key() dispatch to `DispatchQueue.main`, which never drains under libuv (Node/bun). The `drainRunLoop` pump runs a 1ms `setInterval` calling `_drainMainRunLoop` while any main-queue-dependent call is pending.
- **Terminal as surrogate host**: The terminal emulator's bundle ID is detected and passed to Swift as the surrogate host — exempting it from hide, skipping it in the activate z-order walk, and excluding it from screenshots.
- **File-based lock**: Uses O_EXCL (open 'wx') for atomic test-and-set. Stale PID recovery unlinks dead sessions' locks. Registered as a shutdown cleanup handler.
- **Escape abort**: CGEventTap consumes Escape system-wide while registered (PI defense — prompt injection can't dismiss dialogs with Escape). Model-synthesized Escapes are excluded via `notifyExpectedEscape()`.
- **Screenshot sizing**: Pre-sizes to `targetImageSize` output so the API transcoder's early-return fires — no server-side resize, coordinate scaling stays coherent.
- **Clipboard typing**: Save → write → read-back verify → Cmd+V → 100ms sleep → restore. The read-back verify prevents pasting junk if clipboard write silently fails.
- **App name hardening**: Unicode-aware char allowlist (`\p{L}\p{M}\p{N}`), single-space only (not `\s` to prevent newline injection), length cap, count cap with overflow indicator.
