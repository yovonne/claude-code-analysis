# FileEditTool

## Purpose

Performs exact search-and-replace string substitutions in files. It is the primary tool for modifying existing file contents in place, supporting single or multi-occurrence replacements, file creation (via empty `old_string`), quote normalization, patch generation, and file history tracking.

## Location

`restored-src/src/tools/FileEditTool/FileEditTool.ts`

Supporting files:
- `types.ts` — Zod input/output schemas and TypeScript types
- `utils.ts` — Core edit logic: `findActualString`, `preserveQuoteStyle`, `applyEditToFile`, `getPatchForEdits`, snippet extraction, input normalization, equivalence checking
- `UI.tsx` — Terminal UI components for tool-use, result, rejected, and error messages (including inline diff rendering)
- `prompt.ts` — Tool description and usage instructions presented to the model
- `constants.ts` — Tool name and error message constants

## Key Exports

### Functions

- `FileEditTool` — The tool definition object built via `buildTool()`, containing all lifecycle hooks
- `findActualString` — Locates the search string in file content with quote normalization
- `preserveQuoteStyle` — Applies curly-quote style from `actualOldString` to `newString`
- `applyEditToFile` — Applies a single string replacement (single or global) to content
- `getPatchForEdit` / `getPatchForEdits` — Generates unified diff hunks from before/after content
- `normalizeFileEditInput` — Pre-normalizes inputs (desanitization, trailing whitespace stripping)
- `areFileEditsEquivalent` — Semantic equivalence check by applying both edit sets and comparing results
- `areFileEditsInputsEquivalent` — Unified equivalence check for two complete tool inputs
- `getSnippetForPatch` / `getSnippet` — Extracts context-window snippets around changed lines
- `getSnippetForTwoFileDiff` — Generates bounded diff snippets for attachments
- `getEditsForPatch` — Reverse-engineers `FileEdit[]` from structured patch hunks
- `normalizeQuotes` — Replaces curly quotes with straight quotes
- `stripTrailingWhitespace` — Removes trailing whitespace per line while preserving line endings

### Types

- `FileEditInput` — Parsed tool input: `{ file_path, old_string, new_string, replace_all }`
- `FileEdit` — Individual edit without file path: `{ old_string, new_string, replace_all }`
- `EditInput` — Same as `FileEdit` but with optional `replace_all`
- `FileEditOutput` — Tool result: `{ filePath, oldString, newString, originalFile, structuredPatch, userModified, replaceAll, gitDiff? }`

### Constants

- `FILE_EDIT_TOOL_NAME` — `'Edit'`
- `FILE_UNEXPECTEDLY_MODIFIED_ERROR` — Error message for concurrent file modification
- `MAX_EDIT_FILE_SIZE` — `1 GiB` (1024³ bytes), guard against OOM
- `DIFF_SNIPPET_MAX_BYTES` — `8192`, cap on diff snippet size for attachments
- `CONTEXT_LINES` — `4`, default context lines around changes in snippets
- `CHANGE_THRESHOLD` — `0.4`, threshold for word-level vs. line-level diff rendering
- Curly quote constants: `LEFT_SINGLE_CURLY_QUOTE`, `RIGHT_SINGLE_CURLY_QUOTE`, `LEFT_DOUBLE_CURLY_QUOTE`, `RIGHT_DOUBLE_CURLY_QUOTE`

## Dependencies

### Internal Dependencies

