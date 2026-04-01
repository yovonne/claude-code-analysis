# Vendor Modules

## Purpose

The vendor directory would contain native Rust/NAPI modules that provide high-performance implementations of core functionality. In this source map reconstruction, the vendor directory does not exist as a source directory — instead, its functionality has been replaced by pure TypeScript ports in `native-ts/`.

## Location

`restored-src/src/vendor/` — **Does not exist in reconstructed source**

## Original Native Modules (Replaced by native-ts Ports)

The following native modules existed in the original codebase and have been ported to TypeScript:

### color-diff-src (Rust)

**Original:** Rust module using `syntect` (syntax highlighting), `bat` (terminal output), and `similar` crate (word diffing)

**Replaced by:** `native-ts/color-diff/`
- Uses `highlight.js` instead of syntect
- Uses `diff` npm package instead of similar crate
- API matches original exactly so callers don't change

**Known differences:**
- highlight.js has grammar gaps: plain identifiers and operators (`=`, `:`) aren't scoped, rendering in default fg instead of white/pink
- `BAT_THEME` env support is a stub (highlight.js has no bat theme selection)
- Output structure (line numbers, markers, backgrounds, word-diff) is identical

### file-index-src (Rust)

**Original:** Rust NAPI module wrapping `nucleo` (https://github.com/helix-editor/nucleo) for high-performance fuzzy file searching

**Replaced by:** `native-ts/file-index/`
- Reimplements nucleo-style scoring with bitmap filtering and top-k selection
- Same API: `new FileIndex()`, `.loadFromFileList()`, `.search()`

### yoga-layout (C++)

**Original:** Meta's Yoga layout engine compiled to WebAssembly/NAPI

**Replaced by:** `native-ts/yoga-layout/`
- Simplified single-pass flexbox implementation
- Covers the subset of features Ink actually uses
- ~2500 lines of C++ in CalculateLayout.cpp alone → ~3000 lines of TypeScript

## Why Native Modules Were Replaced

1. **Portability**: Native modules require compilation for each platform (macOS, Windows, Linux)
2. **Bundle size**: Native modules add significant binary size to the npm package
3. **Build complexity**: NAPI modules require Rust/C++ toolchains
4. **CI reliability**: Native modules caused CI timeouts on Windows (highlight.js lazy loading pushed tests into GC-pause territory)

## Lazy Loading Pattern

The color-diff port uses lazy loading for highlight.js to avoid paying the ~50MB heap cost at startup:

```typescript
let cachedHljs: HLJSApi | null = null
function hljs(): HLJSApi {
  if (cachedHljs) return cachedHljs
  const mod = require('highlight.js')
  cachedHljs = 'default' in mod && mod.default ? mod.default : mod
  return cachedHljs!
}
```

This mirrors the NAPI wrapper's lazy `dlopen` pattern — the full highlight.js bundle registers 190+ language grammars at require time (~100-200ms on macOS, severalx that on Windows).

## Related Modules

- [Native-TS Modules](./native-ts-modules.md) — The TypeScript ports that replace these native modules
- [UI Components](../06-ui/ink-framework.md) — Yoga layout powers Ink's layout system
