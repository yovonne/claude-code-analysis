# NotebookEditTool

## Purpose

NotebookEditTool provides cell-level editing of Jupyter notebook files (.ipynb). It supports three edit modes — replace, insert, and delete — allowing the model to modify notebook cells while preserving the notebook structure, metadata, and execution state. The tool enforces read-before-edit safety and file modification tracking to prevent silent data loss.

## Location

- `restored-src/src/tools/NotebookEditTool/NotebookEditTool.ts` — Main tool definition (491 lines)
- `restored-src/src/tools/NotebookEditTool/prompt.ts` — Tool prompt and description
- `restored-src/src/tools/NotebookEditTool/constants.ts` — Tool name constant
- `restored-src/src/tools/NotebookEditTool/UI.tsx` — UI rendering for tool use/result/error messages

## Key Exports

| Export | Description |
|--------|-------------|
| `NotebookEditTool` | The complete tool definition built via `buildTool()` |
| `inputSchema` | Zod schema for notebook editing input |
| `outputSchema` | Zod schema for edit result output |
| `Output` | Zod-inferred output type |

## Input/Output Schemas

### Input Schema

```typescript
{
  notebook_path: string,    // Required: absolute path to .ipynb file
  cell_id?: string,         // Optional: cell ID to edit (required for replace/delete)
  new_source: string,       // Required: new source content for the cell
  cell_type?: 'code' | 'markdown',  // Optional: cell type (required for insert)
  edit_mode?: 'replace' | 'insert' | 'delete',  // Default: 'replace'
}
```

### Output Schema

```typescript
{
  new_source: string,           // The new source written to the cell
  cell_id?: string,             // ID of the edited cell
  cell_type: 'code' | 'markdown',  // The cell type
  language: string,             // Programming language (from notebook metadata)
  edit_mode: string,            // The edit mode used
  error?: string,               // Error message if operation failed
  notebook_path: string,        // Path to the notebook file (for attribution)
  original_file: string,        // Original content before modification
  updated_file: string,         // Updated content after modification
}
```

## Edit Modes

### Replace (default)

Replaces the contents of an existing cell:

```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-abc123",
  "new_source": "print('hello')",
  "edit_mode": "replace"
}
```

- `cell_id` is required
- `cell_type` is optional (defaults to current cell type)
- For code cells: resets `execution_count` and clears `outputs`
- If `cell_id` refers to one past the end, automatically converts to insert

### Insert

Inserts a new cell after the specified cell:

```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-abc123",
  "new_source": "## New Section",
  "cell_type": "markdown",
  "edit_mode": "insert"
}
```

- `cell_type` is required
- `cell_id` is optional (defaults to inserting at beginning)
- New cell is inserted AFTER the specified cell
- Generates random cell ID for nbformat 4.5+

### Delete

Removes a cell from the notebook:

```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-abc123",
  "edit_mode": "delete"
}
```

- `cell_id` is required
- `new_source` is still required by schema but ignored

## Validation Pipeline

### File Validation

| Check | Error Code | Message |
|-------|-----------|---------|
| File must be .ipynb | 2 | "File must be a Jupyter notebook (.ipynb file)" |
| File not found | 1 | "Notebook file does not exist" |
| Invalid JSON | 6 | "Notebook is not valid JSON" |
| File not read yet | 9 | "File has not been read yet. Read it first before writing to it." |
| File modified since read | 10 | "File has been modified since read..." |
| UNC path | — | Skip filesystem validation (security) |

### Edit Mode Validation

| Check | Error Code | Message |
|-------|-----------|---------|
| Invalid edit mode | 4 | "Edit mode must be replace, insert, or delete" |
| Insert without cell_type | 5 | "Cell type is required when using edit_mode=insert" |

### Cell ID Validation

| Check | Error Code | Message |
|-------|-----------|---------|
| Cell ID not found (replace/delete) | 7 | "Cell with index N does not exist" or "Cell with ID not found" |
| Cell ID not found (insert) | — | Defaults to inserting at beginning |

### Cell ID Resolution

The tool supports two cell ID formats:
1. **Actual cell ID**: The notebook's internal `cell.id` field (e.g., `"cell-abc123"`)
2. **Numeric index**: `cell-N` format parsed via `parseCellId()` (e.g., `"cell-0"`)

