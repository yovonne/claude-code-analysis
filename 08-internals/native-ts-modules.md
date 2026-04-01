# Native-TS Modules

## Purpose

Pure TypeScript ports of native Rust/NAPI modules. These modules reimplement the same API surface as their native counterparts, allowing the application to run without native dependencies while maintaining API compatibility.

## Location

`restored-src/src/native-ts/`

## Modules

### `color-diff/` — Syntax-Highlighted Diff Rendering

**Purpose:** Renders colored diff hunks and source files with syntax highlighting for terminal display.

**Original:** `vendor/color-diff-src` (Rust with syntect + bat + similar crate)
**Port:** Uses `highlight.js` (already a dependency via cli-highlight) and the `diff` npm package

**Key exports:**
- `ColorDiff`: Renders a unified diff hunk with syntax highlighting, word-level diffs, line numbers, and ANSI color codes
- `ColorFile`: Renders an entire source file with syntax highlighting and line numbers
- `getSyntaxTheme(themeName)`: Returns the syntax theme name for a given Claude theme
- `getNativeModule()`: Returns the module API for compatibility with native callers

**Color system:**
- Supports `truecolor`, `color256`, and `ansi` color modes
- `ansi256FromRgb()`: Port of `ansi_colours::ansi256_from_rgb` — approximates RGB to xterm-256 palette using 6x6x6 cube + 24 greys
- Three pre-measured theme palettes: Monokai Extended (dark), GitHub (light), ANSI

**Syntax highlighting:**
- Lazy-loads `highlight.js` to avoid ~50MB heap cost at startup
- Detects language by filename, extension, and shebang
- Handles both hljs 10.x (`kind`) and 11.x (`scope`) node formats
- Storage keyword re-splitting: keywords like `const`, `function`, `class` get cyan storage color instead of pink keyword color

**Word diff algorithm:**
- Tokenizes into word runs, whitespace runs, and punctuation
- Uses `diffArrays` from the `diff` package
- CHANGE_THRESHOLD (0.4): Skips word diff when >40% of text changed (too noisy)
- Finds adjacent +/- line pairs and computes character-level change ranges

**Transform pipeline:**
1. Parse markers (+, -, ) and line numbers
2. Compute word-diff ranges for adjacent changed lines
3. Highlight each line with syntax colors
4. Apply background colors (line + word-level)
5. Wrap text to terminal width
6. Add line numbers and markers
7. Convert to ANSI escape sequences

### `file-index/` — Fuzzy File Search

**Purpose:** High-performance fuzzy file searching, ported from the nucleo library.

**Original:** `vendor/file-index-src` (Rust wrapping nucleo)
**Port:** Pure TypeScript with bitmap-based filtering and top-k selection

**Key exports:**
- `FileIndex` class:
  - `loadFromFileList(fileList)`: Synchronously build index from path array
  - `loadFromFileListAsync(fileList)`: Async build with progressive querying
  - `search(query, limit)`: Fuzzy search returning top N results

**Search algorithm:**
1. **Bitmap rejection**: Each path has a 26-bit bitmap of letters present. O(1) reject if path doesn't contain all needle letters (~89% survival for "test", 90%+ rejection for rare chars)
2. **Fused indexOf scan**: Find positions using `String.indexOf` (SIMD-accelerated in V8/JSC) and accumulate gap/consecutive terms inline
3. **Boundary/camelCase scoring**: Bonus for matches at word boundaries (`/`, `-`, `_`, `.`) and camelCase transitions
4. **Top-k selection**: Maintains sorted array of best `limit` matches, avoiding O(n log n) full sort
5. **Score normalization**: Final score = position_in_results / result_count (lower = better)
6. **Test penalty**: Paths containing "test" get 1.05x penalty (capped at 1.0)

**Scoring constants (nucleo-style):**
- `SCORE_MATCH = 16`, `BONUS_BOUNDARY = 8`, `BONUS_CAMEL = 6`, `BONUS_CONSECUTIVE = 4`, `BONUS_FIRST_CHAR = 8`
- `PENALTY_GAP_START = 3`, `PENALTY_GAP_EXTENSION = 1`

