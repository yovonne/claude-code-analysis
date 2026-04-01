# FileReadTool

## Purpose

The `FileReadTool` (exposed to users as the **Read** tool) is the primary file reading mechanism in Claude Code. It reads text files, images, PDFs, and Jupyter notebooks from the local filesystem, returning content with line numbers, metadata, and appropriate formatting for each file type. It includes deduplication, token budgeting, permission checks, and multi-format rendering.

## Location

`restored-src/src/tools/FileReadTool/FileReadTool.ts`

Supporting files:
- `restored-src/src/tools/FileReadTool/limits.ts` — Token and size limit configuration
- `restored-src/src/tools/FileReadTool/prompt.ts` — Tool prompt template and constants
- `restored-src/src/tools/FileReadTool/UI.tsx` — Terminal UI rendering for use/result/error messages
- `restored-src/src/tools/FileReadTool/imageProcessor.ts` — Sharp/image-processor-napi abstraction

## Key Exports

### Functions

- `registerFileReadListener(listener)`: Registers a callback invoked whenever a file is read. Returns an unsubscribe function.
- `readImageWithTokenBudget(filePath, maxTokens, maxBytes?)`: Reads an image file and applies token-based compression if needed. Handles resize, aggressive compression, and fallback paths.
- `getDefaultFileReadingLimits()`: Memoized function returning the current `FileReadingLimits` configuration (precedence: env var > GrowthBook > hardcoded defaults).

### Classes

- `MaxFileReadTokenExceededError`: Custom error thrown when file content exceeds the token budget. Includes `tokenCount` and `maxTokens` properties for programmatic handling.

### Types

- `Input`: Zod-inferred input schema — `{ file_path: string, offset?: number, limit?: number, pages?: string }`
- `Output`: Discriminated union of result types: `text`, `image`, `notebook`, `pdf`, `parts`, `file_unchanged`
- `FileReadingLimits`: `{ maxTokens: number, maxSizeBytes: number, includeMaxSizeInPrompt?: boolean, targetedRangeNudge?: boolean }`
- `ImageResult`: `{ type: 'image', file: { base64, type, originalSize, dimensions? } }`

### Constants

| Constant | Value | Description |
|---|---|---|
| `FILE_READ_TOOL_NAME` | `'Read'` | Internal tool identifier |
| `FILE_UNCHANGED_STUB` | `'File unchanged since last read…'` | Message returned on dedup hit |
| `MAX_LINES_TO_READ` | `2000` | Default line count when no `limit` specified |
| `DEFAULT_MAX_OUTPUT_TOKENS` | `25000` | Default token budget for text reads |
| `DESCRIPTION` | `'Read a file from the local filesystem.'` | Short tool description |
| `LINE_FORMAT_INSTRUCTION` | `'Results are returned using cat -n format…'` | Line numbering instruction |
| `OFFSET_INSTRUCTION_DEFAULT` | `'- You can optionally specify a line offset…'` | Default offset guidance |
| `OFFSET_INSTRUCTION_TARGETED` | `'- When you already know which part…'` | Targeted-range nudge variant |
| `CYBER_RISK_MITIGATION_REMINDER` | System reminder text | Appended to text results for non-exempt models |
| `BLOCKED_DEVICE_PATHS` | Set of paths | Device files that would hang or block (e.g., `/dev/zero`, `/dev/stdin`) |
| `IMAGE_EXTENSIONS` | `Set(['png','jpg','jpeg','gif','webp'])` | Recognized image file extensions |
| `MITIGATION_EXEMPT_MODELS` | `Set(['claude-opus-4-6'])` | Models that skip the cyber risk reminder |

## Dependencies

### Internal Dependencies

- `readFileInRange` (`src/utils/readFileInRange.ts`) — Core line-oriented file reader with fast and streaming code paths
- `checkReadPermissionForTool` (`src/utils/permissions/filesystem.ts`) — Permission evaluation
- `expandPath` (`src/utils/path.ts`) — Path normalization (tilde expansion, whitespace trimming, Windows separators)
- `countTokensWithAPI`, `roughTokenCountEstimationForFileType` (`src/services/tokenEstimation.ts`) — Token budgeting
- `readNotebook`, `mapNotebookCellsToToolResult` (`src/utils/notebook.ts`) — Jupyter notebook handling
- `extractPDFPages`, `getPDFPageCount`, `readPDF` (`src/utils/pdf.ts`) — PDF reading and page extraction
- `maybeResizeAndDownsampleImageBuffer`, `compressImageBufferWithTokenLimit` (`src/utils/imageResizer.ts`) — Image compression pipeline
- `logFileOperation` (`src/utils/fileOperationAnalytics.ts`) — File operation telemetry
- `discoverSkillDirsForPaths`, `activateConditionalSkillsForPaths` (`src/skills/loadSkillsDir.ts`) — Skill discovery from file paths
- `findSimilarFile`, `suggestPathUnderCwd`, `addLineNumbers` (`src/utils/file.ts`) — File-not-found assistance and content formatting
- `getFeatureValue_CACHED_MAY_BE_STALE` (`src/services/analytics/growthbook.ts`) — Feature flag evaluation for limits and dedup killswitch