| Module | Purpose |
|--------|---------|
| `Tool.ts` | Tool builder, type definitions, permission framework |
| `utils/fileRead.ts` | `readFileSyncWithMetadata`, encoding and line-ending detection |
| `utils/file.ts` | `writeTextContent`, `findSimilarFile`, `suggestPathUnderCwd`, `getFileModificationTime`, `addLineNumbers`, `convertLeadingTabsToSpaces`, `readFileSyncCached` |
| `utils/diff.ts` | `getPatchForDisplay`, `getPatchFromContents`, `adjustHunkLineNumbers`, `countLinesChanged`, `DIFF_TIMEOUT_MS` |
| `utils/fileHistory.ts` | `fileHistoryEnabled`, `fileHistoryTrackEdit` for undo support |
| `utils/gitDiff.ts` | `fetchSingleFileGitDiff` for remote diff generation |
| `utils/permissions/filesystem.ts` | `checkWritePermissionForTool`, `matchingRuleForInput` |
| `utils/settings/validateEditTool.ts` | `validateInputForSettingsFileEdit` for CLAUDE.md protection |
| `services/lsp/` | LSP server notification (`changeFile`, `saveFile`, `clearDeliveredDiagnosticsForFile`) |
| `services/mcp/vscodeSdkMcp.ts` | `notifyVscodeFileUpdated` for VS Code diff view |
| `services/teamMemorySync/teamMemSecretGuard.ts` | `checkTeamMemSecrets` for secret detection |
| `skills/loadSkillsDir.ts` | Dynamic skill discovery from edited file paths |
| `components/StructuredDiff/` | Word-level diff highlighting in terminal UI |

### External Dependencies

| Package | Purpose |
|---------|---------|
| `diff` | `structuredPatch`, `diffWordsWithSpace` for unified diff generation and word-level highlighting |
| `zod/v4` | Input/output schema validation with `semanticBoolean` preprocessing |

## Implementation Details

### Search-and-Replace Implementation

The tool uses a **literal string search-and-replace** approach (not regex-based). The core replacement is performed by `applyEditToFile` in `utils.ts`:

```typescript
const f = replaceAll
  ? (content, search, replace) => content.replaceAll(search, () => replace)
  : (content, search, replace) => content.replace(search, () => replace)
```

Using a callback function (`() => replace`) instead of a direct string prevents `$&`, `$'`, `$\`` and other `String.prototype.replace` substitution patterns from being interpreted — ensuring the replacement text is inserted verbatim.

**Search resolution** follows a two-stage process in `findActualString`:

1. **Exact match** — `fileContent.includes(searchString)` for the fastest path
2. **Quote-normalized match** — If exact match fails, both file content and search string are passed through `normalizeQuotes` (curly → straight), then the index from the normalized search is used to extract the actual substring from the original file content

This allows edits to succeed even when the model outputs straight quotes (`'`, `"`) but the file contains curly quotes (`''`, `""`).

### Diff Generation

Diff generation uses the `diff` library's `structuredPatch` function, wrapped through two utility functions:

**`getPatchForEdits`** (primary path):
1. Applies each edit sequentially to build `updatedFile`
2. Detects substring conflicts between edits (an `old_string` that is a substring of a previous `new_string` throws an error)
3. Verifies each edit actually changed the content
4. Calls `getPatchFromContents` with tab-to-space-converted content for display-safe patches

**`getPatchFromContents`** (in `utils/diff.ts`):
- Calls `structuredPatch(oldPath, newPath, oldContent, newContent)` with `context: 3` (default unified diff context)
- Converts leading tabs to spaces so patches render consistently regardless of terminal tab width

The generated `StructuredPatchHunk[]` is returned in the tool output and used by UI components for terminal diff rendering and by the remote code path for git diff generation.

### Conflict Resolution

Multiple conflict scenarios are handled at different stages:

**Validation stage** (`validateInput`):

| Scenario | Error Code | Behavior |
|----------|-----------|----------|
| `old_string === new_string` | 1 | Rejects — no changes to make |
| Directory denied by permissions | 2 | Rejects — permission settings block write |
| File too large (>1 GiB) | 10 | Rejects — prevents OOM |
| File doesn't exist, `old_string` non-empty | 4 | Rejects — suggests similar file or CWD path |
| File exists, `old_string` empty (not creating) | 3 | Rejects — file already has content |
| Jupyter Notebook (`.ipynb`) | 5 | Rejects — redirects to `NotebookEditTool` |
| File not yet read | 6 | Rejects — requires prior `FileReadTool` call |
| File modified since read | 7 | Rejects — requires re-read |
| `old_string` not found (even after quote normalization) | 8 | Rejects — string not in file |
| Multiple matches, `replace_all` is false | 9 | Rejects — asks for more context or `replace_all: true` |

