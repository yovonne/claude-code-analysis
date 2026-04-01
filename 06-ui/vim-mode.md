# Vim Mode

## Purpose

Vim Mode provides vim-style keyboard navigation and editing within Claude Code's text input fields. It implements a state machine for normal/insert modes, operator-motion grammar (delete/change/yank + motion), text objects, find motions, dot-repeat, and register-based yank/paste — all as pure functions with no side effects.

## Location

`restored-src/src/vim/`

## Key Exports

### Types (`types.ts`)

| Type | Description |
|------|-------------|
| `Operator` | `'delete' | 'change' | 'yank'` |
| `FindType` | `'f' | 'F' | 't' | 'T'` |
| `TextObjScope` | `'inner' | 'around'` |
| `VimState` | Union of INSERT mode (tracks `insertedText`) and NORMAL mode (tracks `CommandState`) |
| `CommandState` | State machine: `idle`, `count`, `operator`, `operatorCount`, `operatorFind`, `operatorTextObj`, `find`, `g`, `operatorG`, `replace`, `indent` |
| `PersistentState` | Cross-command memory: `lastChange`, `lastFind`, `register`, `registerIsLinewise` |
| `RecordedChange` | Dot-repeat data for all change types (insert, operator, text object, find, replace, x, toggleCase, indent, openLine, join) |

### Constants (`types.ts`)

| Constant | Description |
|----------|-------------|
| `OPERATORS` | `{d: 'delete', c: 'change', y: 'yank'}` |
| `SIMPLE_MOTIONS` | Set of basic motion keys (h, l, j, k, w, b, e, W, B, E, 0, ^, $) |
| `FIND_KEYS` | Set `{f, F, t, T}` |
| `TEXT_OBJ_SCOPES` | `{i: 'inner', a: 'around'}` |
| `TEXT_OBJ_TYPES` | Set of text object keys (w, W, quotes, parens, brackets, braces, angle brackets) |
| `MAX_VIM_COUNT` | `10000` — maximum count value |

### Factory Functions

| Function | Description |
|----------|-------------|
| `createInitialVimState()` | Returns `{mode: 'INSERT', insertedText: ''}` |
| `createInitialPersistentState()` | Returns empty persistent state |

### Motions (`motions.ts`)

| Function | Description |
|----------|-------------|
| `resolveMotion(key, cursor, count)` | Resolves a motion key to target cursor position (applies count times) |
| `isInclusiveMotion(key)` | Returns true for motions that include destination character (e, E, $) |
| `isLinewiseMotion(key)` | Returns true for linewise motions (G) |

Supported motions:
- **Character**: `h` (left), `l` (right)
- **Line**: `j`/`gj` (down logical/visual), `k`/`gk` (up logical/visual)
- **Word**: `w`/`b`/`e` (vim word), `W`/`B`/`E` (WORD — whitespace-delimited)
- **Line positions**: `0` (start), `^` (first non-blank), `$` (end)
- **File**: `G` (last line start)

### Operators (`operators.ts`)

| Function | Description |
|----------|-------------|
| `executeOperatorMotion(op, motion, count, ctx)` | Execute operator with a simple motion |
| `executeOperatorFind(op, findType, char, count, ctx)` | Execute operator with a find motion |
| `executeOperatorTextObj(op, objType, scope, count, ctx)` | Execute operator with a text object |
| `executeOperatorG(op, count, ctx)` | Execute operator + G (to line N) |
| `executeOperatorGg(op, count, ctx)` | Execute operator + gg (from line N) |
| `executeX(count, ctx)` | Delete character under cursor |
| `executeReplace(char, count, ctx)` | Replace character(s) under cursor |
| `executeToggleCase(count, ctx)` | Toggle case of character(s) |
| `executeIndent(dir, count, ctx)` | Indent/unindent line(s) |
| `executeOpenLine(direction, ctx)` | Open line above/below and enter insert mode |
| `executeJoin(count, ctx)` | Join line(s) |
| `executePaste(after, count, ctx)` | Paste from register (before/after cursor) |

**OperatorContext** provides: `cursor`, `text`, `setText`, `setOffset`, `enterInsert`, `getRegister`, `setRegister`, `getLastFind`, `setLastFind`, `recordChange`

### Text Objects (`textObjects.ts`)

| Function | Description |
|----------|-------------|
| `findTextObject(text, offset, objectType, isInner)` | Finds text object boundaries |

