# Ink Framework

## Purpose

The Ink framework is a custom React-based terminal UI rendering engine that powers Claude Code's fullscreen alternate-screen interface. It provides a React reconciler for terminal output, a Yoga-based flexbox layout system, an event dispatcher for keyboard/mouse input, and a double-buffered screen diff engine for flicker-free rendering.

## Location

`restored-src/src/ink/` — the entire `ink/` directory plus the re-export barrel at `restored-src/src/ink.ts`.

## Key Exports (from `ink.ts`)

### Render API

| Export | Description |
|--------|-------------|
| `render(node, options?)` | Async render function that wraps the root in `ThemeProvider` and returns an `Instance` |
| `createRoot(options?)` | Creates a reusable Ink root (like `react-dom`'s `createRoot`) for sequential screens |
| `Root` | Managed root type with `render()`, `unmount()`, `waitUntilExit()` |
| `Instance` | Render instance with `rerender()`, `unmount()`, `waitUntilExit()`, `cleanup()` |
| `RenderOptions` | Configuration: `stdout`, `stdin`, `stderr`, `exitOnCtrlC`, `patchConsole`, `onFrame` |

### Core Components

| Export | Description |
|--------|-------------|
| `Box` | Flexbox layout container (maps to `ink-box`) |
| `Text` | Styled text component (maps to `ink-text`) |
| `BaseBox` / `BaseText` | Unthemed base variants |
| `Button` | Interactive button with focus/hover/active state render prop |
| `Link` | Hyperlink component with OSC 8 support and fallback |
| `Spacer` | Flexible space (`flexGrow={1}`) |
| `Newline` | Inserts newline characters |
| `Ansi` | Parses ANSI escape codes into styled React elements |
| `RawAnsi` | Bypasses the ANSI parse roundtrip for pre-rendered output |
| `NoSelect` | Marks content as non-selectable in fullscreen text selection |
| `ScrollBox` | Overflow-scroll container with imperative scroll API |
| `AlternateScreen` | Enters terminal alternate screen buffer (DEC 1049) |

### Hooks

| Export | Description |
|--------|-------------|
| `useInput(handler, options?)` | Captures keyboard input; enables raw mode |
| `useApp()` | Returns `exit()` to unmount the app |
| `useStdin()` | Accesses stdin stream and raw mode controls |
| `useAnimationFrame(intervalMs?)` | Shared clock animation hook; pauses when offscreen |
| `useInterval(callback, intervalMs)` | Interval hook backed by the shared Clock |
| `useAnimationTimer(intervalMs)` | Returns clock time for pure time-based computations |
| `useSelection()` | Fullscreen text selection operations (copy, clear, shift, capture) |
| `useHasSelection()` | Reactive boolean for selection-exists state |
| `useTerminalFocus()` | Whether the terminal window has focus (DECSET 1004) |
| `useTerminalTitle(title)` | Declaratively sets terminal tab/window title (OSC 0) |
| `useTerminalViewport()` | Detects if an element is within the visible viewport |
| `useTabStatus(kind)` | Sets tab-status indicator (OSC 21337) |
| `measureElement(node)` | Returns `{width, height}` of a Box element |

### Events & Focus

| Export | Description |
|--------|-------------|
| `FocusManager` | DOM-like focus manager with tab cycling and focus stack |
| `EventEmitter` | Event system for input, focus, and terminal events |
| `InputEvent` | Keyboard input event with parsed key info |
| `ClickEvent` | Mouse click event (alt-screen only) |
| `TerminalFocusEvent` | Terminal window focus/blur event |
| `Key` | Key object with modifier flags (ctrl, shift, meta, super) |

### Theme System

| Export | Description |
|--------|-------------|
| `ThemeProvider` | Provides theme context with auto/system theme detection |
| `useTheme()` | Returns `[currentTheme, setThemeSetting]` |
| `useThemeSetting()` | Returns raw stored setting (may be `'auto'`) |
| `usePreviewTheme()` | Returns `{setPreviewTheme, savePreview, cancelPreview}` |
| `color(c, theme, type?)` | Curried theme-aware color function |

## Dependencies

### Internal Dependencies

- `src/utils/` — debug logging, config, system theme detection, env utilities
- `src/native-ts/yoga-layout/` — WASM-based Yoga layout engine
- `src/bootstrap/state.js` — scroll activity marking for background interval suppression

### External Dependencies

- `react` / `react-reconciler` — React 19 with custom host config
- `wrap-ansi` — ANSI-aware text wrapping (with Bun native fallback)
- `strip-ansi` — ANSI escape sequence removal
- `@alcalzone/ansi-tokenize` — ANSI tokenization for style diffing
- `usehooks-ts` — `useEventCallback` for stable handler references

## Implementation Details

### Architecture Overview

The Ink framework follows a layered architecture:

```
React Components (Box, Text, Button, etc.)
    ↓
React Reconciler (custom host config in reconciler.ts)
    ↓
DOM Tree (DOMElement, TextNode — virtual DOM for terminal)
    ↓
Yoga Layout Engine (WASM flexbox layout)
    ↓
Renderer (renderer.ts — DOM → screen buffer)
    ↓
Output (output.ts — screen buffer operations)
    ↓
Screen Buffer (screen.ts — pooled cell arrays)
    ↓
LogUpdate (log-update.ts — diff & patch generation)
    ↓
Terminal (terminal.ts — ANSI sequence emission)
```

### React Reconciler (`reconciler.ts`)

Implements a custom React reconciler host config mapping React elements to terminal DOM nodes:

- **Node types**: `ink-root`, `ink-box`, `ink-text`, `ink-virtual-text`, `ink-link`, `ink-progress`, `ink-raw-ansi`
- **Style application**: React `style` props are translated to Yoga layout properties via `applyStyles()`
- **Text handling**: Text nodes must be inside `<Text>` components; bare text throws
- **Dirty tracking**: Nodes are marked dirty on prop/style changes; ancestors propagate dirty upward
- **Yoga cleanup**: Freed nodes have their WASM references cleared before `freeRecursive()` to prevent dangling pointers
- **Event handlers**: Stored in `_eventHandlers` (separate from attributes) so handler identity changes don't defeat the blit optimization
- **Debug repaints**: When `CLAUDE_CODE_DEBUG_REPAINTS` is set, captures React component owner chains for flicker attribution

### DOM Model (`dom.ts`)

Virtual DOM nodes that the reconciler manipulates:

- **`DOMElement`**: Container nodes with `childNodes`, `attributes`, `style`, `yogaNode`, scroll state, focus manager, and event handlers
- **`TextNode`**: Leaf text nodes with `nodeValue`
- **Yoga integration**: Most elements get a Yoga layout node; `ink-virtual-text`, `ink-link`, and `ink-progress` are excluded
- **Measure functions**: `ink-text` nodes get a measure function that performs text wrapping; `ink-raw-ansi` nodes use constant-time measure (width × height attributes)
- **Scroll state**: `scrollTop`, `pendingScrollDelta`, `scrollAnchor`, `stickyScroll`, `scrollClampMin/Max` for virtual scrolling
- **`markDirty()`**: Walks ancestor chain marking elements dirty and yoga dirty for text remeasurement
- **`scheduleRenderFrom()`**: Walks to root and calls `onRender()` for DOM-level mutations that bypass React

### Layout Engine (`layout/`)

- **`engine.ts`**: Thin wrapper over WASM Yoga (`createYogaLayoutNode()`)
- **`node.ts`**: Yoga type definitions (LayoutAlign, LayoutDisplay, LayoutFlexDirection, etc.)
- **`yoga.ts`**: WASM Yoga bindings and layout node creation
- **`geometry.ts`**: Point, Rectangle, Size types and utility functions (clamp, unionRect)

### Rendering Pipeline (`renderer.ts`, `output.ts`)

**Double-buffered rendering:**
1. **Front frame**: Currently displayed screen buffer
2. **Back frame**: Being constructed for next frame
3. **Renderer**: Walks DOM tree → Yoga layout → fills Output operations → produces Screen buffer
4. **LogUpdate**: Diffs new screen against previous frame → generates Patch array → optimizes → writes to terminal

**Screen buffer (`screen.ts`):**
- **CharPool**: Interned character strings with ASCII fast-path (Int32Array direct lookup)
- **StylePool**: Interned style objects with cached SGR transitions
- **HyperlinkPool**: Interned hyperlink strings
- **Cell model**: Packed integer arrays for character, style, hyperlink, width, and noSelect flags
- **Generational reset**: Pools reset every 5 minutes to prevent unbounded growth

**Output operations:**
- `write`: Text at position with soft-wrap flags
- `blit`: Copy region from previous frame (O(unchanged) fast path)
- `clear`: Clear region
- `clip`/`unclip`: Clip output to sub-region
- `noSelect`: Mark cells as non-selectable
- `shift`: Row shift for scroll optimization

### Frame System (`frame.ts`)

- **`Frame`**: Contains `screen`, `viewport`, `cursor`, `scrollHint`, `scrollDrainPending`
- **`FrameEvent`**: Timing data per frame — `durationMs`, phase breakdown (renderer, diff, optimize, write, yoga, commit), flicker reasons
- **`Patch`**: Terminal update operations — stdout content, clear, cursor, style, hyperlink
- **`shouldClearScreen()`**: Determines when to full-reset (resize, overflow)

### Focus Manager (`focus.ts`)

DOM-like focus system:
- **`activeElement`**: Currently focused element
- **Focus stack**: Max 32 entries; previous elements pushed on focus change
- **Tab cycling**: `focusNext()` / `focusPrevious()` walks tabbable elements (tabIndex >= 0)
- **Node removal**: Automatically restores focus from stack when focused element is removed
- **Click focus**: Only focuses elements with numeric `tabIndex`
- **`getFocusManager(node)`**: Walks up to root (like `node.ownerDocument`)

### Event System (`events/`)

- **`EventEmitter`**: Central event bus with priority-based listener ordering
- **`Dispatcher`**: Bridges reconciler priority system with event handling
- **`InputEvent`**: Keyboard input with parsed `Key` (modifiers, special keys)
- **`ClickEvent`**: Mouse click with position (alt-screen only)
- **`FocusEvent`**: Focus/blur with relatedTarget
- **`TerminalFocusEvent`**: Terminal window focus (DECSET 1004)
- **`keyboard-event.ts`**: Keyboard event with capture/bubble phases

### ANSI Parser (`termio/`)

Semantic ANSI escape sequence parser:
- **`Parser`**: Streaming parser that feeds input incrementally and produces structured actions
- **Action types**: Text segments, cursor operations, erase, scroll, title, link, mode changes
- **Style tracking**: Maintains TextStyle state across parse calls (fg, bg, bold, italic, underline, etc.)
- **Sequence support**: SGR (colors/styles), CSI (cursor/erase), OSC (title/hyperlinks/tab-status), ESC
- **`termio.ts`**: Barrel re-export of parser, types, and utility functions

### Terminal Notifications (`useTerminalNotification.ts`)

Cross-terminal notification support:
- **iTerm2**: Growl-style notifications via OSC
- **Kitty**: Multi-part notifications with ID
- **Ghostty**: Native notification protocol
- **Bell**: Raw BEL for tmux compatibility
- **Progress**: OSC 9;4 progress reporting (ConEmu, Ghostty 1.2+, iTerm2 3.6.6+)

### Text Wrapping (`wrap-text.ts`, `wrapAnsi.ts`)

- **`wrap`**: Word-wrap with soft breaks
- **`wrap-trim`**: Word-wrap with leading/trailing whitespace trimmed
- **`truncate-end`**: Ellipsis at end
- **`truncate-middle`**: Ellipsis in middle
- **`truncate-start`**: Ellipsis at start
- Uses `wrap-ansi` npm package with Bun native fallback (`Bun.wrapAnsi`)

### Selection System (`selection.ts`)

Fullscreen text selection (918 lines):
- **`SelectionState`**: Anchor/focus points, dragging flag, anchorSpan for word/line mode, scrolled-off accumulators, virtual rows for clamp tracking
- **Word detection**: Unicode-aware character classes matching iTerm2 defaults
- **Line selection**: Full row selection
- **Drag-to-scroll**: Accumulates scrolled-off rows above/below with soft-wrap bits
- **Keyboard extension**: Shift+arrow moves focus with anchor fixed
- **Keyboard scroll**: PgUp/PgDn/Home/End shift both anchor and focus with virtual row tracking
- **Text extraction**: Respects noSelect cells, soft-wrap joins, trailing whitespace trimming
- **Selection overlay**: Applies solid background color to selected cells (replaces old SGR-7 inverse)

### Ink Instance (`ink.tsx`)

The main `Ink` class orchestrates the entire rendering lifecycle:
- **Throttled rendering**: `scheduleRender()` coalesces rapid updates
- **Terminal resize**: Handles SIGWINCH, recalculates layout
- **Alt-screen management**: Enters/exits alternate screen buffer, manages cursor
- **Console patching**: Intercepts `console.log`/`console.error` to prevent mixing with Ink output
- **Signal handling**: SIGINT (Ctrl+C), SIGTERM, SIGHUP cleanup
- **External editor pause**: Pauses Ink rendering when external editor opens
- **Frame timing**: Optional `onFrame` callback with phase breakdown
- **Double-buffering**: Front/back frame swap with blit optimization

## Data Flow

```
User Input (stdin)
    ↓
Ink Instance (parse keypress, update selection)
    ↓
EventEmitter → useInput / useKeybinding handlers
    ↓
React State Update
    ↓
Reconciler (diff React tree → DOM mutations)
    ↓
resetAfterCommit → onComputeLayout (Yoga)
    ↓
onRender → scheduleRender (throttled)
    ↓
Renderer (DOM → Yoga → Output operations → Screen)
    ↓
LogUpdate (diff screens → Patches → optimize → write)
    ↓
Terminal (ANSI sequences → stdout)
```

## Key Design Decisions

1. **Double-buffered rendering**: Front/back frame swap with blit optimization for O(unchanged) steady-state frames
2. **Shared pools**: CharPool, StylePool, HyperlinkPool intern values across frames to minimize allocations
3. **Microtask defer**: `queueMicrotask()` coalesces rapid scroll mutations into single renders
4. **Yoga WASM**: Layout computed by native WASM module for performance
5. **React Compiler**: Components use `react/compiler-runtime` for automatic memoization
6. **Generational pool reset**: Pools reset periodically to prevent unbounded memory growth
7. **Virtual scroll**: ScrollBox uses clamp bounds to prevent blank screens during burst scroll
8. **Selection as overlay**: Selection highlight is applied directly to screen buffer cells, not as a separate render pass