**Execution stage** (`call`):

| Scenario | Handling |
|----------|----------|
| File unexpectedly modified between validation and write | Throws `FILE_UNEXPECTEDLY_MODIFIED_ERROR` |
| Edit's `old_string` is substring of a previous edit's `new_string` | Throws error in `getPatchForEdits` |
| Edit produces no change | Throws "String not found in file" |
| All edits applied but file unchanged | Throws "Original and edited file match exactly" |

**Content-level conflict** — When `replace_all` is false and the search string appears multiple times, the tool refuses to guess which occurrence to replace. The model must provide more surrounding context to uniquely identify the target.

### Undo Support

File history is managed through `fileHistory.ts`:

1. **Pre-edit backup** — `fileHistoryTrackEdit` is called before the edit, capturing the pre-edit content keyed on a content hash. This is idempotent — if the same content is already backed up, it's a no-op.
2. **Feature-gated** — Only active when `fileHistoryEnabled()` returns true
3. **State tracking** — `updateFileHistoryState` callback receives the file path and parent message UUID for correlation
4. **Backup timing** — Called before the staleness check so that even if staleness validation fails, an unused backup is harmless (not corrupt state)

The file history system enables the `/rewind` command and other undo operations by maintaining a chain of content snapshots.

### Multiple Edit Handling

The `getPatchForEdits` function supports applying multiple edits to a single file in sequence:

1. Edits are applied **sequentially** in order, each building on the result of the previous
2. **Substring conflict detection** — Before applying each edit, the `old_string` is checked against all previously applied `new_string` values. If it's a substring of any previous replacement, an error is thrown to prevent cascading corruption
3. **Change verification** — After each edit, the result is compared to the pre-edit content. If no change occurred, the edit is considered failed
4. **Final verification** — After all edits, the final content is compared to the original. If identical, an error is thrown

The single-edit `call` method in `FileEditTool.ts` wraps the single `old_string`/`new_string` pair into a one-element array for `getPatchForEdits`.

### Error Handling

Errors are categorized by their source:

**Validation errors** — Return `{ result: false, behavior: 'ask', message, errorCode }` from `validateInput`. These are shown to the user with an option to retry.

**Execution errors** — Thrown as `Error` objects during `call()`. The key error:
- `FILE_UNEXPECTEDLY_MODIFIED_ERROR` — File was modified externally between read and write

**Permission errors** — Handled by the permission framework:
- UNC paths (`\\` or `//`) are skipped during filesystem checks to prevent NTLM credential leaks on Windows
- Directory deny rules are checked before any filesystem access
- Write permission is verified through `checkWritePermissionForTool`

**UI error rendering** — `renderToolUseErrorMessage` simplifies common errors:
- "File has not been read yet" → "File must be read first" (dimmed)
- "File does not exist" → "File not found" (error color)
- Other errors → "Error editing file" (error color)

**Desanitization fallback** — When exact string match fails, `normalizeFileEditInput` tries desanitizing the `old_string` using a lookup table of sanitized tokens (e.g., `<fnr>` → `<function_results>`, `<n>` → `<name>`). If desanitization succeeds, the same replacements are applied to `new_string`.

## Data Flow