Supported text objects:
- **Words**: `w` (vim word), `W` (WORD — whitespace-delimited)
- **Quotes**: `"`, `'`, `` ` `` (inner/around)
- **Brackets**: `(`, `)`, `b` (parens), `[`, `]` (brackets), `{`, `}`, `B` (braces), `<`, `>` (angle brackets)

### State Transitions (`transitions.ts`)

| Function | Description |
|----------|-------------|
| `transition(state, input, ctx)` | Main transition function — dispatches based on current state type |

Transition handlers (internal):
- **`fromIdle(input, ctx)`**: Handles initial keypress — operators, counts, find, g, replace, indent, motions, x, paste, toggle case, open line, join
- **`fromCount(state, input, ctx)`**: Accumulates digit count or transitions to operator/find/g
- **`fromOperator(state, input, ctx)`**: Handles motion after operator — simple motions, find, text objects, G, gg
- **`fromOperatorCount(state, input, ctx)`**: Accumulates count after operator
- **`fromOperatorFind(state, input, ctx)`**: Handles character after operator+find
- **`fromOperatorTextObj(state, input, ctx)`**: Handles text object type after operator+scope
- **`fromFind(state, input, ctx)`**: Handles character after find motion (f/F/t/T)
- **`fromG(state, input, ctx)`**: Handles second key after `g` (gg, ge, etc.)
- **`fromOperatorG(state, input, ctx)`**: Handles second key after operator+g
- **`fromReplace(state, input, ctx)`**: Handles character to replace with
- **`fromIndent(state, input, ctx)`**: Handles second indent key (>>, <<)

**TransitionResult**: `{next?: CommandState, execute?: () => void}` — next state and optional side-effect function

## Dependencies

### Internal Dependencies

- `../utils/Cursor.js` — Cursor class with grapheme-aware navigation
- `../utils/intl.js` — Grapheme segmenter (`getGraphemeSegmenter`, `firstGrapheme`, `lastGrapheme`)
- `../utils/stringUtils.js` — String utilities (`countCharInString`)

### External Dependencies

- None — all pure functions

## State Machine

```
VimState
├── INSERT (tracks insertedText for dot-repeat)
└── NORMAL
    └── CommandState
        ├── idle ──┬─[d/c/y]──► operator
        │          ├─[1-9]────► count
        │          ├─[fFtT]───► find
        │          ├─[g]──────► g
        │          ├─[r]──────► replace
        │          └─[><]─────► indent
        │
        ├── operator ─┬─[motion]──► execute (return to idle)
        │            ├─[0-9]────► operatorCount
        │            ├─[ia]─────► operatorTextObj
        │            └─[fFtT]───► operatorFind
        │
        ├── count ────┬─[d/c/y]──► operator (with count)
        │            ├─[fFtT]───► find (with count)
        │            ├─[g]──────► g (with count)
        │            └─[0-9]────► accumulate digits
        │
        └── ... (similar for other states)
```

## Operator-Motion Grammar

The vim command structure follows the pattern: `[count] operator [count] motion`

Examples:
- `d3w` — delete 3 words (operator=d, count=3, motion=w)
- `ci"` — change inside quotes (operator=c, text object=i")
- `y$` — yank to end of line (operator=y, motion=$)
- `3dd` — delete 3 lines (count=3, operator=d, motion=d [linewise])
- `2f,` — find second comma (count=2, find=f, char=,)

## Dot-Repeat

Changes are recorded in `PersistentState.lastChange` for dot-repeat (`.`):

- **Insert changes**: `{type: 'insert', text}`
- **Operator changes**: `{type: 'operator', op, motion, count}`
- **Text object changes**: `{type: 'operatorTextObj', op, objType, scope, count}`
- **Find changes**: `{type: 'operatorFind', op, find, char, count}`
- **Other changes**: replace, x, toggleCase, indent, openLine, join

## Register System

- **Single register**: Stores yanked/deleted content
- **Linewise flag**: Tracks whether content is line-based (for paste positioning)
- **Get/Set**: Via `getRegister()` and `setRegister(content, linewise)` in OperatorContext

## Key Design Decisions

1. **Pure functions**: All motion resolution and operator execution are pure — no side effects, only return values and context mutations
2. **State machine via discriminated unions**: TypeScript ensures exhaustive handling of all states
3. **Grapheme-aware**: Uses Intl.Segmenter for grapheme cluster boundaries in word/text object detection
4. **Transition table pattern**: Each state has its own transition function — scannable source of truth
5. **OperatorContext abstraction**: All side effects (text modification, register access, mode changes) go through context interface
6. **Count accumulation**: Digits accumulate as strings, parsed to number on execution (supports arbitrary count up to MAX_VIM_COUNT)
7. **Find persistence**: Last find (f/F/t/T) is stored for repeat with `;` and `,`
8. **No external dependencies**: Entire vim mode is self-contained pure TypeScript
