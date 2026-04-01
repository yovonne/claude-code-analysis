# GlobTool

## Purpose

The `GlobTool` (exposed to users as the **Search** tool) is a fast file pattern matching utility that finds files by name patterns or wildcards across any codebase size. It supports standard glob patterns like `**/*.js` or `src/**/*.ts`, returns matching file paths sorted by modification time, and integrates with the permission system for secure filesystem access. Results are truncated at a configurable limit (default 100) to prevent context bloat.

## Location

`restored-src/src/tools/GlobTool/GlobTool.ts`

Supporting files:
- `restored-src/src/tools/GlobTool/prompt.ts` — Tool name constant and description text
- `restored-src/src/tools/GlobTool/UI.tsx` — Terminal UI rendering for use/result/error messages

## Key Exports

### Functions

- `userFacingName()`: Returns `'Search'` — the human-readable tool name shown in the UI
- `renderToolUseMessage(input, { verbose })`: Renders the tool invocation display showing pattern and optional path
- `renderToolUseErrorMessage(result, { verbose })`: Renders error messages with user-friendly text for file-not-found and general search errors
- `renderToolResultMessage(output, progressMessages, { verbose })`: Renders search results (reuses `GrepTool`'s implementation)
- `getToolUseSummary(input)`: Returns a truncated summary of the glob pattern for activity display

### Types

- `Input`: Zod-inferred input schema — `{ pattern: string, path?: string }`
- `Output`: `{ durationMs: number, numFiles: number, filenames: string[], truncated: boolean }`

### Constants

| Constant | Value | Description |
|---|---|---|
| `GLOB_TOOL_NAME` | `'Glob'` | Internal tool identifier |
| `DESCRIPTION` | Multi-line string | Tool capabilities and usage guidance |
| `maxResultSizeChars` | `100_000` | Maximum result size for persistence |

## Dependencies

### Internal Dependencies

- `glob` (`src/utils/glob.ts`) — Core glob pattern matching engine
- `expandPath`, `toRelativePath` (`src/utils/path.ts`) — Path normalization and relativization
- `checkReadPermissionForTool` (`src/utils/permissions/filesystem.ts`) — Permission evaluation
- `matchWildcardPattern` (`src/utils/permissions/shellRuleMatching.ts`) — Wildcard matching for permission rules
- `getCwd` (`src/utils/cwd.ts`) — Current working directory resolution
- `getFsImplementation` (`src/utils/fsOperations.ts`) — Abstracted filesystem operations
- `suggestPathUnderCwd`, `FILE_NOT_FOUND_CWD_NOTE` (`src/utils/file.ts`) — Path suggestion on errors
- `isENOENT` (`src/utils/errors.ts`) — ENOENT error detection
- `lazySchema` (`src/utils/lazySchema.ts`) — Lazy Zod schema evaluation
- `GrepTool` (`src/tools/GrepTool/GrepTool.ts`) — Reused for result message rendering

### External Dependencies

- `zod/v4` — Input/output schema validation
- `react` — UI component rendering (Ink-based terminal UI)

## Implementation Details

### Core Logic

The tool is built using `buildTool()` and follows a streamlined execution pipeline:

1. **Input validation** (`validateInput`) — I/O-based check that the optional `path` parameter exists and is a directory. Skips filesystem operations for UNC paths to prevent NTLM credential leaks.
2. **Permission check** (`checkPermissions`) — Evaluates read permissions via `checkReadPermissionForTool()`.
3. **Permission matcher preparation** (`preparePermissionMatcher`) — Uses wildcard pattern matching against the glob pattern for rule evaluation.
4. **Execution** (`call`) — Delegates to the `glob()` utility with the pattern, resolved path, and a result limit.
5. **Result relativization** — Converts all absolute paths to relative paths under `cwd` to save tokens.

The `call()` method is the core execution:

```typescript
const { files, truncated } = await glob(
  input.pattern,
  GlobTool.getPath(input),  // resolved path or cwd
  { limit, offset: 0 },     // pagination params
  abortController.signal,   // cancellation support
  appState.toolPermissionContext,
)
```

### Input Schema

```typescript
z.strictObject({
  pattern: z.string(),       // The glob pattern to match files against
  path: z.string().optional(), // Directory to search in (defaults to cwd)
})
```

### Output Schema

```typescript
z.object({
  durationMs: z.number(),          // Time taken in milliseconds
  numFiles: z.number(),            // Total number of files found
  filenames: z.array(z.string()),  // Array of matching file paths (relative)
  truncated: z.boolean(),          // Whether results were truncated (limited to 100)
})
```

### Key Algorithms

#### Glob Pattern Matching

Delegates to the `glob()` utility in `src/utils/glob.ts`. The utility handles:
- Standard glob syntax: `*` (any chars except `/`), `**` (any chars including `/`), `?` (single char), `[abc]` (character class), `{a,b}` (alternation)
- Pattern compilation and caching for repeated use
- Recursive directory traversal with early termination on limit reach

#### File Discovery and Path Resolution

Path resolution follows a clear precedence:

1. If `path` is provided, it is expanded via `expandPath()` (tilde expansion, Windows separator normalization)
2. If `path` is omitted, `getCwd()` is used as the search root
3. All returned paths are converted to relative paths via `toRelativePath()` to reduce token usage

#### Result Truncation

Results are limited to a configurable maximum (default 100 files), controlled by `globLimits.maxResults`. The `truncated` boolean in the output signals when more matches exist than were returned, prompting the user to refine their pattern or path.

#### Input Validation

When a `path` is provided, validation performs:
1. **UNC path detection**: Paths starting with `\\` or `//` skip stat checks to prevent NTLM credential leaks
2. **Existence check**: `fs.stat()` verifies the path exists
3. **Directory check**: Confirms the path is a directory, not a file
4. **Suggestion on failure**: If the path doesn't exist, `suggestPathUnderCwd()` offers a similar path from the current working directory

### Edge Cases

- **No results**: Returns `"No files found"` as the tool result content
- **Truncated results**: Appends `"(Results are truncated. Consider using a more specific path or pattern.)"` to guide the user
- **UNC paths**: Validation skips filesystem stat calls for `\\server\share` or `//server/share` paths to prevent NTLM hash leakage
- **Invalid path type**: Returns error code 2 if the path exists but is not a directory
- **Non-existent path**: Returns error code 1 with a suggestion for a similar path under cwd

## Data Flow

```
User Request
    │
    ▼
┌─────────────────────┐
│  validateInput()    │  ← Path exists? Is directory? UNC skip?
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  checkPermissions() │  ← Read permission evaluation
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  call()             │  ← Delegate to glob() utility
│  glob(pattern,      │
│    path,            │
│    {limit, offset}, │
│    abortSignal,     │
│    permContext)     │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Relativize Paths   │  ← toRelativePath() on each match
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Result Mapping     │  ← mapToolResultToToolResultBlockParam
│  + Truncation Note  │
└─────────┬───────────┘
          │
          ▼
    Tool Result
```

## Integration Points

### GrepTool UI Reuse

`GlobTool` reuses `GrepTool.renderToolResultMessage` for rendering search results. Both tools produce filename lists, so the shared component displays `"Found N files"` summaries with expandable content via `Ctrl+O`.

### Permission System

- `preparePermissionMatcher()` creates a matcher function that compares permission rule patterns against the glob pattern using `matchWildcardPattern()`
- `checkPermissions()` delegates to `checkReadPermissionForTool()`, which evaluates the tool against configured read permission rules
- The tool is marked `isReadOnly: true`, reinforcing its non-mutating nature

### Auto-Classifier

`toAutoClassifierInput()` returns `input.pattern`, enabling the auto-classifier to categorize tool invocations by the glob pattern being searched.

### Search/Read Classification

`isSearchOrReadCommand()` returns `{ isSearch: true, isRead: false }`, classifying this as a search operation (not a read operation) for activity tracking and analytics.

### Concurrency Safety

`isConcurrencySafe()` returns `true`, allowing parallel execution with other tools. Glob operations are read-only and stateless.

## Configuration

### Result Limits

| Parameter | Default | Description |
|---|---|---|
| `globLimits.maxResults` | `100` | Maximum number of files returned per glob call |
| `maxResultSizeChars` | `100,000` | Character limit for persisting tool results |

### Environment Variables

No tool-specific environment variables. Limits are controlled through the `globLimits` context parameter passed by the runtime.

## Error Handling

### Validation Errors (Pre-I/O)

| Error Code | Condition | Message |
|---|---|---|
| 1 | Directory does not exist | "Directory does not exist: {path}. … Did you mean {suggestion}?" |
| 2 | Path is not a directory | "Path is not a directory: {path}" |

### Runtime Errors

| Error | Condition | Handling |
|---|---|---|
| `ENOENT` | Path not found during validation | Suggests similar paths under cwd |
| Abort | `abortController.signal` fired | Propagates cancellation to `glob()` utility |

### Error UI Rendering

`renderToolUseErrorMessage()` in `UI.tsx` provides user-friendly error display:
- Detects "file not found" pattern (containing `FILE_NOT_FOUND_CWD_NOTE`) and shows "File not found"
- Detects `<tool_use_error>` tags and shows "Error searching files"
- Falls back to `FallbackToolUseErrorMessage` for other errors

## Performance Optimization

### Result Limiting at Source

The `glob()` utility accepts a `limit` parameter, stopping traversal once the limit is reached. This avoids scanning the entire filesystem when only the first N matches are needed.

### Path Relativization

All returned paths are converted from absolute to relative (under `cwd`) via `toRelativePath()`. This reduces token usage in tool results, especially for deeply nested projects.

### Lazy Schema Evaluation

Input and output schemas use `lazySchema()` to defer Zod schema construction until first access, reducing cold-start overhead.

### AbortController Support

The `abortController.signal` is passed to the `glob()` utility, enabling cancellation of long-running searches when the user navigates away or the request is superseded.

### Concurrency Safety

Marked as concurrency-safe (`isConcurrencySafe: true`), allowing multiple glob searches to run in parallel without race conditions.

## Testing

No tool-specific test files are present in the `GlobTool/` directory. Testing is likely covered by integration tests in the broader tool test suite.

## Related Modules

- [GrepTool](./GrepTool.md) — Complementary content search tool; shares UI rendering
- [BashTool](./BashTool.md) — Used for directory listing and filesystem exploration
- [AgentTool](./AgentTool.md) — Used for open-ended searches requiring multiple rounds
- [glob utility](../05-utils/glob.md) — Core glob pattern matching engine

## Notes

- The tool name exposed to users is `'Search'` (via `userFacingName()`), while the internal identifier is `'Glob'`
- Results are sorted by modification time by the underlying `glob()` utility
- The truncation limit of 100 files is intentionally low to prevent context window bloat — users are encouraged to narrow their pattern or path for more targeted results
- For open-ended searches that may require multiple rounds of globbing and grepping, the tool description recommends using the `Agent` tool instead
- The tool reuses GrepTool's `renderToolResultMessage` because both produce filename lists with similar display requirements