### External Dependencies

- `zod/v4` — Input/output schema validation
- `fs/promises` — Async file system operations
- `sharp` — Image processing (resize, compression, format conversion)
- `image-processor-napi` — Native image processor for bundled builds
- `lodash-es/memoize` — Memoization for limit configuration

## Implementation Details

### Core Logic

The tool is built using `buildTool()` and follows a multi-stage execution pipeline:

1. **Input validation** (`validateInput`) — Pure checks with no I/O: pages parsing, deny rules, UNC path detection, binary extension check, blocked device path check
2. **Permission check** (`checkPermissions`) — Evaluates read permissions against configured rules
3. **Deduplication** — Checks `readFileState` cache for identical file+range+unchanged-mtime
4. **Skill discovery** — Discovers and activates skills matching the file's directory (fire-and-forget)
5. **Inner call** (`callInner`) — Type-specific read dispatch based on file extension

The `callInner` function dispatches to four distinct handlers:

| File Type | Extension | Handler | Output Type |
|---|---|---|---|
| Jupyter Notebook | `.ipynb` | `readNotebook()` | `notebook` |
| Image | `.png,.jpg,.jpeg,.gif,.webp` | `readImageWithTokenBudget()` | `image` |
| PDF | `.pdf` | `readPDF()` / `extractPDFPages()` | `pdf` or `parts` |
| Text (default) | All others | `readFileInRange()` | `text` |

### Input Schema

```typescript
z.strictObject({
  file_path: z.string(),                    // Absolute path to the file
  offset: z.number().int().nonnegative(),   // 1-indexed line to start from
  limit: z.number().int().positive(),       // Number of lines to read
  pages: z.string(),                        // PDF page range (e.g., "1-5", "3")
})
```

### Output Schema (Discriminated Union)

```typescript
z.discriminatedUnion('type', [
  // Text file result
  z.object({ type: 'text', file: { filePath, content, numLines, startLine, totalLines } }),
  // Image result with base64 data
  z.object({ type: 'image', file: { base64, type, originalSize, dimensions? } }),
  // Jupyter notebook result
  z.object({ type: 'notebook', file: { filePath, cells } }),
  // Full PDF result
  z.object({ type: 'pdf', file: { filePath, base64, originalSize } }),
  // Extracted PDF pages
  z.object({ type: 'parts', file: { filePath, originalSize, count, outputDir } }),
  // Dedup hit — file unchanged since last read
  z.object({ type: 'file_unchanged', file: { filePath } }),
])
```

### Key Algorithms

#### Deduplication Strategy

When a file is read, its content, modification timestamp, offset, and limit are stored in `readFileState` (a `Map<string, FileState>`). On subsequent reads:

1. Look up the file path in `readFileState`
2. Check if the stored entry is a full view (not a partial read from Edit/Write)
3. Compare the requested offset/limit with the stored range
4. Re-stat the file on disk and compare `mtimeMs` with the stored timestamp
5. If all match, return `FILE_UNCHANGED_STUB` instead of re-sending content

This optimization reduces cache_creation token usage by ~18% on Read calls. Controlled by the `tengu_read_dedup_killswitch` feature flag.

#### Token Budgeting

Two caps apply to text reads:

| Limit | Default | Check Point | Cost | On Overflow |
|---|---|---|---|---|
| `maxSizeBytes` | 256 KB | Pre-read (file stat) | 1 stat call | Throws before read |
| `maxTokens` | 25,000 | Post-read (content analysis) | API roundtrip | Throws after read |

Token estimation uses a two-phase approach:
1. **Fast estimate**: `roughTokenCountEstimationForFileType()` — cheap heuristic based on file type
2. **Precise count**: `countTokensWithAPI()` — only called if the fast estimate exceeds `maxTokens / 4`

This avoids unnecessary API calls for files that are clearly under budget.

#### Image Processing with Token Budget

The `readImageWithTokenBudget()` function implements a tiered compression strategy:

1. **Read once**: File is read a single time into a buffer (capped by `maxBytes` to prevent OOM)
2. **Format detection**: `detectImageFormatFromBuffer()` identifies the image format from magic bytes
3. **Standard resize**: `maybeResizeAndDownsampleImageBuffer()` applies normal resizing
4. **Token check**: Estimated tokens = `base64.length * 0.125`
5. **Aggressive compression**: If over budget, `compressImageBufferWithTokenLimit()` applies stronger compression from the same buffer (no re-read)
6. **Fallback**: If compression fails, a hard-coded 400x400 JPEG at quality 20 is produced via sharp
7. **Final fallback**: If sharp fails, the original buffer is returned as-is

#### Screenshot Path Resolution (macOS)

macOS screenshot filenames use either a regular space or thin space (U+202F) before AM/PM depending on the OS version. `getAlternateScreenshotPath()` detects this pattern and returns the alternate path to try if the original path returns ENOENT.

### Edge Cases

- **Blocked device paths**: Files like `/dev/zero`, `/dev/random`, `/dev/stdin` are blocked by path check (no I/O) to prevent hangs
- **UNC paths**: Network paths (`\\server\share` or `//server/share`) pass validation but defer actual I/O until after permission grant (prevents NTLM credential leaks)
- **Binary files**: Rejected with a helpful message, except for PDF, images, and SVG which are natively supported
- **Empty files**: Return a system-reminder warning instead of empty content
- **Offset beyond file length**: Returns a warning indicating the file is shorter than the requested offset
- **macOS screenshot spaces**: Tries alternate space character (regular vs thin space) for AM/PM in filenames
- **Large PDFs**: PDFs exceeding `PDF_AT_MENTION_INLINE_THRESHOLD` pages require a `pages` parameter
- **PDF extraction fallback**: When PDF.js is unsupported or file exceeds `PDF_EXTRACT_SIZE_THRESHOLD`, falls back to page extraction via poppler-utils

## Data Flow

```
User Request
    │
    ▼
┌─────────────────────┐
│  validateInput()    │  ← Pure checks: pages, deny rules, UNC, binary, devices
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  checkPermissions() │  ← Permission evaluation against rules
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Dedup Check        │  ← readFileState cache + mtime comparison
│  (optional stub)    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Skill Discovery    │  ← Fire-and-forget skill activation
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  callInner()        │  ← Extension-based dispatch
│  ┌───────────────┐  │
│  │ .ipynb → read │  │
│  │ image → read  │  │
│  │ .pdf → read   │  │
│  │ text → read   │  │
│  └───────────────┘  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Token Validation   │  ← Fast estimate → API count (if needed)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Result Mapping     │  ← mapToolResultToToolResultBlockParam
│  + Line Numbers     │
│  + Mitigation Note  │
└─────────┬───────────┘
          │
          ▼
    Tool Result
```

## Integration Points

### FileReadListener System

Other services can subscribe to file read events via `registerFileReadListener()`. This is used by:
- Memory system — tracks which files have been read for context awareness
- Analytics — logs file read patterns and session file access

### Session File Detection

`detectSessionFileType()` identifies when Claude reads its own session files:
- **Session memory**: `~/.claude/session-memory/*.md` — logged with `is_session_memory: true`
- **Session transcript**: `~/.claude/projects/*/*.jsonl` — logged with `is_session_transcript: true`

This enables analytics on self-referential reads and potential feedback loops.

### Memory Attachment Triggers

After a successful read, the file path is added to `context.nestedMemoryAttachmentTriggers`, signaling the memory system to consider this file for context attachment in future turns.

### Auto-Memory Freshness

For files detected as auto-memory files (`isAutoMemFile()`), a `WeakMap` stores the mtime, which is later used by `memoryFileFreshnessPrefix()` to append a freshness note to the rendered result.

## Configuration

### Environment Variables

| Variable | Description |
|---|---|
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | Override the default max token budget (25,000). Must be a positive integer. |
| `CLAUDE_CODE_SIMPLE` | When set, disables skill discovery during file reads. |

### Feature Flags

| Flag | Description | Default |
|---|---|---|
| `tengu_amber_wren` | GrowthBook experiment for custom limits (`maxSizeBytes`, `maxTokens`, `includeMaxSizeInPrompt`, `targetedRangeNudge`) | `{}` (use hardcoded defaults) |
| `tengu_read_dedup_killswitch` | Disables the deduplication optimization if it causes model confusion | `false` (dedup enabled) |

### Limit Precedence (maxTokens)

```
Env var → GrowthBook flag → DEFAULT_MAX_OUTPUT_TOKENS (25,000)
```

