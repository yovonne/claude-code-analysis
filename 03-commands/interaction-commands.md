# Interaction Commands

## Purpose

Documents the CLI commands that facilitate user interaction during a session: planning, context management, file handling, directory management, and content copying. These commands help users control the conversation flow and manage what Claude Code has access to.

---

## /plan (Plan Mode)

### Purpose
Enables plan mode for structured task planning, or displays/edits the current session plan when already in plan mode.

### Location
`restored-src/src/commands/plan/index.ts`
`restored-src/src/commands/plan/plan.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `plan` |
| Type | `local-jsx` |
| Description | Enable plan mode or view the current session plan |
| Argument Hint | `[open\|<description>]` |

### Key Exports

#### Functions
- `call(onDone, context, args)`: Entry point; enables plan mode or displays/edits the current plan

#### Components
- `PlanDisplay`: React component that renders the current plan content, file path, and editor hint

### Dependencies

#### Internal
- `../../bootstrap/state.js` — `handlePlanModeTransition` for mode switching
- `../../utils/plans.js` — `getPlan`, `getPlanFilePath` for plan retrieval
- `../../utils/promptEditor.js` — `editFileInEditor` for opening plan in external editor
- `../../utils/editor.js` — `getExternalEditor` for detecting configured editor
- `../../utils/ide.js` — `toIDEDisplayName` for editor display formatting
- `../../utils/permissions/PermissionUpdate.js` — `applyPermissionUpdate` for permission changes
- `../../utils/permissions/permissionSetup.js` — `prepareContextForPlanMode` for plan mode context
- `../../utils/staticRender.js` — `renderToString` for converting React to terminal output
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../commands.js` — `LocalJSXCommandContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone` type

#### External
- `react` — Component rendering with React compiler (`_c` memoization)

### Implementation Details

#### Core Logic — Two Modes

