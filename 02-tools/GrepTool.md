# GrepTool

## Purpose

The `GrepTool` (exposed to users as the **Search** tool) is a powerful content search utility built on ripgrep. It searches file contents using regular expressions across any codebase size, with support for glob-based file filtering, context lines, multiple output modes, and pagination. It automatically excludes version control directories, respects `.gitignore` patterns, and applies intelligent result limits to prevent context window bloat. The tool is the primary search mechanism — users are instructed to always use it instead of invoking `grep` or `rg` via Bash.

## Location

`restored-src/src/tools/GrepTool/GrepTool.ts`

Supporting files:
- `restored-src/src/tools/GrepTool/prompt.ts` — Tool name constant and dynamic description generator
- `restored-src/src/tools/GrepTool/UI.tsx` — Terminal UI rendering with `SearchResultSummary` component

## Key Exports

### Functions

- `getDescription()`: Returns the full tool description with usage instructions, regex syntax notes, and ripgrep-specific guidance
- `renderToolUseMessage(input, { verbose })`: Renders the tool invocation display showing pattern, path, and optional filters
- `renderToolUseErrorMessage(result, { verbose })`: Renders error messages with user-friendly text for file-not-found and general search errors
- `renderToolResultMessage(output, progressMessages, { verbose })`: Renders search results with mode-specific summaries (content lines, match counts, or file lists)
- `getToolUseSummary(input)`: Returns a truncated summary of the search pattern for activity display

### Types

- `Input`: Zod-inferred input schema — `{ pattern: string, path?: string, glob?: string, output_mode?: 'content' | 'files_with_matches' | 'count', -B?: number, -A?: number, -C?: number, context?: number, -n?: boolean, -i?: boolean, type?: string, head_limit?: number, offset?: number, multiline?: boolean }`
- `Output`: `{ mode?: 'content' | 'files_with_matches' | 'count', numFiles: number, filenames: string[], content?: string, numLines?: number, numMatches?: number, appliedLimit?: number, appliedOffset?: number }`

### Constants

| Constant | Value | Description |
|---|---|---|
| `GREP_TOOL_NAME` | `'Grep'` | Internal tool identifier |
| `VCS_DIRECTORIES_TO_EXCLUDE` | `['.git', '.svn', '.hg', '.bzr', '.jj', '.sl']` | Version control directories automatically excluded |
| `DEFAULT_HEAD_LIMIT` | `250` | Default result cap when `head_limit` is unspecified |
| `maxResultSizeChars` | `20_000` | Maximum result size for persistence (20K — tool result persistence threshold) |

## Dependencies

### Internal Dependencies

- `ripGrep` (`src/utils/ripgrep.ts`) — Ripgrep subprocess wrapper with timeout handling
- `expandPath`, `toRelativePath` (`src/utils/path.ts`) — Path normalization and relativization
- `checkReadPermissionForTool`, `getFileReadIgnorePatterns`, `normalizePatternsToPath` (`src/utils/permissions/filesystem.ts`) — Permission evaluation and ignore pattern extraction
- `matchWildcardPattern` (`src/utils/permissions/shellRuleMatching.ts`) — Wildcard matching for permission rules
- `getGlobExclusionsForPluginCache` (`src/utils/plugins/orphanedPluginFilter.ts`) — Plugin directory exclusion patterns
- `semanticBoolean`, `semanticNumber` (`src/utils/semanticBoolean.ts`, `src/utils/semanticNumber.ts`) — Zod schema wrappers for LLM-friendly number/boolean handling
- `plural` (`src/utils/stringUtils.ts`) — Pluralization utility for result summaries
- `getCwd` (`src/utils/cwd.ts`) — Current working directory resolution
- `getFsImplementation` (`src/utils/fsOperations.ts`) — Abstracted filesystem operations (for sorting by mtime)
- `suggestPathUnderCwd`, `FILE_NOT_FOUND_CWD_NOTE`, `getDisplayPath` (`src/utils/file.ts`) — Path suggestion and display formatting
- `isENOENT` (`src/utils/errors.ts`) — ENOENT error detection
- `truncate` (`src/utils/format.ts`) — Text truncation for summaries
- `extractTag` (`src/utils/messages.ts`) — XML tag extraction from error messages

### External Dependencies

- `zod/v4` — Input/output schema validation
- `react` — UI component rendering (Ink-based terminal UI with React Compiler)
- `@anthropic-ai/sdk` — Tool result block types

## Implementation Details

### Core Logic

The tool is built using `buildTool()` and follows a multi-stage execution pipeline:

1. **Input validation** (`validateInput`) — I/O-based check that the optional `path` parameter exists. Skips filesystem operations for UNC paths to prevent NTLM credential leaks.
2. **Permission check** (`checkPermissions`) — Evaluates read permissions via `checkReadPermissionForTool()`.
3. **Permission matcher preparation** (`preparePermissionMatcher`) — Uses wildcard pattern matching against the regex pattern for rule evaluation.
4. **Argument construction** (`call`) — Builds ripgrep command-line arguments from the input parameters.
5. **Ripgrep execution** — Invokes `ripGrep()` with the constructed arguments and search path.
6. **Result processing** — Mode-specific processing: relativize paths, apply head limits, parse counts, or sort by mtime.

### Input Schema

```typescript
z.strictObject({
  pattern: z.string(),              // Regex pattern to search for
  path: z.string().optional(),      // File or directory to search in (defaults to cwd)
  glob: z.string().optional(),      // Glob pattern to filter files (e.g., "*.js", "*.{ts,tsx}")
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  '-B': z.number().optional(),      // Lines before each match (rg -B)
  '-A': z.number().optional(),      // Lines after each match (rg -A)
  '-C': z.number().optional(),      // Context lines (rg -C, alias for context)
  context: z.number().optional(),   // Context lines before and after (rg -C)
  '-n': z.boolean().optional(),     // Show line numbers (defaults to true)
  '-i': z.boolean().optional(),     // Case insensitive search (rg -i)
  type: z.string().optional(),      // File type filter (rg --type: js, py, rust, go, etc.)
  head_limit: z.number().optional(),// Limit output to first N results (default 250)
  offset: z.number().optional(),    // Skip first N results before applying head_limit
  multiline: z.boolean().optional(),// Enable multiline mode (rg -U --multiline-dotall)
})
```

### Output Schema (Mode-Dependent)

```typescript
z.object({
  mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  numFiles: z.number(),
  filenames: z.array(z.string()),
  content: z.string().optional(),       // For content and count modes
  numLines: z.number().optional(),      // For content mode
  numMatches: z.number().optional(),    // For count mode
  appliedLimit: z.number().optional(),  // The limit that was actually applied
  appliedOffset: z.number().optional(), // The offset that was applied
})
```

### Key Algorithms

#### Regex Search via Ripgrep

The tool constructs ripgrep command-line arguments from the input parameters:

```typescript
const args = ['--hidden']  // Include hidden files by default

// Exclude VCS directories
for (const dir of VCS_DIRECTORIES_TO_EXCLUDE) {
  args.push('--glob', `!${dir}`)
}

// Limit line length to prevent minified/base64 clutter
args.push('--max-columns', '500')

// Multiline mode (only when explicitly requested)
if (multiline) {
  args.push('-U', '--multiline-dotall')
}

// Case insensitive
if (case_insensitive) {
  args.push('-i')
}

// Output mode
if (output_mode === 'files_with_matches') {
  args.push('-l')  // List files with matches
} else if (output_mode === 'count') {
  args.push('-c')  // Show match counts
}

// Line numbers (only for content mode)
if (show_line_numbers && output_mode === 'content') {
  args.push('-n')
}

// Context lines (-C/context takes precedence over -B/-A)
if (output_mode === 'content') {
  if (context !== undefined) {
    args.push('-C', context.toString())
  } else if (context_c !== undefined) {
    args.push('-C', context_c.toString())
  } else {
    if (context_before !== undefined) args.push('-B', context_before.toString())
    if (context_after !== undefined) args.push('-A', context_after.toString())
  }
}

// Pattern handling (use -e flag if pattern starts with dash)
if (pattern.startsWith('-')) {
  args.push('-e', pattern)
} else {
  args.push(pattern)
}
```

The ripgrep subprocess is invoked via `ripGrep(args, absolutePath, abortController.signal)`, which handles execution, output parsing, and timeout management.

#### File Filtering with Glob Patterns

The `glob` parameter supports comma-separated patterns and brace expansion:

```typescript
// Split on commas and spaces, but preserve patterns with braces
const globPatterns: string[] = []
const rawPatterns = glob.split(/\s+/)

for (const rawPattern of rawPatterns) {
  if (rawPattern.includes('{') && rawPattern.includes('}')) {
    globPatterns.push(rawPattern)  // Don't split brace patterns
  } else {
    globPatterns.push(...rawPattern.split(',').filter(Boolean))
  }
}

for (const globPattern of globPatterns.filter(Boolean)) {
  args.push('--glob', globPattern)
}
```

This correctly handles patterns like `*.{ts,tsx}` (preserved as-is) and `*.js,*.jsx` (split into separate `--glob` flags).