```
Model Output
    │
    ▼
┌─────────────────────────────────┐
│  normalizeFileEditInput          │  Desanitize, strip trailing whitespace
│  (utils.ts)                      │  (skip for .md/.mdx)
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  validateInput                   │  Check permissions, file state,
│  (FileEditTool.ts)               │  string match, multiplicity
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  checkPermissions                │  Write permission verification
│  (FileEditTool.ts)               │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  call()                          │
│  ┌───────────────────────────┐  │
│  │ fileHistoryTrackEdit      │  │  Backup pre-edit content
│  ├───────────────────────────┤  │
│  │ readFileForEdit           │  │  Read with encoding + line endings
│  ├───────────────────────────┤  │
│  │ Staleness check           │  │  Verify file unchanged since read
│  ├───────────────────────────┤  │
│  │ findActualString          │  │  Locate string (with quote norm)
│  ├───────────────────────────┤  │
│  │ preserveQuoteStyle        │  │  Apply curly quote style to new
│  ├───────────────────────────┤  │
│  │ getPatchForEdit           │  │  Generate unified diff hunks
│  ├───────────────────────────┤  │
│  │ writeTextContent          │  │  Atomic write preserving encoding
│  ├───────────────────────────┤  │
│  │ LSP notification          │  │  didChange + didSave
│  ├───────────────────────────┤  │
│  │ VS Code notification      │  │  notifyVscodeFileUpdated
│  ├───────────────────────────┤  │
│  │ Update readFileState      │  │  Invalidate stale writes
│  ├───────────────────────────┤  │
│  │ Analytics logging         │  │  Line counts, byte lengths
│  ├───────────────────────────┤  │
│  │ Git diff (remote only)    │  │  fetchSingleFileGitDiff
│  └───────────────────────────┘  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  renderToolResultMessage         │  Terminal diff display via
│  (UI.tsx)                        │  FileEditToolUpdatedMessage
└─────────────────────────────────┘
```

## Integration Points

### LSP (Language Server Protocol)

After writing, the tool notifies LSP servers:
1. `clearDeliveredDiagnosticsForFile` — Clears previous diagnostics so new ones will be shown
2. `lspManager.changeFile` — Sends `textDocument/didChange` notification
3. `lspManager.saveFile` — Sends `textDocument/didSave` notification (triggers TypeScript diagnostics)

Both LSP calls are fire-and-forget with error logging — failures do not block the tool result.

### VS Code Integration

`notifyVscodeFileUpdated` is called with the file path, original content, and updated content. This enables VS Code to show a diff view of the changes when Claude Code runs as a VS Code extension.

### Skill Discovery

After editing a file, the tool discovers and activates skills related to the file's path:
- `discoverSkillDirsForPaths` — Finds skill directories matching the edited file
- `addSkillDirectories` — Registers new skill directories (background, non-blocking)
- `activateConditionalSkillsForPaths` — Activates skills whose path patterns match

Skipped in simple mode (`CLAUDE_CODE_SIMPLE` env var).

### File Read State

The tool maintains a `readFileState` map tracking:
- `timestamp` — When the file was last read
- `content` — The content at read time (for full reads)
- `offset` / `limit` — For partial reads (chunked reading)

This enables staleness detection: if the file's modification time exceeds the read timestamp, the write is rejected unless the content is verified unchanged (full read comparison fallback).

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | Disables skill discovery when truthy |
| `CLAUDE_CODE_REMOTE` | Enables git diff generation when truthy |

### Feature Flags

| Flag | Purpose |
|------|---------|
| `tengu_quartz_lantern` | Controls git diff computation for remote sessions |

### Size Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_EDIT_FILE_SIZE` | 1 GiB | Maximum file size that can be edited (prevents OOM) |
| `maxResultSizeChars` | 100,000 | Maximum characters in tool result |
| `DIFF_SNIPPET_MAX_BYTES` | 8,192 | Maximum diff snippet size in attachments |

## Error Handling

### Error Code Reference