**Mode 1: Not in plan mode — Enable plan mode**
1. Calls `handlePlanModeTransition(currentMode, 'plan')` to handle the mode transition
2. Updates app state with plan mode permissions via `applyPermissionUpdate(prepareContextForPlanMode(...), { type: 'setMode', mode: 'plan', destination: 'session' })`
3. If a description argument is provided (and isn't "open"), triggers a query after enabling
4. Returns "Enabled plan mode"

**Mode 2: Already in plan mode — View or edit plan**
1. Retrieves plan content via `getPlan()` and file path via `getPlanFilePath()`
2. If no plan content exists: Returns "Already in plan mode. No plan written yet."
3. If argument is "open": Opens the plan file in the external editor
4. Otherwise: Renders `PlanDisplay` component showing:
   - "Current Plan" header (bold)
   - Plan file path (dim)
   - Plan content
   - Editor hint (if external editor is detected): `"/plan open" to edit this plan in <editor-name>`

#### Plan Display Component
The `PlanDisplay` component uses React compiler memoization to efficiently render:
- Plan title (static, memoized)
- File path (re-renders only when path changes)
- Plan content (re-renders only when content changes)
- Editor hint (re-renders only when editor name changes)

#### Edge Cases
- No plan written yet: Informative message instead of empty display
- Editor not configured: Editor hint is omitted
- Editor open fails: Error message with details returned via `onDone`

### Data Flow
```
User runs /plan [args]
  → Check current mode
  │
  ├─ Not in plan mode → handlePlanModeTransition() → setAppState(plan mode)
  │                     → [Has description?] → onDone('Enabled plan mode', { shouldQuery: true })
  │                     → [No description?] → onDone('Enabled plan mode')
  │
  └─ In plan mode → getPlan() → getPlanFilePath()
                    │
                    ├─ No plan → onDone('Already in plan mode. No plan written yet.')
                    ├─ arg === 'open' → editFileInEditor(planPath) → onDone result
                    └─ Otherwise → renderToString(PlanDisplay) → onDone(output)
```

---

## /compact (Context Compaction)

### Purpose
Clears conversation history while keeping a summary in context. Reduces token usage by summarizing the conversation. Accepts optional custom summarization instructions.

### Location
`restored-src/src/commands/compact/index.ts`
`restored-src/src/commands/compact/compact.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `compact` |
| Type | `local` |
| Description | Clear conversation history but keep a summary in context |
| Non-interactive | `true` |
| Argument Hint | `<optional custom summarization instructions>` |
| Enabled | Unless `DISABLE_COMPACT` env var is truthy |

### Key Exports

#### Functions
- `call(args, context)`: Main entry point; performs conversation compaction
- `compactViaReactive(messages, context, customInstructions, reactive)`: Reactive-mode compaction path
- `buildDisplayText(context, userDisplayMessage?)`: Builds the dimmed display text shown after compaction
- `getCacheSharingParams(context, messages)`: Builds system prompt and context for cache-aware compaction

### Dependencies

#### Internal
- `../../services/compact/compact.js` — `compactConversation`, `mergeHookInstructions`, error constants
- `../../services/compact/microCompact.js` — `microcompactMessages` for pre-compaction token reduction
- `../../services/compact/reactiveCompact.js` — Reactive compaction (feature-gated)
- `../../services/compact/postCompactCleanup.js` — `runPostCompactCleanup` for post-compaction cleanup
- `../../services/compact/sessionMemoryCompact.js` — `trySessionMemoryCompaction` for SM-based compaction
- `../../services/compact/compactWarningState.js` — `suppressCompactWarning` to hide context warnings
- `../../services/api/promptCacheBreakDetection.js` — `notifyCompaction` for cache break tracking
- `../../services/SessionMemory/sessionMemoryUtils.js` — `setLastSummarizedMessageId`
- `../../utils/hooks.js` — `executePreCompactHooks` for pre-compact hook execution
- `../../utils/messages.js` — `getMessagesAfterCompactBoundary` for message filtering
- `../../utils/systemPrompt.js` — `buildEffectiveSystemPrompt`
- `../../utils/model/contextWindowUpgradeCheck.js` — `getUpgradeMessage` for upgrade tips
- `../../utils/errors.js` — `hasExactErrorMessage` for error classification
- `../../utils/log.js` — `logError` for error logging
- `../../keybindings/shortcutFormat.js` — `getShortcutDisplay` for shortcut hints
- `../../context.js` — `getSystemContext`, `getUserContext`
- `../../constants/prompts.js` — `getSystemPrompt`
- `../../bootstrap/state.js` — `markPostCompaction`
- `../../Tool.js` — `ToolUseContext` type
- `../../types/command.js` — `LocalCommandCall` type
- `../../types/message.js` — `Message` type

#### External
- `bun:bundle` — `feature` flag for `REACTIVE_COMPACT`, `PROMPT_CACHE_BREAK_DETECTION`
- `chalk` — Terminal text formatting

### Implementation Details

#### Compaction Strategy Priority
The command tries three compaction strategies in order:

1. **Session Memory Compaction** (no custom instructions only)
   - Fastest path — uses session memory to compact without a full model call
   - Doesn't support custom instructions
   - On success: clears user context cache, runs post-compact cleanup, notifies cache break detection, suppresses compact warning

2. **Reactive Compaction** (feature-gated via `REACTIVE_COMPACT`)
   - Used when `reactiveCompact.isReactiveOnlyMode()` returns true
   - Runs pre-compact hooks concurrently with cache-safe param building
   - Merges hook instructions with custom instructions
   - Shows progress via `context.onCompactProgress` callbacks
   - Handles failure reasons: `too_few_groups`, `aborted`, `exhausted`, `error`, `media_unstrippable`

3. **Traditional Compaction** (fallback)
   - Runs microcompact first to reduce tokens before summarization
   - Calls `compactConversation` with cache sharing params
   - Resets `lastSummarizedMessageId` (legacy compaction replaces all messages)

#### Pre-Compact Hooks
When using reactive compaction, pre-compact hooks are executed:
- Hooks run concurrently with cache param building (independent operations)
- Hook results can add custom instructions via `mergeHookInstructions`
- Post-compact hooks run inside `reactiveCompactOnPromptTooLong`
- Pre and post hook display messages are combined for the final output

#### Display Text
The `buildDisplayText` function builds a dimmed summary:
- Includes expand shortcut hint (`ctrl+o` to see full summary) when not in verbose mode
- Appends user display message from hooks if present
- Appends upgrade message if available (tip to use a larger context window model)

#### Edge Cases
- No messages: Throws "No messages to compact"
- User abort: Throws "Compaction canceled."
- Too few messages: Throws `ERROR_MESSAGE_NOT_ENOUGH_MESSAGES`
- Incomplete response: Throws `ERROR_MESSAGE_INCOMPLETE_RESPONSE`
- Other errors: Logged and wrapped as "Error during compaction: <error>"

### Data Flow
```
User runs /compact [instructions]
  → getMessagesAfterCompactBoundary(messages)
  → [No custom instructions?] → trySessionMemoryCompaction()
  │                              ├─ Success → cleanup → return
  │                              └─ No result → continue
  │
  → [Reactive mode?] → compactViaReactive()
  │                     → executePreCompactHooks() + getCacheSharingParams() (parallel)
  │                     → mergeHookInstructions()
  │                     → reactiveCompactOnPromptTooLong()
  │                     → cleanup → return
  │
  → [Fallback] → microcompactMessages() → compactConversation()
                 → setLastSummarizedMessageId(undefined)
                 → cleanup → return
```

---

## /context (Context Usage Visualization)

### Purpose
Visualizes current context usage as a colored grid (interactive) or markdown table (non-interactive). Shows token consumption breakdown by category.

### Location
`restored-src/src/commands/context/index.ts`
`restored-src/src/commands/context/context.tsx` (interactive)
`restored-src/src/commands/context/context-noninteractive.ts` (non-interactive)

### Command Definition

**Interactive version:**
| Property | Value |
|----------|-------|
| Name | `context` |
| Type | `local-jsx` |
| Description | Visualize current context usage as a colored grid |
| Enabled | Only when NOT in non-interactive session |

**Non-interactive version:**
| Property | Value |
|----------|-------|
| Name | `context` |
| Type | `local` |
| Description | Show current context usage |
| Non-interactive | `true` |
| Hidden | When NOT in non-interactive session |

### Key Exports

#### Functions
- `call(onDone, context)` (interactive): Analyzes context, renders `ContextVisualization` component
- `call(_args, context)` (non-interactive): Returns context data as markdown table
- `collectContextData(context)`: Shared data collection used by both versions and the SDK
- `formatContextAsMarkdownTable(data)` (non-interactive): Formats context data as markdown tables

### Dependencies

#### Internal
- `../../utils/analyzeContext.js` — `analyzeContextUsage` for token breakdown analysis
- `../../services/compact/microCompact.js` — `microcompactMessages` for accurate token counts
- `../../utils/messages.js` — `getMessagesAfterCompactBoundary` for message filtering
- `../../utils/staticRender.js` — `renderToAnsiString` for interactive rendering
- `../../utils/format.js` — `formatTokens` for human-readable token counts
- `../../utils/stringUtils.js` — `plural` for count formatting
- `../../utils/settings/constants.js` — `getSourceDisplayName` for agent source labels
- `../../components/ContextVisualization.js` — Interactive grid component
- `../../bootstrap/state.js` — `getIsNonInteractiveSession` for mode detection
- `../../commands.js` — `Command` type
- `../../Tool.js` — `ToolUseContext`, `Tools` types
- `../../types/message.js` — `Message` type
- `../../state/AppStateStore.js` — `AppState` type

#### External
- `bun:bundle` — `feature` flag for `CONTEXT_COLLAPSE`

### Implementation Details

#### Shared Data Collection (`collectContextData`)
Both versions use the same data collection pipeline:
1. Gets messages after compact boundary via `getMessagesAfterCompactBoundary`
2. Applies `projectView` transform if `CONTEXT_COLLAPSE` feature is enabled
3. Runs `microcompactMessages` to get accurate token representation
4. Calls `analyzeContextUsage` with compacted messages, model, tools, agents, and full context

#### Interactive Version
1. Gets terminal width from `process.stdout.columns` (default 80)
2. Collects context data via the shared pipeline
3. Renders `ContextVisualization` component with the data
4. Converts to ANSI string via `renderToAnsiString` and passes to `onDone`

#### Non-Interactive Version (Markdown Table)
Formats context data into structured markdown:

**Main sections:**
- Model name and token summary (used / max, percentage)
- Context collapse status (if enabled): collapsed spans, staged spans, spawn health, errors
- Estimated usage by category table
- MCP Tools table (tool name, server, tokens)
- Custom Agents table (agent type, source, tokens)
- Memory Files table (type, path, tokens)
- Skills table (skill name, source, tokens)
- System Tools table (ant-only)
- System Prompt Sections table (ant-only)
- Message Breakdown (ant-only): tool calls, tool results, attachments, assistant/user messages
- Top Tools breakdown (ant-only): call tokens and result tokens per tool
- Top Attachments breakdown (ant-only): tokens per attachment type

#### Agent Source Display
Agent sources are mapped to display names:
| Source | Display |
|--------|---------|
| `projectSettings` | Project |
| `userSettings` | User |
| `localSettings` | Local |
| `flagSettings` | Flag |
| `policySettings` | Policy |
| `plugin` | Plugin |
| `built-in` | Built-in |

#### Context Collapse Integration
When `CONTEXT_COLLAPSE` is enabled:
- Shows current strategy status (collapsed spans, staged spans, spawn count)
- Reports errors if any spawns failed (with last error snippet)
- Reports idle warnings if consecutive empty runs detected

### Data Flow
```
User runs /context
  → [Interactive or non-interactive?]
  │
  ├─ Interactive → toApiView(messages) → microcompactMessages()
  │                → analyzeContextUsage(data, terminalWidth, ...)
  │                → renderToAnsiString(ContextVisualization) → onDone(output)
  │
  └─ Non-interactive → collectContextData() → formatContextAsMarkdownTable(data)
                       → return { type: 'text', value: markdown }
```

---

## /files (Files in Context)

### Purpose
Lists all files currently loaded into the conversation context. Ant-only feature.

### Location
`restored-src/src/commands/files/index.ts`
`restored-src/src/commands/files/files.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `files` |
| Type | `local` |
| Description | List all files currently in context |
| Non-interactive | `true` |
| Enabled | Only when `USER_TYPE === 'ant'` |

### Key Exports

#### Functions
- `call(_args, context)`: Returns list of files in context as relative paths

### Dependencies

#### Internal
- `../../utils/cwd.js` — `getCwd` for current working directory
- `../../utils/fileStateCache.js` — `cacheKeys` for extracting file paths from file state
- `../../Tool.js` — `ToolUseContext` type
- `../../types/command.js` — `LocalCommandResult` type

#### External
- `path` — `relative` for computing relative paths

### Implementation Details

#### Core Logic
1. Extracts file keys from `context.readFileState` via `cacheKeys()`
2. If no files: Returns "No files in context"
3. Otherwise: Maps each file path to a relative path from cwd and joins with newlines
4. Returns formatted list: "Files in context:\n<file1>\n<file2>\n..."

### Data Flow
```
User runs /files
  → cacheKeys(context.readFileState) → file path list
  → [Empty?] → "No files in context"
  → [Has files?] → map to relative paths → join with newlines
  → "Files in context:\n<relative-paths>"
```

---

## /add-dir (Add Working Directory)

### Purpose
Adds a new working directory to the session, expanding the filesystem scope that Claude Code can access.

### Location
`restored-src/src/commands/add-dir/index.ts`
`restored-src/src/commands/add-dir/add-dir.tsx`
`restored-src/src/commands/add-dir/validation.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `add-dir` |
| Type | `local-jsx` |
| Description | Add a new working directory |
| Argument Hint | `<path>` |

### Key Exports

#### Functions
- `call(onDone, context, args?)`: Entry point; validates path and shows add directory UI
- `validateDirectoryForWorkspace(directoryPath, permissionContext)`: Validates a directory path
- `addDirHelpMessage(result)`: Generates human-readable error messages for validation failures

#### Types
- `AddDirectoryResult`: Discriminated union for validation outcomes:
  - `{ resultType: 'success', absolutePath }`
  - `{ resultType: 'emptyPath' }`
  - `{ resultType: 'pathNotFound' | 'notADirectory', directoryPath, absolutePath }`
  - `{ resultType: 'alreadyInWorkingDirectory', directoryPath, workingDir }`

### Dependencies

#### Internal
- `../../bootstrap/state.js` — `getAdditionalDirectoriesForClaudeMd`, `setAdditionalDirectoriesForClaudeMd`
- `../../utils/permissions/PermissionUpdate.js` — `applyPermissionUpdate`, `persistPermissionUpdate`
- `../../utils/permissions/filesystem.js` — `allWorkingDirectories`, `pathInWorkingPath`
- `../../utils/sandbox/sandbox-adapter.js` — `SandboxManager.refreshConfig`
- `../../utils/errors.js` — `getErrnoCode` for error classification
- `../../utils/path.js` — `expandPath` for tilde/path expansion
- `../../components/permissions/rules/AddWorkspaceDirectory.js` — Directory input UI
- `../../components/MessageResponse.js` — Error message display
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../commands.js` — `LocalJSXCommandContext` type
- `../../types/command.js` — `LocalJSXCommandOnDone`, `PermissionUpdateDestination` types

#### External
- `chalk` — Terminal text formatting
- `figures` — Unicode symbols (pointer icon)
- `fs/promises` — `stat` for filesystem checks
- `path` — `dirname`, `resolve` for path manipulation
- `react` — Component rendering with `useEffect`

### Implementation Details

#### Validation Pipeline (`validateDirectoryForWorkspace`)
1. Empty path check → `emptyPath` result
2. Resolve and expand path (strips trailing slash for consistent storage keys)
3. `stat()` check: path must exist and be a directory
   - `ENOENT`, `ENOTDIR`, `EACCES`, `EPERM` → `pathNotFound`
   - Not a directory → `notADirectory` (suggests parent directory)
4. Check if already within an existing working directory → `alreadyInWorkingDirectory`
5. All checks pass → `success` with absolute path

#### Add Directory Flow
1. **No path argument**: Shows `AddWorkspaceDirectory` input form directly
2. **Path provided**:
   - Validates the directory via `validateDirectoryForWorkspace`
   - On validation failure: Shows `AddDirError` with help message, then closes
   - On success: Shows `AddWorkspaceDirectory` with pre-filled path for confirmation
3. **On confirmation** (`handleAddDirectory`):
   - Creates permission update (`type: 'addDirectories'`) with target destination (session or local settings)
   - Applies update to session context via `applyPermissionUpdate`
   - Updates sandbox config for Bash command access
   - Refreshes sandbox manager config
   - Optionally persists to local settings
   - Shows confirmation with hint about `/permissions` for management

#### Remember Option
When `remember = true`, the directory is persisted to local settings via `persistPermissionUpdate`. If persistence fails, the directory is still added to the session with a warning.

#### Edge Cases
- Path not found: Clear error with absolute path
- Not a directory: Suggests the parent directory instead
- Already accessible: Informs user it's within an existing working directory
- Trailing slash: Normalized by `resolve()` so `/foo` and `/foo/` map to the same key

### Data Flow
```
User runs /add-dir [path]
  → [No path?] → Show AddWorkspaceDirectory form → User enters path → Validate
  → [Has path?] → validateDirectoryForWorkspace(path)
                  │
                  ├─ Failure → AddDirError(helpMessage) → onDone → close
                  └─ Success → Show AddWorkspaceDirectory(prefilled) → User confirms
                                → applyPermissionUpdate(addDirectories)
                                → setAdditionalDirectoriesForClaudeMd([...dirs, path])
                                → SandboxManager.refreshConfig()
                                → [Remember?] → persistPermissionUpdate()
                                → onDone("Added <path> as a working directory")
```

---

## /copy (Copy Last Response)

### Purpose
Copies Claude's last response to the clipboard. Supports selecting specific code blocks or the full response. Can target Nth-latest response with `/copy N`.

### Location
`restored-src/src/commands/copy/index.ts`
`restored-src/src/commands/copy/copy.tsx`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `copy` |
| Type | `local-jsx` |
| Description | Copy Claude's last response to clipboard (or /copy N for the Nth-latest) |

### Key Exports

#### Functions
- `call(onDone, context, args)`: Entry point; collects recent assistant texts and shows copy picker
- `collectRecentAssistantTexts(messages)`: Walks messages newest-first, extracting assistant message text
- `extractCodeBlocks(markdown)`: Parses markdown to extract fenced code blocks
- `fileExtension(lang)`: Maps language identifier to file extension (sanitized)
- `copyOrWriteToFile(text, filename)`: Copies to clipboard and writes to temp file
- `writeToFile(text, filename)`: Writes text to temp file in `COPY_DIR`

#### Components
- `CopyPicker`: Interactive dialog for selecting what to copy (full response or individual code blocks)

### Dependencies

#### Internal
- `../../components/CustomSelect/select.js` — `Select` component for option picking
- `../../components/design-system/Byline.js` — `Byline` for keyboard shortcut hints
- `../../components/design-system/KeyboardShortcutHint.js` — Shortcut hint display
- `../../components/design-system/Pane.js` — `Pane` container
- `../../ink/termio/osc.js` — `setClipboard` for clipboard operations
- `../../ink/stringWidth.js` — `stringWidth` for accurate string measurement
- `../../services/analytics/index.js` — `logEvent` for usage tracking
- `../../utils/messages.js` — `extractTextContent`, `stripPromptXMLTags`
- `../../utils/config.js` — `getGlobalConfig`, `saveGlobalConfig` for copyFullResponse preference
- `../../utils/stringUtils.js` — `countCharInString` for line counting
- `../../ink.js` — `Box`, `Text` for UI rendering
- `../../types/command.js` — `LocalJSXCommandCall`, `CommandResultDisplay` types
- `../../types/message.js` — `AssistantMessage`, `Message` types

#### External
- `fs/promises` — `mkdir`, `writeFile` for temp file writing
- `os` — `tmpdir` for temp directory
- `path` — `join` for path construction
- `marked` — Markdown parsing for code block extraction
- `react` — Component rendering with React compiler

### Implementation Details

#### Assistant Text Collection
`collectRecentAssistantTexts` walks messages newest-first:
- Skips non-assistant messages and API error messages
- Extracts text content from array-typed message content
- Caps at `MAX_LOOKBACK` (20) messages
- Returns array where index 0 = latest, 1 = second-to-latest, etc.

#### Code Block Extraction
Uses `marked` lexer to parse markdown and extract fenced code blocks:
- Strips prompt XML tags before parsing
- Returns array of `{ code, lang }` objects

#### Copy Picker UI
When there are code blocks and `copyFullResponse` is not set:
1. Shows a `Select` dialog with options:
   - "Full response" (chars, lines count)
   - Each code block (truncated label, language, line count)
   - "Always copy full response" (preference setting)
2. Keyboard shortcuts:
   - `Enter` — Copy selected content
   - `w` — Write selected content to file (skip clipboard)
   - `Esc` — Cancel
3. On "Always copy full response": Saves preference to global config

#### Clipboard + File Fallback
`copyOrWriteToFile`:
1. Attempts clipboard via `setClipboard` (OSC 52 — best-effort)
2. Always writes to temp file as reliable fallback
3. Returns message with char count, line count, and file path

#### File Extension Mapping
`fileExtension(lang)`:
- Sanitizes language identifier (alphanumeric only, prevents path traversal)
- Returns `.<lang>` for valid languages (except `plaintext`)
- Returns `.txt` as default

#### Edge Cases
- No assistant messages: "No assistant message to copy"
- Invalid N argument: Usage hint with valid range
- N exceeds available messages: Shows count of available messages
- Clipboard fails: File write still succeeds as fallback
- File write fails: Returns clipboard-only message

### Data Flow
```
User runs /copy [N]
  → collectRecentAssistantTexts(messages) → text array
  → [Empty?] → "No assistant message to copy"
  → [Has N?] → Validate N → age = N - 1 → text = texts[age]
  → [No N?] → text = texts[0] (latest)
  → extractCodeBlocks(text) → code blocks
  → [No blocks OR copyFullResponse set?] → copyOrWriteToFile(text, 'response.md')
  → [Has blocks AND no preference?] → Show CopyPicker
                                      → User selects option
                                      → copyOrWriteToFile(selected content, filename)
```