Each field is individually validated; invalid values fall through to the next tier. There is no route to `maxTokens = 0`.

## Error Handling

### Validation Errors (Pre-I/O)

| Error Code | Condition | Message |
|---|---|---|
| 1 | File path matches a deny rule | "File is in a directory that is denied by your permission settings." |
| 4 | Binary file extension (non-PDF/image) | "This tool cannot read binary files…" |
| 7 | Invalid pages parameter format | "Invalid pages parameter…" |
| 8 | Page range exceeds `PDF_MAX_PAGES_PER_READ` | "Page range exceeds maximum…" |
| 9 | Blocked device path | "Cannot read… this device file would block…" |

### Runtime Errors

| Error | Condition | Handling |
|---|---|---|
| `MaxFileReadTokenExceededError` | Content exceeds token budget | Includes actual count and max; suggests using `offset`/`limit` |
| `ENOENT` | File not found | Tries macOS screenshot alternate path, then suggests similar files or CWD-relative paths |
| Image resize error | Image processing failure | Falls back to uncompressed buffer, then to aggressive compression, then to original |
| PDF read error | PDF parsing failure | Throws with descriptive message; suggests page extraction or poppler-utils install |
| Notebook size error | Serialized cells exceed `maxSizeBytes` | Suggests using Bash tool with `jq` to read specific portions |

### Error UI Rendering

`renderToolUseErrorMessage()` in `UI.tsx` provides user-friendly error display:
- Detects "file not found" pattern and shows a concise "File not found" message
- Detects `<tool_use_error>` tags and shows "Error reading file"
- Falls back to `FallbackToolUseErrorMessage` for other errors

## Performance Optimization

### Single-Read Image Pipeline

Images are read exactly once into a buffer. All compression operations (standard resize, aggressive compression, fallback) operate on the same buffer — no re-reads. This eliminates redundant I/O for images that need multiple compression attempts.

### Fast vs Streaming File Read

`readFileInRange()` internally chooses between two strategies:
- **Fast path**: For files within size limits, reads the entire file and slices lines in memory
- **Streaming path**: For large files, streams line-by-line using a `createReadStream`, avoiding loading the full file into memory

### Token Estimation Caching

The two-phase token estimation avoids expensive API calls:
1. Fast heuristic (`roughTokenCountEstimationForFileType`) runs first — O(n) string scan
2. API call (`countTokensWithAPI`) only fires if the estimate exceeds `maxTokens / 4`

### Memoized Limit Configuration

`getDefaultFileReadingLimits()` is memoized via `lodash-es/memoize`. The GrowthBook flag value is fixed at first call, preventing the cap from changing mid-session as flags refresh in the background.

### WeakMap for Memory File Mtimestamps

The `memoryFileMtimes` WeakMap stores mtime data keyed by the result data object identity. This avoids:
- Adding presentation-only fields to the output schema (which would flow into SDK types)
- Synchronous `fs.stat` calls in the result mapper
- Memory leaks (WeakMap entries are auto-GC'd when the data object becomes unreachable)

### Background Skill Loading

Skill discovery (`discoverSkillDirsForPaths`) and activation (`activateConditionalSkillsForPaths`) are fire-and-forget operations that do not block the file read response. Skills are loaded asynchronously in the background.

## Testing

The tool includes a render-fidelity test that verifies UI.tsx summary rendering — ensuring it displays metadata ("Read N lines", "Read image (42KB)") rather than raw content. This test caught an initial implementation error where `file.content` was incorrectly used in the summary chrome.

## Related Modules

- [FileWriteTool](./FileWriteTool.md) — Complementary file writing tool
- [FileEditTool](./FileEditTool.md) — Complementary file editing tool
- [BashTool](./BashTool.md) — Used for directory listing and binary file inspection
- [readFileInRange](../05-utils/file-state-cache.md) — Core line-oriented file reading utility
- [tool-system](../01-core-modules/tool-system.md) — Tool abstraction and registry

## Notes

- The tool is marked as `isConcurrencySafe: true` and `isReadOnly: true`, indicating it can be called in parallel with other tools and does not modify state
- `maxResultSizeChars` is set to `Infinity` because output is bounded by `maxTokens` validation — persisting tool results to files would be circular
- The `backfillObservableInput` hook expands relative paths to absolute paths, preventing permission allowlist bypass via `~` or relative path tricks
- PDF content is sent as a supplemental `DocumentBlockParam` to the API, while the tool result block contains only metadata (file path and size)
- The cyber risk mitigation reminder is appended to text file results for all models except those in `MITIGATION_EXEMPT_MODELS` (currently only `claude-opus-4-6`)