#### Ignore Pattern Integration

The tool automatically applies ignore patterns from the permission context:

```typescript
const ignorePatterns = normalizePatternsToPath(
  getFileReadIgnorePatterns(appState.toolPermissionContext),
  getCwd(),
)
for (const ignorePattern of ignorePatterns) {
  // Absolute paths: !/path/to/ignore
  // Relative paths: !**/path/to/ignore (for ripgrep's relative matching)
  const rgIgnorePattern = ignorePattern.startsWith('/')
    ? `!${ignorePattern}`
    : `!**/${ignorePattern}`
  args.push('--glob', rgIgnorePattern)
}
```

Additionally, orphaned plugin version directories are excluded via `getGlobExclusionsForPluginCache()`.

#### Context Lines

Context line handling supports three specification methods with clear precedence:

1. `context` or `-C` — Symmetric context (N lines before and after)
2. `-B` and `-A` — Asymmetric context (separate before/after counts)

When `context` or `-C` is specified, it takes precedence over `-B`/`-A`. Both `-C` and `context` map to ripgrep's `-C` flag.

#### Output Modes

The tool supports three distinct output modes:

| Mode | Ripgrep Flag | Output | Pagination |
|---|---|---|---|
| `files_with_matches` (default) | `-l` | List of file paths | `head_limit` limits file count |
| `content` | None (default) | Matching lines with context | `head_limit` limits output lines |
| `count` | `-c` | Per-file match counts | `head_limit` limits count entries |

**Content mode** processes ripgrep output lines in the format `/absolute/path:line_content` or `/absolute/path:num:content`, converting absolute paths to relative:

```typescript
const finalLines = limitedResults.map(line => {
  const colonIndex = line.indexOf(':')
  if (colonIndex > 0) {
    const filePath = line.substring(0, colonIndex)
    const rest = line.substring(colonIndex)
    return toRelativePath(filePath) + rest
  }
  return line
})
```

**Count mode** parses ripgrep's `filename:count` format and aggregates totals:

```typescript
let totalMatches = 0
let fileCount = 0
for (const line of finalCountLines) {
  const colonIndex = line.lastIndexOf(':')
  if (colonIndex > 0) {
    const countStr = line.substring(colonIndex + 1)
    const count = parseInt(countStr, 10)
    if (!isNaN(count)) {
      totalMatches += count
      fileCount += 1
    }
  }
}
```

**Files with matches mode** (default) sorts results by modification time:

```typescript
const stats = await Promise.allSettled(
  results.map(_ => getFsImplementation().stat(_)),
)
const sortedMatches = results
  .map((_, i) => {
    const r = stats[i]!
    return [_, r.status === 'fulfilled' ? (r.value.mtimeMs ?? 0) : 0] as const
  })
  .sort((a, b) => {
    if (process.env.NODE_ENV === 'test') {
      return a[0].localeCompare(b[0])  // Deterministic in tests
    }
    const timeComparison = b[1] - a[1]
    if (timeComparison === 0) {
      return a[0].localeCompare(b[0])  // Filename tiebreaker
    }
    return timeComparison
  })
  .map(_ => _[0])
```

