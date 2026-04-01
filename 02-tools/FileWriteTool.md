# FileWriteTool

## Purpose

The FileWriteTool creates new files or overwrites existing files in the local filesystem. It is the primary tool for full-file content replacement, used by the model when creating new files from scratch or performing complete rewrites. Unlike the FileEditTool (which sends only diffs), the FileWriteTool replaces the entire file content in a single operation.

## Location

- `restored-src/src/tools/FileWriteTool/FileWriteTool.ts` — Main tool implementation (~435 lines)
- `restored-src/src/tools/FileWriteTool/prompt.ts` — Tool name and description strings (~19 lines)
- `restored-src/src/tools/FileWriteTool/UI.tsx` — React UI rendering components (~405 lines)

## Key Exports

### Functions

- `FileWriteTool`: The complete tool definition built via `buildTool()`
- `countLines(content)`: Counts visible lines in file content, treating trailing newlines as terminators
- `userFacingName(input)`: Returns `'Write'` or `'Updated plan'` for plan files
- `getWriteToolDescription()`: Returns the tool prompt shown to the model
- `getToolUseSummary(input)`: Returns the display path for activity summaries
- `isResultTruncated(output)`: Determines if created file content is truncated in the UI

### Types

- `Output`: `{ type: 'create' | 'update', filePath, content, structuredPatch, originalFile, gitDiff? }`
- `FileWriteToolInput`: Zod input schema type with `file_path` and `content`

### Constants

- `FILE_WRITE_TOOL_NAME`: `'Write'`
- `DESCRIPTION`: `'Write a file to the local filesystem.'`
- `MAX_LINES_TO_RENDER`: `10` — Maximum lines shown in condensed UI view
- `FILE_UNEXPECTEDLY_MODIFIED_ERROR`: Error message for concurrent modification detection

## Dependencies

### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `utils/file.ts` | `writeTextContent()`, `getFileModificationTime()`, `getDisplayPath()` |
| `utils/fileRead.ts` | `readFileSyncWithMetadata()` — synchronous file read with encoding detection |
| `utils/fsOperations.ts` | `getFsImplementation()` — abstracted filesystem operations |
| `utils/permissions/filesystem.ts` | `checkWritePermissionForTool()`, `matchingRuleForInput()` |
| `utils/permissions/shellRuleMatching.ts` | `matchWildcardPattern()` — glob-style permission matching |
| `utils/path.ts` | `expandPath()` — resolves `~` and relative paths to absolute |
| `utils/diff.ts` | `getPatchForDisplay()`, `countLinesChanged()` |
| `utils/gitDiff.ts` | `fetchSingleFileGitDiff()` — generates git diff for changes |
| `utils/fileHistory.ts` | `fileHistoryEnabled()`, `fileHistoryTrackEdit()` — backup tracking |
| `utils/fileOperationAnalytics.ts` | `logFileOperation()` — analytics logging |
| `services/lsp/manager.js` | LSP server notification for file changes |
| `services/lsp/LSPDiagnosticRegistry.js` | Diagnostic clearing before writes |
| `services/mcp/vscodeSdkMcp.js` | VSCode file change notification |
| `services/teamMemorySync/teamMemSecretGuard.js` | Secret detection in file content |
| `skills/loadSkillsDir.js` | Dynamic skill discovery from file paths |
| `services/analytics/growthbook.js` | Feature flag evaluation |
| `tools/FileEditTool/constants.js` | Shared error constant for modified files |
| `tools/FileEditTool/types.js` | Shared Zod schemas for patches and diffs |

### External Dependencies

| Package | Purpose |
|---------|---------|
| `zod/v4` | Input/output schema validation |
| `path` (Node.js) | `dirname()`, `sep()`, path manipulation |
| `diff` | Structured patch generation |

## Implementation Details

### File Creation

The tool distinguishes between two operation types:

1. **Create** — Writing to a file that does not exist (`isENOENT` on stat)
2. **Update** — Overwriting an existing file

The distinction is made during `validateInput()` by attempting `fs.stat(fullFilePath)`:

```
fs.stat(fullFilePath)
  ├── ENOENT → new file creation (validateInput returns { result: true })
  ├── stat succeeds → existing file (proceed with staleness checks)
  └── other error → rethrow
```

For new files, the output sets `type: 'create'`, `originalFile: null`, and `structuredPatch: []`. For updates, it includes the original content and a structured diff.

### Writing Implementation

The write path follows a strict sequence to ensure data integrity:

```
1. expandPath(file_path) → fullFilePath
2. dirname(fullFilePath) → dir
3. discoverSkillDirsForPaths([fullFilePath], cwd) → newSkillDirs (fire-and-forget)
4. activateConditionalSkillsForPaths([fullFilePath], cwd)
5. diagnosticTracker.beforeFileEdited(fullFilePath)
6. fs.mkdir(dir) → ensure parent directory exists
7. fileHistoryTrackEdit() → backup pre-edit content (if enabled)
8. readFileSyncWithMetadata(fullFilePath) → meta (current file state)
9. Staleness check: compare mtime against readFileState timestamp
10. writeTextContent(fullFilePath, content, enc, 'LF') → atomic write
11. LSP notifications: clearDeliveredDiagnostics + changeFile + saveFile
12. notifyVscodeFileUpdated(fullFilePath, oldContent, content)
13. Update readFileState with new content and timestamp
14. Log CLAUDE.md writes (analytics event)
15. Compute git diff (if remote + feature flag enabled)
16. Return result with type, content, structuredPatch, originalFile, gitDiff
```

**Critical section**: Steps 8–10 form the atomic read-modify-write section. No async operations are permitted between the staleness check (step 9) and the actual write (step 10) to prevent race conditions with concurrent edits.

**Line ending policy**: The tool writes content exactly as provided by the model with explicit `'LF'` line ending enforcement. It does NOT preserve the old file's line endings or sample the repo for line ending conventions. This prevents silent corruption of files (e.g., bash scripts with `\r` on Linux when overwriting a CRLF file).

### Directory Creation

Before writing, the tool ensures the parent directory exists:

```typescript
await getFsImplementation().mkdir(dir)
```

This call is placed **outside** the critical section (between staleness check and write) for two reasons:

1. A `yield` between the check and write would allow concurrent edits to interleave
2. Lazy mkdir-on-ENOENT inside `writeTextContent` would fire a spurious `tengu_atomic_write_error` before the actual ENOENT propagates

The `mkdir` call is idempotent — it succeeds silently if the directory already exists.

### Permission Handling

The tool implements a multi-layered permission system:

#### 1. Write Permission Check (`checkPermissions`)

Delegates to `checkWritePermissionForTool()` which evaluates:
- Always-allow rules
- Always-deny rules  
- Permission mode (bypassPermissions, acceptEdits, auto, etc.)
- Hooks (PreToolUse/PostToolUse)
- Auto-mode classifier

#### 2. Permission Matcher (`preparePermissionMatcher`)

Returns a function that matches wildcard patterns against the file path:

```typescript
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}
```

This enables fine-grained permission rules like `Write(src/**/*.ts)` to allow writes only to TypeScript files in `src/`.

#### 3. Path Expansion (`backfillObservableInput`)

Expands `~` and relative paths to absolute paths **before** permission evaluation. This prevents bypassing permission allowlists via path tricks:

```typescript
backfillObservableInput(input) {
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)
  }
}
```

#### 4. Deny Rule Validation (`validateInput`)

Checks if the target path matches any deny rules:

```typescript
const denyRule = matchingRuleForInput(
  fullFilePath,
  appState.toolPermissionContext,
  'edit',
  'deny',
)
if (denyRule !== null) {
  return { result: false, message: 'File is in a directory that is denied...', errorCode: 1 }
}
```

### Safety Checks

The tool implements multiple safety mechanisms to prevent data loss and corruption:

#### 1. Read-Before-Write Requirement

Files must be read before they can be written. The `validateInput()` function checks `readFileState`:

```typescript
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
    errorCode: 2,
  }
}
```

Partial views (offset/limited reads) are treated as unread — the model must perform a full read before writing.

#### 2. Modification Time Staleness Check

The tool detects if a file has been modified since it was last read:

```typescript
const lastWriteTime = Math.floor(fileMtimeMs)
if (lastWriteTime > readTimestamp.timestamp) {
  return {
    result: false,
    message: 'File has been modified since read...',
    errorCode: 3,
  }
}
```

#### 3. Double-Check Before Write (Race Condition Prevention)

A second staleness check occurs immediately before the write inside the critical section:

```typescript
if (meta !== null) {
  const lastWriteTime = getFileModificationTime(fullFilePath)
  const lastRead = readFileState.get(fullFilePath)
  if (!lastRead || lastWriteTime > lastRead.timestamp) {
    // Content comparison fallback for full reads
    const isFullRead = lastRead && lastRead.offset === undefined && lastRead.limit === undefined
    if (!isFullRead || meta.content !== lastRead.content) {
      throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
    }
  }
}
```

