# Session Lifecycle Commands

## Purpose

Documents the CLI commands that manage the full session lifecycle: viewing session info, resuming past conversations, renaming, exporting, sharing, backfilling, and cross-device teleportation.

---

## /session (Remote Session Info)

### Purpose
Displays the remote session URL and generates a QR code for browser access when running in remote mode.

### Location
`restored-src/src/commands/session/index.ts`
`restored-src/src/commands/session/session.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `session` |
| Aliases | `remote` |
| Type | `local-jsx` |
| Description | Show remote session URL and QR code |
| Enabled | Only when `getIsRemoteMode()` returns true |
| Hidden | When not in remote mode |

### Key Exports

#### Functions
- `call`: `LocalJSXCommandCall` — Renders the `SessionInfo` React component

### Dependencies

#### Internal
- `../../bootstrap/state.js` — `getIsRemoteMode` for mode detection
- `../../state/AppState.js` — `useAppState` for reading `remoteSessionUrl`
- `../../keybindings/useKeybinding.js` — Escape key binding to dismiss
- `../../components/design-system/Pane.js` — UI container

#### External
- `qrcode` — QR code generation via `toString`
- `react` — Component rendering with `useEffect`, `useState`

### Implementation Details

#### Core Logic
The command renders a `SessionInfo` React component that:
1. Reads `remoteSessionUrl` from app state via `useAppState`
2. Generates an ASCII QR code from the URL using the `qrcode` library (UTF8 format, low error correction)
3. Displays the QR code line-by-line with the raw URL beneath it
4. Shows a warning message if not in remote mode

#### UI Structure
```
Pane (context: Confirmation)
├── "Remote session" (bold header)
├── QR code lines (or "Generating QR code…" while loading)
├── "Open in browser: <url>" (URL in ide color)
└── "(press esc to close)" (dim)
```

#### Edge Cases
- If `remoteSessionUrl` is null/undefined, displays a warning: "Not in remote mode. Start with `claude --remote` to use this command."
- QR code generation failures are caught and logged via `logForDebugging` without crashing
- The component is hidden entirely when not in remote mode

### Data Flow
```
User runs /session
  → getIsRemoteMode() check
  → SessionInfo component mounts
  → useAppState(s => s.remoteSessionUrl) reads URL
  → useEffect triggers qrToString(url) → setQrCode()
  → QR code lines rendered as Text components
  → User presses Escape → onDone() closes
```

### Integration Points
- **Remote mode bootstrap**: The `getIsRemoteMode()` function determines whether the session is connected to a remote backend
- **App state**: `remoteSessionUrl` is populated during remote session initialization
- **Keybinding system**: Binds to `confirm:no` (Escape) for dismissal

---

## /resume (Session Resumption)

### Purpose
Resumes a previous conversation by selecting from session logs or by providing a session ID, search term, or custom title.

### Location
`restored-src/src/commands/resume/index.ts`
`restored-src/src/commands/resume/resume.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `resume` |
| Aliases | `continue` |
| Type | `local-jsx` |
| Description | Resume a previous conversation |
| Argument Hint | `[conversation id or search term]` |

### Key Exports

#### Functions
- `call`: `LocalJSXCommandCall` — Entry point; shows picker or resolves by argument
- `filterResumableSessions(logs, currentSessionId)`: Filters out sidechain sessions and the current session from the log list

#### Types
- `ResumeResult`: Discriminated union — `{ resultType: 'sessionNotFound', arg }` | `{ resultType: 'multipleMatches', arg, count }`

### Dependencies

#### Internal
- `../../bootstrap/state.js` — `getOriginalCwd`, `getSessionId`
- `../../utils/sessionStorage.js` — `getLastSessionLog`, `getSessionIdFromLog`, `isCustomTitleEnabled`, `isLiteLog`, `loadAllProjectsMessageLogs`, `loadFullLog`, `loadSameRepoMessageLogs`, `searchSessionsByCustomTitle`
- `../../utils/agenticSessionSearch.js` — `agenticSessionSearch` for AI-powered session search
- `../../utils/crossProjectResume.js` — `checkCrossProjectResume` for cross-directory detection
- `../../utils/getWorktreePaths.js` — `getWorktreePaths` for git worktree support
- `../../utils/uuid.js` — `validateUuid` for UUID validation
- `../../components/LogSelector.js` — Interactive log selection UI
- `../../ink/termio/osc.js` — `setClipboard` for copying cross-project commands

