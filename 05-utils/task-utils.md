# Task Utilities

## Purpose

The task utilities module provides the framework for managing background tasks in Claude Code, including task state management, output handling (in-memory and disk-based), progress tracking, SDK event emission, and output formatting. It supports both bash command output (file mode) and hook output (pipe mode).

## Location

- `restored-src/src/utils/task/framework.ts` — Task state management, polling, notifications
- `restored-src/src/utils/task/TaskOutput.ts` — Task output management (in-memory buffer + disk spill)
- `restored-src/src/utils/task/diskOutput.ts` — Async disk write queue for task output files
- `restored-src/src/utils/task/outputFormatting.ts` — Output truncation and formatting
- `restored-src/src/utils/task/sdkProgress.ts` — SDK progress event emission

## Key Exports

### Framework (`framework.ts`)

#### Constants
- `POLL_INTERVAL_MS` (1000): Standard polling interval for all tasks
- `STOPPED_DISPLAY_MS` (3000): Duration to display killed tasks before eviction
- `PANEL_GRACE_MS` (30000): Grace period for terminal local_agent tasks in coordinator panel

#### Types
- `TaskAttachment`: Attachment type for task status updates with taskId, status, description, and delta summary

#### Functions
- `updateTaskState<T>(taskId, setAppState, updater)`: Generic type-safe task state update in AppState
- `registerTask(task, setAppState)`: Register a new task and emit `task_started` SDK event
- `evictTerminalTask(taskId, setAppState)`: Remove terminal tasks (completed/failed/killed) from AppState with panel grace period support
- `getRunningTasks(state)`: Get all running tasks from AppState
- `generateTaskAttachments(state)`: Generate attachments for tasks with new output or status changes, read delta from disk
- `applyTaskOffsetsAndEvictions(setAppState, updatedTaskOffsets, evictedTaskIds)`: Apply output offset patches and evictions against fresh state (avoids TOCTOU races)
- `pollTasks(getAppState, setAppState)`: Main polling loop — generates attachments, applies offsets, sends notifications

### TaskOutput (`TaskOutput.ts`)

#### Class: `TaskOutput`
Single source of truth for a shell command's output with two modes:

**File mode** (bash commands): stdout/stderr go directly to a file via stdio fds. Progress is extracted by polling the file tail.

**Pipe mode** (hooks): Data flows through `writeStdout()`/`writeStderr()` and is buffered in memory, spilling to disk if it exceeds the limit (8MB default).

#### Key Methods
- `constructor(taskId, onProgress, stdoutToFile, maxMemory?)`: Creates output handler, registers for polling
- `static startPolling(taskId)`, `static stopPolling(taskId)`: Start/stop file polling (called from React)
- `writeStdout(data)`, `writeStderr(data)`: Write data (pipe mode only)
- `getStdout()`: Get stdout content (reads from file in file mode, from buffer in pipe mode)
- `getStderr()`: Get stderr content (sync, from buffer only)
- `spillToDisk()`: Force all buffered content to disk
- `flush()`: Wait for all pending disk writes
- `deleteOutputFile()`: Delete the output file
- `clear()`: Clear all state and stop polling

#### Properties
- `isOverflowed`: True when output has spilled to disk
- `totalLines`, `totalBytes`: Output statistics
- `outputFileRedundant`: True after `getStdout()` when file was fully read (can be deleted)
- `outputFileSize`: Total file size in bytes

#### Static Polling
- Shared `#registry` and `#activePolling` maps manage all file-mode instances
- Single `setInterval` ticks all active tasks, reading file tails (4096 bytes) and extrapolating line counts

### Disk Output (`diskOutput.ts`)

#### Constants
- `MAX_TASK_OUTPUT_BYTES` (5GB): Disk cap for task output files
- `MAX_TASK_OUTPUT_BYTES_DISPLAY` (`'5GB'`): Human-readable display string

#### Class: `DiskTaskOutput`
Encapsulates async disk writes using a flat array write queue processed by a single drain loop. Each chunk can be GC'd immediately after its write completes (avoids memory retention from chained `.then()` closures).

#### Key Methods
- `append(content)`: Queue content for writing; auto-flushes
- `flush()`: Wait for all pending writes
- `cancel()`: Clear the write queue

#### Module Functions
- `getTaskOutputDir()`: Get session-specific output directory (project temp + session ID)
- `getTaskOutputPath(taskId)`: Get output file path for a task
- `appendTaskOutput(taskId, content)`: Append content to task's disk file
- `flushTaskOutput(taskId)`: Wait for pending writes
- `evictTaskOutput(taskId)`: Flush and remove from in-memory map (does not delete file)
- `getTaskOutputDelta(taskId, fromOffset, maxBytes?)`: Read new content since last offset
- `getTaskOutput(taskId, maxBytes?)`: Get tail of output file
- `getTaskOutputSize(taskId)`: Get current file size
- `cleanupTaskOutput(taskId)`: Cancel writes and delete output file
- `initTaskOutput(taskId)`: Create empty output file (with O_NOFOLLOW for security)
- `initTaskOutputAsSymlink(taskId, targetPath)`: Create output file as symlink

#### Security
- `O_NOFOLLOW` flag prevents symlink-following attacks from sandboxed processes
- `O_EXCL` ensures new file creation fails if path already exists

### Output Formatting (`outputFormatting.ts`)

#### Constants
- `TASK_MAX_OUTPUT_UPPER_LIMIT` (160,000): Maximum allowed output length
- `TASK_MAX_OUTPUT_DEFAULT` (32,000): Default output length limit

#### Functions
- `getMaxTaskOutputLength()`: Get configured limit from `TASK_MAX_OUTPUT_LENGTH` env var (bounded)
- `formatTaskOutput(output, taskId)`: Truncate output if too large, including header with file path

### SDK Progress (`sdkProgress.ts`)

#### Functions
- `emitTaskProgress(params)`: Emit `task_progress` SDK event with taskId, token usage, tool uses, duration, and optional workflow progress

## Dependencies

- `../CircularBuffer.js` — Circular buffer for recent output lines
- `../fsOperations.js` — File range reading and tailing
- `../sdkEventQueue.js` — SDK event emission
- `../messageQueueManager.js` — Pending notification queue
- `../shell/outputLimits.js` — Output length limits
- `../permissions/filesystem.js` — Project temp directory

## Design Notes

- **Memory-efficient disk writes**: `DiskTaskOutput` uses a flat array queue with `splice(0, length)` to in-place clear the array, informing the GC that string references can be freed immediately. This avoids the memory ballooning problem of chained `.then()` closures.
- **TOCTOU safety**: `applyTaskOffsetsAndEvictions` merges patches against fresh `prev.tasks` (not stale pre-await snapshot), so concurrent status transitions aren't clobbered.
- **Session isolation**: Output directories include the session ID so concurrent sessions don't clobber each other's files. The session ID is captured at first call, not re-read, so background tasks survive `/clear`.
- **Shared polling**: All file-mode `TaskOutput` instances share a single `setInterval` ticker, reducing overhead when multiple tasks are running.
- **Two eviction paths**: Terminal tasks are evicted eagerly via `evictTerminalTask` (when notified) and lazily via `generateTaskAttachments` (safety net).