**Async building:**
- Yields to event loop every ~4ms of sync work
- `queryable` promise resolves after first chunk (~5-10ms for 270k paths)
- `done` promise resolves when entire index is built
- `readyCount` tracks how many paths are indexed; `search()` searches the ready prefix while build continues

**Top-level cache:**
- Extracts unique top-level path segments (first 100)
- Sorted by length then alphabetically
- Returned for empty queries

### `yoga-layout/` — Flexbox Layout Engine

**Purpose:** Pure TypeScript port of Meta's Yoga layout engine for terminal UI layout.

**Original:** Yoga C++ (~2500 lines in CalculateLayout.cpp alone)
**Port:** Simplified single-pass flexbox implementation covering the subset Ink actually uses

**Key exports:**
- `Node` class: Yoga node with full style/layout API
- `Config`: Layout configuration
- Enums: `Align`, `FlexDirection`, `Justify`, `Wrap`, `Overflow`, `PositionType`, `Display`, `Edge`, `Gutter`, `MeasureMode`, `Unit`, `Direction`, `BoxSizing`, `Dimension`, `Errata`, `ExperimentalFeature`

**Implemented features:**
- `flex-direction` (row/column + reverse)
- `flex-grow` / `flex-shrink` / `flex-basis`
- `align-items` / `align-self` (stretch, flex-start, center, flex-end)
- `justify-content` (all six values)
- `margin` / `padding` / `border` / `gap`
- `width` / `height` / `min` / `max` (point, percent, auto)
- `position: relative / absolute`
- `display: flex / none / contents`
- Measure functions (for text nodes)
- `flex-wrap: wrap / wrap-reverse` (multi-line)
- `align-content` (positions wrapped lines)
- `margin: auto` (main + cross axis)
- Multi-pass flex clamping for min/max constraints
- Baseline alignment

**Not implemented:**
- `aspect-ratio`
- `box-sizing: content-box`
- RTL direction (Ink always passes LTR)

**Performance optimizations:**
1. **Dirty-flag caching**: Skip clean subtrees when inputs match cached values
2. **Multi-entry layout cache**: 4-slot LRU cache stores (inputs → computed w,h) — reduced dirty-leaf relayout from 76k to 4k layoutNode calls (6.86ms → 550µs for 500-message scrollbox)
3. **Basis cache**: Cached `computeFlexBasis` result with generation tracking — 105k visits → ~10k for 1593-node tree
4. **Fast-path flags**: `_hasAutoMargin`, `_hasPosition`, `_hasPadding`, `_hasBorder`, `_hasMargin` skip expensive edge resolution for common cases
5. **`resolveEdges4Into`**: Resolves all 4 physical edges in one pass, hoisting shared fallback lookups — was the #1 hotspot per CPU profile
6. **Generation-based cache validity**: `_cGen` / `_fbGen` stamp entries with the current `calculateLayout` generation, allowing cross-generation hits for clean nodes

**Layout algorithm (Yoga STEP 1-9):**
1. Compute flex-basis for each child, break into lines
2. Resolve flexible lengths (grow/shrink)
3. Measure cross sizes
4. Determine container dimensions
5. Position children (main axis, then cross axis)
6. Handle absolute positioning
7. Apply baseline alignment
8. Handle overflow
9. Round to pixel grid

## Dependencies

### Internal Dependencies

- `ink/stringWidth.js` — String width calculation for text wrapping (color-diff)
- `utils/log.js` — Error logging

### External Dependencies

- `diff` — Array diffing for word-level diffs (color-diff)
- `highlight.js` — Syntax highlighting (color-diff, lazy-loaded)
- `path` — `basename`, `extname` for language detection (color-diff)

## Related Modules

- [UI Components](../06-ui/ink-framework.md) — Yoga layout powers Ink component layout
- [Vendor Modules](./vendor-modules.md) — Original native implementations these ports replace