#### External
- `chalk` — Terminal text formatting
- `figures` — Unicode symbols (pointer icon)
- `react` — Component rendering

### Implementation Details

#### Core Logic — Two Modes

**Mode 1: No argument — Interactive picker**
1. Loads session logs for the current repo (including worktree paths)
2. Filters out sidechain sessions and the current session via `filterResumableSessions`
3. Renders `LogSelector` component with:
   - Toggle for "all projects" vs "same repo" filtering
   - Agentic session search capability
   - Max height based on terminal size and modal context
4. On selection:
   - Validates session UUID from log
   - Loads full log if it's a lite log
   - Checks for cross-project resume:
     - **Same repo worktree**: Resumes directly
     - **Different project**: Copies the resume command to clipboard and displays instructions
   - Calls `context.resume(sessionId, log, entrypoint)` to perform the resume

**Mode 2: Argument provided — Direct resolution**
Resolution priority order:
1. **Valid UUID**: Direct session ID match, with fallback to `getLastSessionLog` for sessions filtered out by enrichment
2. **Exact custom title match**: When custom titles are enabled, searches by title
3. **Error**: Shows "session not found" or "multiple matches" message

#### Cross-Project Resume Handling
When selecting a session from a different directory:
- The command to resume (e.g., `claude --continue <id> --cwd <path>`) is copied to clipboard
- User sees formatted output with the command and "(Command copied to clipboard)" notice
- Same-repo worktrees are treated as resumable directly

#### Edge Cases
- No resumable sessions: Shows "No conversations found to resume" and closes
- Failed log loading: Shows "Failed to load conversations"
- Invalid UUID: Falls through to title search, then shows error
- Multiple title matches: Shows count and prompts user to be more specific
- Lite logs: Automatically upgraded to full logs before resuming

### Data Flow
```
User runs /resume [arg]
  │
  ├─ No arg → Load logs → Filter resumable → Show LogSelector
  │           → User selects → Validate UUID → Load full log
  │           → Check cross-project → Resume or show command
  │
  └─ Has arg → Validate UUID?
                ├─ Yes → Find matching log → Resume
                ├─ No → Custom title enabled?
                │        ├─ Yes → Search by title
                │        │         ├─ 1 match → Resume
                │        │         └─ >1 match → Error
                │        └─ No → Error: session not found
```

### Integration Points
- **Session storage**: Reads from the session log filesystem for history
- **Resume entrypoint**: Delegates to `context.resume()` which handles the actual session state restoration
- **Clipboard**: Uses OSC terminal sequences for cross-project command copying
- **Worktree support**: Integrates with git worktree path detection

---

## /rename (Session Renaming)

### Purpose
Renames the current conversation session, either with a user-provided name or an AI-generated name based on conversation content.

