# Keybindings System

## Purpose

The keybindings system provides a configurable, context-aware keyboard shortcut management layer for Claude Code's terminal UI. It supports multi-key chord sequences, platform-specific display, user overrides via JSON config, reserved shortcut validation, and React hooks for declarative binding in components.

## Location

`restored-src/src/keybindings/`

## Key Exports

### Core Modules

| Module | Description |
|--------|-------------|
| `parser.ts` | Parses keystroke/chord strings into structured `ParsedKeystroke` objects |
| `schema.ts` | Zod validation schema for `keybindings.json` configuration |
| `resolver.ts` | Resolves key input to actions with chord state management |
| `match.ts` | Modifier matching logic between Ink's Key and parsed bindings |
| `defaultBindings.ts` | Default keybinding definitions for all contexts |
| `reservedShortcuts.ts` | Reserved/non-rebindable shortcut definitions |
| `template.ts` | Keybinding config template generation |
| `validate.ts` | User keybinding validation logic |
| `shortcutFormat.ts` | Shortcut display formatting |

### React Integration

| Module | Description |
|--------|-------------|
| `KeybindingContext.tsx` | React context providing resolution, handlers, and state |
| `KeybindingProviderSetup.tsx` | Provider setup with bindings loading and state management |
| `useKeybinding.ts` | Hook for registering a single action handler |
| `useShortcutDisplay.ts` | Hook for displaying the current shortcut for an action |

### User Bindings

| Module | Description |
|--------|-------------|
| `loadUserBindings.ts` | Loads and merges user `keybindings.json` with defaults |

## Dependencies

### Internal Dependencies

- `../ink.js` — Ink framework (Key type, useInput)
- `../ink/events/input-event.js` — InputEvent type
- `../utils/platform.js` — Platform detection (macos, windows, linux)
- `../utils/bundledMode.js` — Bun vs Node detection
- `../utils/semver.js` — Semver comparison for VT mode support
- `../utils/lazySchema.js` — Lazy Zod schema evaluation
- `../utils/envUtils.js` — Environment variable utilities
- `../hooks/` — Shared hooks

### External Dependencies

- `react` — React 19
- `zod/v4` — Schema validation
- `bun:bundle` — Bun feature flag system

## Implementation Details

### Parsing (`parser.ts`)

Converts string representations to structured data:

- **`parseKeystroke(input)`**: Parses `"ctrl+shift+k"` into `{key, ctrl, alt, shift, meta, super}`
- **`parseChord(input)`**: Parses `"ctrl+k ctrl+s"` into array of `ParsedKeystroke`
- **Modifier aliases**: `ctrl`/`control`, `alt`/`opt`/`option`/`meta`, `cmd`/`command`/`super`/`win`
- **Key aliases**: `esc`→`escape`, `return`→`enter`, `space`→` `, arrows (`↑`,`↓`,`←`,`→`)
- **`keystrokeToString(ks)`**: Canonical string representation
- **`chordToString(chord)`**: Joins keystrokes with spaces
- **Platform display**: `keystrokeToDisplayString()` uses `"opt"` for alt on macOS, `"alt"` elsewhere
- **`parseBindings(blocks)`**: Parses JSON config blocks into flat `ParsedBinding[]` array

### Schema (`schema.ts`)

Zod schema for `keybindings.json` validation:

- **Contexts**: 20+ valid contexts (`Global`, `Chat`, `Autocomplete`, `Confirmation`, `Help`, `Transcript`, `HistorySearch`, `Task`, `ThemePicker`, `Settings`, `Tabs`, `Attachments`, `Footer`, `MessageSelector`, `DiffDialog`, `ModelPicker`, `Select`, `Plugin`)
- **Actions**: 80+ valid action identifiers organized by domain (`app:*`, `chat:*`, `confirm:*`, `tabs:*`, `history:*`, `autocomplete:*`, `select:*`, `plugin:*`, etc.)
- **Binding values**: Action string, `command:*` command binding, or `null` to unbind
- **Metadata**: Optional `$schema` and `$docs` fields for editor validation

### Resolution (`resolver.ts`)

Resolves key input to actions with chord support:

- **`resolveKey(input, key, activeContexts, bindings)`**: Single-keystroke resolution (last binding wins for user overrides)
- **`resolveKeyWithChordState(input, key, activeContexts, bindings, pendingChord)`**: Chord-aware resolution
- **Result types**: `match` (action found), `none` (no binding), `unbound` (explicitly null), `chord_started` (first keystroke of chord), `chord_cancelled` (invalid continuation)
- **`getBindingDisplayText(action, context, bindings)`**: Gets display text for an action (searches in reverse for user override precedence)
- **`buildParsedKeystroke(input, key)`**: Builds `ParsedKeystroke` from Ink's input/key

### Matching (`match.ts`)

Modifier matching between Ink and parsed bindings:

- **`getKeyName(input, key)`**: Extracts normalized key name from Ink's Key object (maps boolean flags like `key.escape`, `key.return` to string names)
- **`modifiersMatch(inkMods, target)`**: Checks modifier equality
- **Alt/Meta aliasing**: Ink sets `key.meta` for Alt/Option — both `alt` and `meta` in config match when `key.meta` is true
- **Super (Cmd/Win)**: Distinct from alt/meta; only arrives via kitty keyboard protocol
- **`matchesBinding(input, key, binding)`**: Full binding match check
- **Wheel support**: `wheelup`/`wheeldown` key names for scroll wheel events