`Promise.allSettled` is used so that a single `ENOENT` (file deleted between ripgrep's scan and this stat call) does not reject the entire batch. Failed stats sort as mtime 0.

#### Pagination with Head Limit and Offset

The `applyHeadLimit()` function implements pagination across all output modes:

```typescript
function applyHeadLimit<T>(
  items: T[],
  limit: number | undefined,
  offset: number = 0,
): { items: T[]; appliedLimit: number | undefined } {
  // Explicit 0 = unlimited escape hatch
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT  // 250
  const sliced = items.slice(offset, offset + effectiveLimit)
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

Key design decisions:
- `limit === 0` is the unlimited escape hatch (avoids falsy `0` being treated as "no limit specified")
- `appliedLimit` is only set when truncation actually occurred, so the model knows there may be more results
- `offset` enables pagination: fetch results 250-500 with `offset: 250, head_limit: 250`

The `formatLimitInfo()` helper generates display text for pagination metadata:

```typescript
function formatLimitInfo(
  appliedLimit: number | undefined,
  appliedOffset: number | undefined,
): string {
  const parts: string[] = []
  if (appliedLimit !== undefined) parts.push(`limit: ${appliedLimit}`)
  if (appliedOffset) parts.push(`offset: ${appliedOffset}`)
  return parts.join(', ')
}
```

### Edge Cases

- **Pattern starting with dash**: Uses `-e` flag to prevent ripgrep from interpreting it as a command-line option
- **No results**: Returns mode-appropriate empty messages ("No files found", "No matches found")
- **File deleted between scan and stat**: `Promise.allSettled` handles gracefully; failed stats sort as mtime 0
- **WSL performance**: Ripgrep timeout is handled internally; `RipgrepTimeoutError` propagates up so the agent knows the search didn't complete
- **Minified/base64 content**: `--max-columns 500` limits line length to prevent cluttering output
- **Test determinism**: In `NODE_ENV === 'test'`, sorting uses filename instead of mtime for reproducible results
- **Multiline patterns**: Only enabled when explicitly requested via `multiline: true` to avoid performance degradation on single-line searches

## Data Flow

```
User Request
    │
    ▼
┌─────────────────────┐
│  validateInput()    │  ← Path exists? UNC skip?
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  checkPermissions() │  ← Read permission evaluation
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Build rg args      │  ← Flags from input: mode, context, glob, type, etc.
│  --hidden            │
│  --glob !.git ...    │  ← VCS exclusions
│  --max-columns 500   │  ← Line length limit
│  -l/-c/content       │  ← Output mode
│  -C/-B/-A            │  ← Context lines
│  --glob (user)       │  ← User glob filter
│  --glob (ignore)     │  ← Permission ignore patterns
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  ripGrep()          │  ← Subprocess execution with timeout
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Mode Processing    │
│  ┌───────────────┐  │
│  │ content:      │  │  Apply head_limit → relativize paths → join lines
│  │ count:        │  │  Apply head_limit → relativize → parse totals
│  │ files:        │  │  Stat → sort by mtime → apply head_limit → relativize
│  └───────────────┘  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Result Mapping     │  ← mapToolResultToToolResultBlockParam
│  + Pagination Info  │
└─────────┬───────────┘
          │
          ▼
    Tool Result
```

## Integration Points

### GlobTool UI Sharing

`GlobTool` reuses `GrepTool.renderToolResultMessage` for rendering file list results. The shared `SearchResultSummary` component in `UI.tsx` handles display for both tools.

### Permission System

- `preparePermissionMatcher()` creates a matcher function that compares permission rule patterns against the regex pattern using `matchWildcardPattern()`
- `checkPermissions()` delegates to `checkReadPermissionForTool()`, which evaluates the tool against configured read permission rules
- Ignore patterns from `getFileReadIgnorePatterns()` are automatically converted to ripgrep `--glob` exclusions
- The tool is marked `isReadOnly: true`, reinforcing its non-mutating nature

### Auto-Classifier

`toAutoClassifierInput()` returns `input.pattern` or `input.pattern in input.path`, enabling the auto-classifier to categorize tool invocations by the search pattern and scope.

### Search/Read Classification

`isSearchOrReadCommand()` returns `{ isSearch: true, isRead: false }`, classifying this as a search operation (not a read operation) for activity tracking and analytics.

### Concurrency Safety

`isConcurrencySafe()` returns `true`, allowing parallel execution with other tools. Grep operations are read-only and stateless.

### Plugin System Integration

`getGlobExclusionsForPluginCache()` provides exclusion patterns for orphaned plugin version directories, preventing stale plugin artifacts from appearing in search results.

## Configuration

### Result Limits

| Parameter | Default | Description |
|---|---|---|
| `DEFAULT_HEAD_LIMIT` | `250` | Default result cap when `head_limit` is unspecified |
| `maxResultSizeChars` | `20,000` | Character limit for persisting tool results |
| `--max-columns` | `500` | Maximum line length in ripgrep output |

### Environment Variables

No tool-specific environment variables. Limits are controlled through input parameters (`head_limit`, `offset`).

### Feature Flags

No feature flags directly control GrepTool behavior.

## Error Handling

### Validation Errors (Pre-I/O)

| Error Code | Condition | Message |
|---|---|---|
| 1 | Path does not exist | "Path does not exist: {path}. … Did you mean {suggestion}?" |

### Runtime Errors

| Error | Condition | Handling |
|---|---|---|
| `ENOENT` | Path not found during validation | Suggests similar paths under cwd |
| `RipgrepTimeoutError` | Ripgrep execution timeout (WSL, large codebases) | Propagates up to agent loop — signals incomplete search |
| Abort | `abortController.signal` fired | Propagates cancellation to ripgrep subprocess |

### Error UI Rendering

`renderToolUseErrorMessage()` in `UI.tsx` provides user-friendly error display:
- Detects "file not found" pattern (containing `FILE_NOT_FOUND_CWD_NOTE`) and shows "File not found"
- Detects `<tool_use_error>` tags and shows "Error searching files"
- Falls back to `FallbackToolUseErrorMessage` for other errors

### Result Mapping by Mode

`mapToolResultToToolResultBlockParam()` formats results differently per mode:

**Content mode**: Shows matching lines with optional pagination footer:
```
[matching content lines]

[Showing results with pagination = limit: 250, offset: 0]
```

**Count mode**: Shows count lines with summary footer:
```
[filename:count lines]

Found 42 total occurrences across 8 files. with pagination = limit: 250
```

**Files with matches mode**: Shows file count header and file list:
```
Found 15 files
src/file1.ts
src/file2.ts
...
```

## Performance Optimization

### Ripgrep as Search Engine

Ripgrep is chosen for its performance advantages:
- Rust-based implementation with SIMD optimizations
- Automatic `.gitignore` respect
- Parallel directory traversal
- Memory-mapped file reading
- Smart regex engine with literal optimizations

### Automatic Exclusions

Several exclusion strategies reduce search scope:
- **VCS directories**: `.git`, `.svn`, `.hg`, `.bzr`, `.jj`, `.sl` are always excluded
- **Hidden files**: Included by default (`--hidden`) but subject to `.gitignore` rules
- **Permission ignore patterns**: Extracted from the permission context and applied as `--glob` exclusions
- **Orphaned plugin directories**: Dynamically excluded via `getGlobExclusionsForPluginCache()`

### Line Length Limiting

`--max-columns 500` prevents extremely long lines (minified JavaScript, base64 strings, generated code) from cluttering output and consuming context tokens.

### Head Limit Application Order

In content and count modes, `head_limit` is applied **before** path relativization:

```typescript
// Apply head_limit first — relativize is per-line work, so
// avoid processing lines that will be discarded
const { items: limitedResults, appliedLimit } = applyHeadLimit(
  results, head_limit, offset,
)
```

This avoids unnecessary string operations on lines that will be discarded, which matters for broad patterns returning 10k+ lines.

### Path Relativization

All returned paths are converted from absolute to relative (under `cwd`) via `toRelativePath()`. This reduces token usage in tool results, especially for deeply nested projects.

### Lazy Schema Evaluation

Input and output schemas use `lazySchema()` to defer Zod schema construction until first access, reducing cold-start overhead.

### AbortController Support

The `abortController.signal` is passed to the `ripGrep()` function, enabling cancellation of long-running searches when the user navigates away or the request is superseded.

### Timeout Handling for WSL

WSL has a 3-5x performance penalty for file reads on WSL2. Ripgrep handles timeouts internally via `execFile` timeout, and `RipgrepTimeoutError` propagates up to the agent loop. This is intentional — it signals that the search didn't complete rather than implying no matches were found.

### Sorting Optimization

For `files_with_matches` mode, file stats are fetched in parallel via `Promise.allSettled()`:
- Individual stat failures don't block the entire sort
- Failed stats default to mtime 0 (sort to the end)
- In test mode, filename-based sorting ensures deterministic results

### Default Head Limit Rationale

The default limit of 250 is calibrated against the 20K character tool result persistence threshold (~6-24K tokens for grep-heavy sessions). It's generous enough for exploratory searches while preventing context bloat. Users can pass `head_limit: 0` explicitly for unlimited results.

## Testing

No tool-specific test files are present in the `GrepTool/` directory. Testing is likely covered by integration tests in the broader tool test suite.

## Related Modules

- [GlobTool](./GlobTool.md) — Complementary file pattern matching tool; shares UI rendering
- [BashTool](./BashTool.md) — Used for directory listing; GrepTool is preferred over `rg`/`grep` via Bash
- [AgentTool](./AgentTool.md) — Used for open-ended searches requiring multiple rounds
- [ripgrep utility](../05-utils/ripgrep.md) — Ripgrep subprocess wrapper

## Notes

- The tool name exposed to users is `'Search'` (via `userFacingName()`), while the internal identifier is `'Grep'`
- Users are explicitly instructed to **never** invoke `grep` or `rg` as a Bash command — the GrepTool has optimized permissions and access
- Ripgrep pattern syntax differs from grep: literal braces need escaping (use `interface\{\}` to find `interface{}` in Go code)
- Multiline mode (`multiline: true`) enables cross-line patterns where `.` matches newlines, but should be used sparingly due to performance impact
- The tool is marked as `isConcurrencySafe: true` and `isReadOnly: true`, indicating it can be called in parallel with other tools and does not modify state
- The `strict: true` flag on the tool definition enforces strict input schema validation
- Results in `files_with_matches` mode are sorted by modification time (newest first), with filename as a tiebreaker