### Location
`restored-src/src/commands/rename/index.ts`
`restored-src/src/commands/rename/rename.ts`
`restored-src/src/commands/rename/generateSessionName.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `rename` |
| Type | `local-jsx` |
| Description | Rename the current conversation |
| Immediate | `true` |
| Argument Hint | `[name]` |

### Key Exports

#### Functions
- `call(onDone, context, args)`: Main entry point for the rename command
- `generateSessionName(messages, signal)`: Uses Haiku model to generate a kebab-case session name from conversation content

### Dependencies

#### Internal
- `../../bootstrap/state.js` — `getSessionId`
- `../../bridge/bridgeConfig.js` — `getBridgeBaseUrlOverride`, `getBridgeTokenOverride`
- `../../utils/messages.js` — `getMessagesAfterCompactBoundary`
- `../../utils/sessionStorage.js` — `getTranscriptPath`, `saveCustomTitle`, `saveAgentName`
- `../../utils/teammate.js` — `isTeammate` for swarm session check
- `../../services/api/claude.js` — `queryHaiku` for AI name generation
- `../../utils/sessionTitle.js` — `extractConversationText`
- `../../bridge/createSession.js` — `updateBridgeSessionTitle` (dynamic import)

#### External
- `crypto` — `UUID` type

### Implementation Details

#### Core Logic
1. **Teammate guard**: If the session is a swarm teammate, renaming is blocked with message: "This session is a swarm teammate. Teammate names are set by the team leader."
2. **Name resolution**:
   - **No argument**: Calls `generateSessionName()` which queries the Haiku model with conversation text to produce a kebab-case name (2-4 words, e.g., "fix-login-bug")
   - **With argument**: Uses the trimmed argument as the new name
3. **Persistence** (three stores):
   - `saveCustomTitle(sessionId, newName, fullPath)` — Saves the custom title to session storage
   - `updateBridgeSessionTitle(bridgeSessionId, newName, ...)` — Syncs title to claude.ai/code bridge (best-effort, non-blocking)
   - `saveAgentName(sessionId, newName, fullPath)` — Persists as agent name for prompt-bar display
4. **App state update**: Updates `standaloneAgentContext.name` in app state
5. **Confirmation**: Shows "Session renamed to: <name>"

#### AI Name Generation
The `generateSessionName` function:
- Extracts conversation text from messages after the compact boundary
- Sends to Haiku model with system prompt requesting kebab-case JSON output
- Uses structured output with JSON schema: `{ name: string }`
- Returns `null` on failure (timeout, rate limit, network error)
- Errors are logged via `logForDebugging` (not `logError`) to avoid flooding — this runs automatically on every 3rd bridge message

#### Edge Cases
- No conversation context yet: Returns "Could not generate a name: no conversation context yet"
- Haiku model failure: Falls back to requiring manual name input
- Bridge sync failure: Silently caught (best-effort)
- Swarm teammate: Rename is blocked entirely

### Data Flow
```
User runs /rename [name]
  → isTeammate() check → Block if true
  → No name? → generateSessionName(messages) → Haiku API → Parse JSON → Extract name
  → Has name? → Use trimmed argument
  → saveCustomTitle(sessionId, name, path)
  → updateBridgeSessionTitle (async, non-blocking)
  → saveAgentName(sessionId, name, path)
  → Update appState.standaloneAgentContext.name
  → Display confirmation
```

### Integration Points
- **Session storage**: Custom title and agent name are persisted to disk
- **Bridge sync**: Title is synced to claude.ai/code when a bridge session exists
- **Haiku API**: Lightweight model used for name generation (separate from main query model)
- **Swarm system**: Teammate sessions cannot be renamed independently

---

## /export (Session Export)

### Purpose
Exports the current conversation to a text file or clipboard, with options for direct file write or interactive dialog.

### Location
`restored-src/src/commands/export/index.ts`
`restored-src/src/commands/export/export.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `export` |
| Type | `local-jsx` |
| Description | Export the current conversation to a file or clipboard |
| Argument Hint | `[filename]` |

### Key Exports

#### Functions
- `call(onDone, context, args)`: Main entry point
- `extractFirstPrompt(messages)`: Extracts and truncates the first user message for filename generation
- `sanitizeFilename(text)`: Converts text to a safe filename (lowercase, hyphens, no special chars)
- `formatTimestamp(date)`: Formats date as `YYYY-MM-DD-HHMMSS`

### Dependencies

#### Internal
- `../../components/ExportDialog.js` — Interactive export dialog UI
- `../../utils/exportRenderer.js` — `renderMessagesToPlainText` for conversation rendering
- `../../utils/cwd.js` — `getCwd` for file path resolution
- `../../utils/slowOperations.js` — `writeFileSync_DEPRECATED` for file writing

#### External
- `path` — `join` for path construction
- `react` — Component rendering

### Implementation Details

#### Core Logic — Two Modes