| Code | Condition | User Message |
|------|-----------|-------------|
| 0 | Secret detected in team memory file | Secret guard error message |
| 1 | `old_string === new_string` | "No changes to make: old_string and new_string are exactly the same." |
| 2 | Directory denied by permissions | "File is in a directory that is denied by your permission settings." |
| 3 | Empty `old_string` on non-empty file | "Cannot create new file - file already exists." |
| 4 | File doesn't exist | "File does not exist." (+ suggestion) |
| 5 | Jupyter Notebook file | "File is a Jupyter Notebook. Use the NotebookEdit tool." |
| 6 | File not read before edit | "File has not been read yet. Read it first before writing to it." |
| 7 | File modified since read | "File has been modified since read... Read it again before attempting to write it." |
| 8 | `old_string` not found | "String to replace not found in file." |
| 9 | Multiple matches, `replace_all` false | "Found N matches... set replace_all to true" |
| 10 | File exceeds 1 GiB | "File is too large to edit (X). Maximum editable file size is 1 GiB." |

### Staleness Detection (Double-Check Pattern)

The tool performs staleness checks at **two points**:

1. **`validateInput`** — Before permission check, using `readFileState` timestamp and content comparison
2. **`call()`** — After acquiring current state, using a second timestamp check

Between these two points, `readFileForEdit` performs a synchronous read, creating a narrow critical section. The comment in the code explicitly warns: *"Please avoid async operations between here and writing to disk to preserve atomicity."*

The content comparison fallback prevents false positives on Windows where timestamps can change without content changes (cloud sync, antivirus scanning).

## Edge Cases

### Encoding Handling

**Detection** — `readFileSyncWithMetadata` reads the first 4096 bytes and checks for:
- UTF-16 LE BOM (`0xFF 0xFE`) → `'utf16le'`
- UTF-8 BOM (`0xEF 0xBB 0xBF`) → `'utf8'`
- Default → `'utf8'` (superset of ASCII, handles all Unicode)
- Empty file → `'utf8'` (prevents emoji/CJK corruption)

**Preservation** — The detected encoding is passed to `writeTextContent`, which writes the file back in the same encoding. This ensures UTF-16 files remain UTF-16 after editing.

**Validation stage** — Reads file as bytes and detects encoding identically, normalizing `\r\n` → `\n` for consistent internal representation.

### Line Ending Handling

**Detection** — `detectLineEndingsForString` samples the first 4096 characters and counts CRLF vs. LF occurrences. The dominant style is recorded.

**Normalization** — File content is normalized to LF (`\n`) internally by `readFileSyncWithMetadata` (`raw.replaceAll('\r\n', '\n')`).

**Preservation** — The detected line ending style (`'CRLF'` or `'LF'`) is passed to `writeTextContent`, which converts back before writing. This ensures Windows files (CRLF) remain CRLF after editing.

**`stripTrailingWhitespace`** — Preserves line endings by splitting on `/(\r\n|\n|\r)/` and only stripping content lines, not line-ending separators.

### Quote Normalization

**Problem** — Claude models cannot output curly/smart quotes. If a file contains `''hello''` and the model requests replacing `'hello'`, the straight quotes won't match.

**Solution** — Three-stage pipeline:
1. `findActualString` — Normalizes both file and search to straight quotes for matching, then extracts the actual substring from the original file
2. `preserveQuoteStyle` — Detects curly quotes in the matched `actualOldString` and applies the same open/close curly quote style to `newString`
3. Open/close heuristic — A quote preceded by whitespace, start of string, or opening punctuation `([{` is treated as opening; otherwise closing

**Contraction handling** — `applyCurlySingleQuotes` detects apostrophes between two letters (e.g., `don't`, `it's`) and always uses the right single curly quote (`'`), not the left.

### Markdown Trailing Whitespace

Markdown and MDX files use two trailing spaces as hard line breaks. `normalizeFileEditInput` skips `stripTrailingWhitespace` for `.md`/`.mdx` files to preserve this semantic meaning.

### Empty String Edits

- **Creating a new file** — `old_string: ''` on a non-existent file is valid and creates the file with `new_string` as content
- **Writing to an empty file** — `old_string: ''` on an existing empty file is valid
- **Writing to a non-empty file** — `old_string: ''` on a file with content is rejected (error code 3)
- **Deleting content** — `new_string: ''` is valid and removes the matched `old_string`
- **Trailing newline on deletion** — When `new_string` is empty and `old_string` doesn't end with `\n` but the file has `old_string + '\n'`, the trailing newline is also removed

