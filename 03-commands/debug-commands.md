# Debug Commands

## Purpose

Documents the CLI commands used for debugging, profiling, and internal diagnostics. Most of these commands are hidden from normal users and intended for development or support purposes.

---

## /debug-tool-call

### Purpose
Placeholder for debug tool call functionality.

### Location
`restored-src/src/commands/debug-tool-call/index.js`

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
The debug tool call functionality is not implemented in this version of the codebase.

---

## /heapdump (Heap Dump)

### Purpose
Dumps the JavaScript heap to the user's Desktop for memory profiling and leak detection.

### Location
`restored-src/src/commands/heapdump/index.ts`
`restored-src/src/commands/heapdump/heapdump.ts`

### Command Definition
| Property | Value |
|----------|-------|
| Name | `heapdump` |
| Type | `local` |
| Description | Dump the JS heap to ~/Desktop |
| Non-interactive | `true` |
| Hidden | `true` |

### Key Exports

#### Functions
- `call()`: Triggers heap dump and returns file paths

### Dependencies

#### Internal
- `../../utils/heapDumpService.js` — `performHeapDump` for heap dump execution
- `../../commands.js` — `Command` type

### Implementation Details

#### Core Logic
1. Calls `performHeapDump()` which creates:
   - A `.heapsnapshot` file (V8 heap snapshot for Chrome DevTools)
   - A diagnostics file
2. Returns file paths on success
3. Returns error message on failure

#### Edge Cases
- Heap dump failure: Returns descriptive error message
- Files are written to `~/Desktop` for easy access

### Data Flow
```
User runs /heapdump
  → performHeapDump()
  │
  ├─ Success → return { type: 'text', value: '<heap-path>\n<diag-path>' }
  └─ Failure → return { type: 'text', value: 'Failed to create heap dump: <error>' }
```

---

## /ant-trace

### Purpose
Placeholder for Ant tracing functionality.

### Location
`restored-src/src/commands/ant-trace/index.js`

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
The Ant tracing functionality is not implemented in this version of the codebase.

---

## /ctx_viz (Context Visualization Debug)

### Purpose
Placeholder for context visualization debugging.

### Location
`restored-src/src/commands/ctx_viz/index.js`

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
Note: The interactive `/context` command (documented in interaction-commands.md) provides context visualization for end users. This stub is a separate debug tool.

---

## /perf-issue (Performance Issue Debug)

### Purpose
Placeholder for performance issue debugging.

### Location
`restored-src/src/commands/perf-issue/index.js`

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
The performance issue debugging functionality is not implemented in this version of the codebase.

---

## /bughunter (Bug Hunting Debug)

### Purpose
Placeholder for bug hunting functionality.

### Location
`restored-src/src/commands/bughunter/index.js`

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
The bug hunting functionality is not implemented in this version of the codebase.

---

## Debug Commands Overview

Of the six debug commands, only `/heapdump` is implemented. The remaining five are reserved stubs:

| Command | Status | Purpose |
|---------|--------|---------|
| `/heapdump` | Implemented | JS heap snapshot for memory profiling |
| `/debug-tool-call` | Stub | Reserved for tool call debugging |
| `/ant-trace` | Stub | Reserved for Ant tracing |
| `/ctx_viz` | Stub | Reserved for context visualization debug |
| `/perf-issue` | Stub | Reserved for performance issue debugging |
| `/bughunter` | Stub | Reserved for bug hunting |

All stub commands follow the same pattern:
```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

This ensures they don't appear in help output, aren't callable, and won't interfere with the command system while reserving the command names for future implementation.