**Mode 1: Filename argument provided — Direct export**
1. Trims the argument; if it doesn't end with `.txt`, appends it
2. Resolves full path via `join(getCwd(), filename)`
3. Renders conversation to plain text via `renderMessagesToPlainText`
4. Writes file synchronously with UTF-8 encoding and flush
5. Shows confirmation: "Conversation exported to: <filepath>"

**Mode 2: No argument — Interactive dialog**
1. Renders conversation to plain text
2. Generates a default filename:
   - Extracts first user prompt → sanitize → `<timestamp>-<sanitized>.txt`
   - If no prompt: `conversation-<timestamp>.txt`
3. Renders `ExportDialog` component with content and default filename
4. Dialog handles user choice (file write or clipboard copy)

#### Filename Sanitization
The `sanitizeFilename` function applies these transformations in order:
1. Convert to lowercase
2. Remove special characters (keep `a-z`, `0-9`, whitespace, hyphens)
3. Replace spaces with hyphens
4. Collapse multiple hyphens into one
5. Strip leading/trailing hyphens

#### First Prompt Extraction
`extractFirstPrompt` finds the first user message and:
- Handles both string and array content types
- Takes only the first line
- Truncates to 49 characters with ellipsis if longer than 50

### Data Flow
```
User runs /export [filename]
  → renderMessagesToPlainText(messages, tools)
  │
  ├─ Has filename → Append .txt if needed → Write file → Confirm
  │
  └─ No filename → Extract first prompt → Sanitize → Generate default filename
                   → Show ExportDialog → User chooses export method
```

### Integration Points
- **Export renderer**: Converts the full message tree (including tool calls, responses) to plain text
- **ExportDialog**: Provides the interactive UI for choosing export destination
- **File system**: Writes to current working directory

---

## /share (Session Sharing)

### Purpose
Placeholder for session sharing functionality.

### Location
`restored-src/src/commands/share/index.js`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `stub` |
| Enabled | `false` |
| Hidden | `true` |

### Implementation Details
This command is currently a disabled stub:
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
The share functionality is not implemented in this version of the codebase.

---

## /backfill-sessions (Session Backfill)

### Purpose
Placeholder for session backfill functionality.

### Location
`restored-src/src/commands/backfill-sessions/index.js`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `stub` |
| Enabled | `false` |
| Hidden | `true` |

### Implementation Details
This command is currently a disabled stub:
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
The backfill-sessions functionality is not implemented in this version of the codebase.

---

## /teleport (Cross-Device Session Migration)

### Purpose
Placeholder for cross-device session migration functionality.

### Location
`restored-src/src/commands/teleport/index.js`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `stub` |
| Enabled | `false` |
| Hidden | `true` |

### Implementation Details
This command is currently a disabled stub:
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```
The teleport functionality is not implemented in this version of the codebase.

---

## Session Lifecycle Overview

The session lifecycle commands form a cohesive system for managing conversations:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  /session   │     │   /resume   │     │   /rename   │
│  (view URL) │     │ (continue)  │     │  (set name) │
└─────────────┘     └─────────────┘     └─────────────┘
       │                    │                    │
       │              ┌─────┴─────┐              │
       │              │ Session   │              │
       │              │ Storage   │◄─────────────┘
       │              └─────┬─────┘
       │                    │
┌─────────────┐     ┌───────┴───────┐
│   /export   │     │  Reserved     │
│  (save to   │     │  /share       │
│   file)     │     │  /teleport    │
└─────────────┘     │  /backfill    │
                    └───────────────┘
```

### Lifecycle Stages
1. **Active session**: `/session` shows remote URL, `/rename` sets identity, `/export` saves content
2. **Session end**: Session is persisted to log storage automatically
3. **Resumption**: `/resume` restores from logs with cross-project and worktree awareness
4. **Future**: `/share`, `/teleport`, `/backfill-sessions` are reserved for upcoming features

### Related Modules
- [clear-exit-commands.md](./clear-exit-commands.md) — Session termination and cleanup
- [session-lifecycle.md](../09-data-flow/session-lifecycle.md) — Session lifecycle data flow
- [sessionStorage](../../restored-src/src/utils/sessionStorage.js) — Session storage utilities