The tool first tries to find by actual ID, then falls back to numeric index parsing.

## Execution Pipeline

```
1. INPUT RECEIVED
   - Model returns NotebookEdit tool_use with { notebook_path, cell_id?, new_source, cell_type?, edit_mode? }

2. PATH RESOLUTION
   - Resolve to absolute path (relative paths resolved against cwd)
   - Skip UNC paths for security (NTLM credential leak prevention)

3. VALIDATION
   - Check file extension (.ipynb)
   - Check edit mode validity
   - Check cell_type required for insert
   - Verify file was previously read (read-before-edit safety)
   - Verify file not modified externally since read
   - Parse and validate notebook JSON
   - Verify cell_id exists (for replace/delete)

4. PERMISSION CHECK
   - checkWritePermissionForTool() — filesystem write permission

5. EDIT EXECUTION
   - Read file content and encoding (readFileSyncWithMetadata)
   - Parse notebook JSON (non-memoized jsonParse to avoid cache mutation)
   - Locate target cell by ID or index
   - Apply edit:
     - replace: update cell source, reset execution_count/outputs for code cells
     - insert: create new cell with random ID (nbformat 4.5+), splice into cells array
     - delete: splice cell out of cells array
   - Write updated notebook to disk (writeTextContent, preserving encoding/line endings)

6. STATE UPDATE
   - Update readFileState with post-write mtime (prevents stale Read dedup)
   - Track file history if enabled

7. RETURN
   - Return { new_source, cell_type, language, edit_mode, cell_id, ... }
```

## Read-Before-Edit Safety

The tool enforces that the notebook must have been read before editing:

```typescript
const readTimestamp = toolUseContext.readFileState.get(fullPath)
if (!readTimestamp) {
  return { result: false, message: 'File has not been read yet...', errorCode: 9 }
}
```

This prevents the model from editing a notebook it never saw, or editing against a stale view after an external change — which would cause silent data loss.

## File History Tracking

When file history is enabled (`fileHistoryEnabled()`):

```typescript
await fileHistoryTrackEdit(updateFileHistoryState, fullPath, parentMessage.uuid)
```

This tracks the edit for undo functionality, linking it to the parent message UUID.

## Post-Write State Management

After writing, the tool updates `readFileState` with the new file content and modification timestamp:

```typescript
readFileState.set(fullPath, {
  content: updatedContent,
  timestamp: getFileModificationTime(fullPath),
  offset: undefined,
  limit: undefined,
})
```

This is critical: without this update, a Read→NotebookEdit→Read sequence in the same millisecond would return a stale `file_unchanged` stub against the in-context content.

## Notebook Format Support

- Supports Jupyter notebook format nbformat 4+
- Cell IDs generated for nbformat 4.5+ (using `Math.random().toString(36)`)
- Language detected from `notebook.metadata.language_info.name` (defaults to `'python'`)
- Output written with 1-space indentation (`IPYNB_INDENT = 1`)
- Preserves original file encoding and line endings

## Error Handling

Errors during editing return structured error output rather than throwing:

```typescript
{
  new_source: "...",
  cell_type: "code",
  language: "python",
  edit_mode: "replace",
  error: "Error message",
  cell_id: "...",
  notebook_path: "/path/to/notebook.ipynb",
  original_file: "",
  updated_file: "",
}
```

This allows the model to receive error feedback without interrupting the conversation flow.

## Dependencies

| Module | Purpose |
|--------|---------|
| `utils/fileRead.ts` | readFileSyncWithMetadata — single-pass read with encoding/line endings |
| `utils/file.ts` | writeTextContent, getFileModificationTime |
| `utils/notebook.ts` | parseCellId — numeric cell ID parsing |
| `utils/json.ts` | safeParseJSON — JSON parsing |
| `utils/permissions/filesystem.ts` | checkWritePermissionForTool |
| `utils/cwd.ts` | getCwd — current working directory |
| `utils/errors.ts` | isENOENT — file-not-found detection |
| `utils/fileHistory.ts` | File history tracking for undo |
| `utils/slowOperations.ts` | jsonParse (non-memoized), jsonStringify |
| `services/analytics/` | Event logging |