### Desanitization

When Claude's output contains sanitized tokens (escaped by the API), `normalizeFileEditInput` reverses them:

| Sanitized | Original |
|-----------|----------|
| `<fnr>` | `<function_results>` |
| `<n>` / `</n>` | `<name>` / `</name>` |
| `<o>` / `</o>` | `<output>` / `</output>` |
| `<e>` / `</e>` | `<error>` / `</error>` |
| `<s>` / `</s>` | `<system>` / `</system>` |
| `<r>` / `</r>` | `<result>` / `</result>` |
| `< META_START >` | `<META_START>` |
| `< META_END >` | `<META_END>` |
| `< EOT >` | `<EOT>` |
| `< META >` | `<META>` |
| `< SOS >` | `<SOS>` |
| `\n\nH:` | `\n\nHuman:` |
| `\n\nA:` | `\n\nAssistant:` |

### UNC Path Security

On Windows, paths starting with `\\` or `//` are UNC paths. Calling `fs.existsSync()` on UNC paths triggers SMB authentication, which could leak NTLM credentials to malicious servers. The tool skips filesystem checks for UNC paths and lets the permission framework handle them.

### V8 String Length Limit

The 1 GiB file size limit is derived from V8/Bun's string length limit of ~2³⁰ characters (~1 billion). For ASCII/Latin-1 files, 1 byte = 1 character, so 1 GiB on disk ≈ 1 billion characters. Multi-byte UTF-8 files can be larger on disk per character, but 1 GiB is a safe byte-level guard.

### Jupyter Notebooks

`.ipynb` files are explicitly rejected with a redirect to `NotebookEditTool`. Jupyter Notebooks are JSON structures, not plain text, and require cell-aware editing.

### Settings File Protection

`validateInputForSettingsFileEdit` applies additional validation for Claude settings files (e.g., `CLAUDE.md`). It simulates the edit and validates the result against JSON/schema constraints to prevent corrupting configuration files.

## Testing

The tool's behavior is verified through:

- **Input equivalence checking** — `areFileEditsInputsEquivalent` compares two edit inputs semantically by applying both to the original content and comparing results, handling cases where different edit specifications produce the same outcome
- **Error code specificity** — Each validation failure returns a distinct error code for precise error handling and analytics
- **Patch round-tripping** — `getEditsForPatch` can reverse-engineer edits from a structured patch, enabling verification of patch correctness

## Related Modules

- [FileReadTool](./FileReadTool.md) — Required prerequisite for file edits
- [FileWriteTool](./FileWriteTool.md) — Alternative for full file replacement
- [NotebookEditTool](./NotebookEditTool.md) — Specialized editor for Jupyter Notebooks
- [tool-system](../01-core-modules/tool-system.md) — Tool abstraction framework
- [StructuredDiff](../06-ui/StructuredDiff.md) — Terminal diff rendering components

## Notes

- The tool requires files to be read before editing. This is enforced by the `readFileState` tracking and prevents blind overwrites.
- The `strict: true` flag on the tool definition enforces strict Zod schema validation.
- `semanticBoolean` preprocessing on `replace_all` allows the model to pass truthy/falsy values (e.g., `0`, `1`, `"true"`) that are coerced to proper booleans.
- Git diff generation (`fetchSingleFileGitDiff`) is only performed in remote sessions (`CLAUDE_CODE_REMOTE`) when the `tengu_quartz_lantern` feature flag is enabled.
- The tool uses synchronous file reads in the critical section between staleness check and write to prevent TOCTOU race conditions with concurrent edits.
- Analytics events logged: `tengu_write_claudemd` (for CLAUDE.md edits), `tengu_edit_string_lengths` (byte lengths of old/new strings), `tengu_tool_use_diff_computed` (git diff timing).