This double-check pattern (TOCTOU mitigation) catches modifications that occur between `validateInput()` and `call()`. For full reads, it compares actual content as a fallback to avoid false positives from timestamp-only changes (cloud sync, antivirus scans on Windows).

#### 4. UNC Path Protection

On Windows, UNC paths (`\\server\share` or `//server/share`) trigger SMB authentication which could leak credentials. The tool skips filesystem operations for UNC paths and lets the permission system handle them:

```typescript
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true } // Skip stat check, let permission check handle it
}
```

#### 5. Team Memory Secret Guard

Before writing, the tool checks if the content contains secrets that should not be written to team memory files:

```typescript
const secretError = checkTeamMemSecrets(fullFilePath, content)
if (secretError) {
  return { result: false, message: secretError, errorCode: 0 }
}
```

#### 6. Path Validation

- `file_path` must be an absolute path (enforced by Zod schema description)
- Paths are expanded via `expandPath()` to resolve `~`, environment variables, and relative components

### Error Handling

The tool uses a structured error code system in `validateInput()`:

| Error Code | Condition | Message |
|------------|-----------|---------|
| `0` | Secret detected in team memory file | Secret-specific error message |
| `1` | Path matches a deny rule | "File is in a directory that is denied by your permission settings." |
| `2` | File not read before write | "File has not been read yet. Read it first before writing to it." |
| `3` | File modified since read | "File has been modified since read, either by the user or by a linter..." |

**Runtime errors** (thrown during `call()`):

| Error | Condition |
|-------|-----------|
| `FILE_UNEXPECTEDLY_MODIFIED_ERROR` | File content changed between read and write (double-check failure) |
| FS errors from `mkdir()` | Parent directory creation failure |
| FS errors from `writeTextContent()` | Write failure (permissions, disk full, etc.) |

**Error UI rendering**:

- Non-verbose mode: Shows "Error writing file" in red
- Verbose mode: Shows full fallback error message via `<FallbackToolUseErrorMessage />`

### Atomic Writes

The write operation uses `writeTextContent()` which performs an atomic write:

1. Content is written to a temporary file in the same directory
2. The temporary file is flushed to disk (`fsync`)
3. The temporary file is renamed over the target file (atomic on POSIX and Windows)

This ensures that:
- The target file is never left in a partially-written state
- Power loss or crash during write does not corrupt the file
- Concurrent readers see either the old content or the new content, never a mix

The atomic write is protected by the staleness check — if another process modifies the file between the check and the rename, the write throws `FILE_UNEXPECTEDLY_MODIFIED_ERROR` rather than silently overwriting.

## Data Flow

```
Model returns tool_use { name: 'Write', input: { file_path, content } }
  |
  v
backfillObservableInput() → expand path to absolute
  |
  v
preparePermissionMatcher() → build wildcard matcher for path
  |
  v
checkPermissions() → evaluate permission rules
  |
  v
validateInput() → security checks:
  ├── checkTeamMemSecrets() → reject if secrets detected
  ├── matchingRuleForInput() → reject if path is denied
  ├── UNC path check → skip stat for UNC paths
  ├── fs.stat() → check if file exists
  ├── readFileState.get() → verify file was read
  └── mtime comparison → verify file not modified
  |
  v
call() → execute write:
  ├── discoverSkillDirsForPaths() → background skill discovery
  ├── activateConditionalSkillsForPaths() → activate matching skills
  ├── diagnosticTracker.beforeFileEdited() → LSP prep
  ├── fs.mkdir(dir) → ensure parent directory
  ├── fileHistoryTrackEdit() → backup (if enabled)
  ├── readFileSyncWithMetadata() → read current state
  ├── staleness double-check → verify no concurrent modification
  ├── writeTextContent() → atomic write with LF line endings
  ├── LSP notifications → changeFile + saveFile
  ├── notifyVscodeFileUpdated() → VSCode integration
  ├── readFileState.set() → update read cache
  ├── logEvent('tengu_write_claudemd') → if CLAUDE.md
  ├── fetchSingleFileGitDiff() → compute diff (if remote)
  └── return { type, filePath, content, structuredPatch, originalFile, gitDiff }
  |
  v
mapToolResultToToolResultBlockParam() → serialize for API
  |
  v
renderToolResultMessage() → render in terminal UI
```

## Integration Points

### LSP (Language Server Protocol)

After writing, the tool notifies LSP servers:

1. **Clear diagnostics** — `clearDeliveredDiagnosticsForFile()` removes stale diagnostics
2. **Content change** — `lspManager.changeFile()` sends `textDocument/didChange`
3. **File save** — `lspManager.saveFile()` sends `textDocument/didSave` (triggers TypeScript server diagnostics)

Errors in LSP notifications are caught and logged but do not fail the write operation.

### VSCode Integration

`notifyVscodeFileUpdated(fullFilePath, oldContent, content)` informs VSCode about the file change, enabling the diff view and external editor synchronization.

### File History

When file history is enabled (`fileHistoryEnabled()`), the tool tracks edits via `fileHistoryTrackEdit()`. This creates a backup keyed on content hash, enabling undo functionality. The backup is created before the staleness check — if staleness fails later, the unused backup is harmless.

### Git Diff Computation

For remote sessions with the `tengu_quartz_lantern` feature flag enabled, the tool computes a git diff of the changes:

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_quartz_lantern', false)) {
  const diff = await fetchSingleFileGitDiff(fullFilePath)
  if (diff) gitDiff = diff
}
```

Duration is logged for performance monitoring.

### Dynamic Skill Discovery

Writing to a file can trigger skill discovery:

1. `discoverSkillDirsForPaths()` — finds skill directories matching the file's path
2. `addSkillDirectories()` — registers new skills (background, non-blocking)
3. `activateConditionalSkillsForPaths()` — activates skills whose path patterns match

This enables context-aware skill loading based on what files the model is editing.

### Analytics

- File operations are logged via `logFileOperation()` with operation type, tool name, file path, and create/update classification
- CLAUDE.md writes trigger a special `tengu_write_claudemd` event
- Git diff computation duration is tracked via `tengu_tool_use_diff_computed`

## Configuration

### Feature Flags

| Flag | Purpose |
|------|---------|
| `tengu_quartz_lantern` | Enables git diff computation for remote sessions |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_REMOTE` | When truthy, enables git diff computation |

### Tool Limits

| Setting | Value |
|---------|-------|
| `maxResultSizeChars` | 100,000 |
| `strict` | `true` (strict mode for tengu_tool_pear) |

## Error Handling

### Validation Errors (Pre-Execution)

Validation errors are caught before permission checks and returned to the model as error messages. The model can retry after addressing the issue (e.g., reading the file first).

### Concurrent Modification Errors

The tool implements a double-check pattern to detect concurrent modifications:

1. **First check** in `validateInput()` — compares file mtime against read timestamp
2. **Second check** in `call()` — re-reads file and compares content if timestamps differ

This TOCTOU (Time-of-Check-Time-of-Use) mitigation catches:
- User edits in an external editor
- Linter auto-fixes
- Cloud sync changes
- Antivirus file modifications

### Write Failures

Write failures (disk full, permission denied, etc.) propagate as thrown errors. The query engine catches these and renders them via `renderToolUseErrorMessage()`.

## Testing

### Error Code Testing

Each validation error code can be tested by setting up the corresponding precondition:

| Code | Test Setup |
|------|------------|
| `0` | Write to team memory file containing secrets |
| `1` | Write to a path matching a deny rule |
| `2` | Write to a file that was never read |
| `3` | Modify a file after reading but before writing |

### Edge Cases

- **UNC paths**: Should skip stat check and let permission system handle
- **Partial reads**: Should be treated as unread (require full read before write)
- **New files**: Should create parent directories automatically
- **Existing files**: Should require read-first and staleness checks
- **CLAUDE.md**: Should trigger special analytics event
- **Line endings**: Model-provided line endings should be preserved (not rewritten)

## Related Modules

- [FileReadTool](./FileReadTool.md) — Complementary read tool; files must be read before writing
- [FileEditTool](./FileEditTool.md) — Diff-based editing; preferred for modifications to existing files
- [Tool System](../01-core-modules/tool-system.md) — Core tool abstraction and registry
- [Permission System](../01-core-modules/permission-system.md) — Permission evaluation and rules
- [File History](../05-utils/file-history.md) — Backup and undo functionality

## Notes

- The tool name is `'Write'` (not `FileWriteTool`) — this is the name the model sees
- Plan files (in the plans directory) get special UI treatment: "Updated plan" name and `/plan to preview` hint
- Created files are truncated to 10 lines in the UI with a "… +N lines" indicator
- Updated files show the structured diff instead of full content
- The tool uses `z.strictObject()` for input validation — unknown fields are rejected
- Output schema includes `gitDiff` as optional — only present when feature flag is enabled
- The `backfillObservableInput` mutation is critical for security — it prevents permission bypass via path tricks