### Default Bindings (`defaultBindings.ts`)

Platform-aware default keybindings:

- **Global**: `ctrl+c` (interrupt), `ctrl+d` (exit), `ctrl+l` (redraw), `ctrl+t` (toggle todos), `ctrl+o` (toggle transcript), `ctrl+r` (history search)
- **Chat**: `escape` (cancel), `enter` (submit), `up`/`down` (history), `ctrl+x ctrl+k` (kill agents), mode cycle key
- **Platform differences**: Image paste uses `alt+v` on Windows, `ctrl+v` elsewhere; mode cycle uses `shift+tab` or `meta+m` depending on VT mode support
- **VT mode detection**: Checks Node.js >= 22.17.0/<23.0.0 or >= 24.2.0, or Bun >= 1.2.23 for Windows VT mode support
- **Feature-gated bindings**: Quick search (`ctrl+shift+f`), terminal panel (`meta+j`), brief toggle (`ctrl+shift+b`)

### Reserved Shortcuts (`reservedShortcuts.ts`)

Defines shortcuts that cannot or should not be rebound:

- **Non-rebindable**: `ctrl+c`, `ctrl+d` (hardcoded interrupt/exit), `ctrl+m` (identical to Enter in terminals)
- **Terminal reserved**: `ctrl+z` (SIGTSTP), `ctrl+\` (SIGQUIT)
- **macOS reserved**: `cmd+c/v/x/q/w/tab/space` (system-level interception)
- **Severity levels**: `error` (will never work) vs `warning` (likely intercepted)
- **`getReservedShortcuts()`**: Returns platform-appropriate reserved list
- **`getNonRebindableShortcuts()`**: Returns shortcuts that cannot be rebound
- **`isReservedShortcut(chord)`**: Checks if a chord is reserved
- **`getReservedWarning(chord)`**: Gets warning message for a reserved shortcut

### React Context (`KeybindingContext.tsx`)

Provides keybinding resolution and handler management:

- **`KeybindingContextValue`**: `{resolve, setPendingChord, getDisplayText, bindings, pendingChord, activeContexts, registerActiveContext, unregisterActiveContext, registerHandler, invokeAction}`
- **Handler registry**: `Map<string, Set<HandlerRegistration>>` maps action names to handler sets
- **Pending chord ref**: `RefObject` for immediate access (avoids React state delay in hot path)
- **Active contexts**: `Set<KeybindingContextName>` tracks which contexts are currently active
- **`KeybindingProvider`**: Wraps children with context value, memoized resolve/getDisplayText functions

### useKeybinding Hook (`useKeybinding.ts`)

Declarative keybinding registration:

- **Single binding**: `useKeybinding(action, handler, options?)`
- **Multiple bindings**: `useKeybindings(bindingsMap, options?)`
- **Options**: `context` (default `'Global'`), `isActive` (enable/disable)
- **Chord support**: Automatically manages pending chord state
- **stopImmediatePropagation**: Prevents other handlers from firing once binding is handled
- **Handler return**: Returning `false` prevents `stopImmediatePropagation` (allows other handlers)
- **Context building**: Checks registered active contexts + binding context + Global (deduplicated, first occurrence wins)

### useShortcutDisplay Hook (`useShortcutDisplay.ts`)

Gets display text for an action:

- **`useShortcutDisplay(action, context?)`**: Returns formatted shortcut string (e.g., `"ctrl+t"`)
- **Platform-aware**: Uses platform-appropriate modifier names
- **Reactive**: Updates when bindings change

### User Bindings Loading (`loadUserBindings.ts`)

Loads and merges user configuration:

- **Default bindings loaded first**, then user bindings override
- **Validation**: User bindings validated against schema
- **Error handling**: Invalid bindings reported with specific error messages
- **Merge strategy**: User bindings appended after defaults (last-one-wins resolution)

## Data Flow

```
keybindings.json (user config)
    ↓
loadUserBindings.ts (load + validate + merge with defaults)
    ↓
ParsedBinding[] (flat array of all bindings)
    ↓
KeybindingProviderSetup.tsx (creates context with bindings + state)
    ↓
KeybindingContext.tsx (provides resolve, handlers, pending chord)
    ↓
useKeybinding.ts (components register handlers)
    ↓
useInput (Ink hook captures keyboard input)
    ↓
resolveKeyWithChordState (resolves input → action)
    ↓
invokeAction (calls registered handlers)
    ↓
stopImmediatePropagation (prevents further handling)
```

## Context Priority Resolution

Contexts are checked in priority order:
1. Registered active contexts (component-declared, e.g., "Chat" when input is focused)
2. Binding's declared context
3. `Global` (always checked last)

More specific contexts take precedence. Last binding wins within the same context (user overrides defaults).

## Key Design Decisions

1. **Last-one-wins resolution**: User bindings appended after defaults, so user config always wins
2. **Chord state in ref**: Pending chord stored in `RefObject` for hot-path access without React state delay
3. **Platform-aware display**: Modifier names adapt to platform (`opt` vs `alt`, `cmd` vs `super`)
4. **Reserved shortcut validation**: Prevents users from binding shortcuts that will never work
5. **Handler return value**: Returning `false` from handler prevents `stopImmediatePropagation`, allowing other handlers
6. **Context registration**: Components register/unregister contexts on mount/unmount for accurate priority resolution
7. **Zod schema validation**: User config validated with descriptive errors
8. **Command bindings**: `command:*` syntax allows binding keys to slash commands
