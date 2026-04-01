# Design System

## Purpose

The design system provides theme-aware UI primitives that form the foundation of Claude Code's terminal interface. It bridges the raw Ink components with the application's theme system, enabling consistent styling across all UI components through theme key resolution rather than hardcoded colors.

## Location

`restored-src/src/components/design-system/`

## Key Exports

### Core Components

| Component | Description |
|-----------|-------------|
| `ThemedBox` | Theme-aware Box; resolves theme color keys (borderColor, backgroundColor) to raw colors |
| `ThemedText` | Theme-aware Text; resolves color/backgroundColor theme keys, supports hover color context |
| `ThemeProvider` | Provides theme context with auto/system theme detection, preview, and persistence |
| `Dialog` | Modal dialog with title, subtitle, border pane, input guide, and cancel keybindings |
| `FuzzyPicker<T>` | Generic fuzzy-search picker with filtering, preview, keyboard navigation, and action hints |
| `Tabs` / `Tab` | Tabbed interface with keyboard navigation, controlled/uncontrolled modes, and content height management |
| `Pane` | Bordered container with optional color theming |
| `ListItem` | Selectable list item with focus state |
| `Byline` | Compact hint/action line for keyboard shortcuts |
| `KeyboardShortcutHint` | Displays a keyboard shortcut with action label |
| `ProgressBar` | Unicode block-character progress bar with sub-character precision |
| `Ratchet` | Progress indicator with incremental steps |
| `StatusIcon` | Status indicator icon |
| `LoadingState` | Loading state display |
| `Divider` | Horizontal divider line |

### Utilities

| Export | Description |
|--------|-------------|
| `color(c, theme, type?)` | Curried theme-aware color function; resolves theme keys to raw colors |
| `TextHoverColorContext` | React context for overriding text color on hover across subtree |

## Dependencies

### Internal Dependencies

- `../../ink.js` — Ink framework (Box, Text, useTheme, useInput, useTerminalFocus, etc.)
- `../../ink/styles.js` — Ink style types
- `../../ink/dom.js` — DOMElement type
- `../../ink/events/` — Event types (ClickEvent, FocusEvent, KeyboardEvent)
- `../../utils/theme.js` — Theme definitions and `getTheme()`
- `../../utils/systemTheme.js` — System theme detection
- `../../utils/config.js` — Config persistence (`getGlobalConfig`, `saveGlobalConfig`)
- `../../ink/hooks/use-stdin.js` — useStdin for terminal querier access
- `../../hooks/` — Shared hooks (useExitOnCtrlCDWithKeybindings, useSearchInput, useTerminalSize)
- `../../keybindings/useKeybinding.js` — useKeybinding, useKeybindings
- `../../context/modalContext.js` — Modal context (useIsInsideModal, useModalScrollRef)

### External Dependencies

- `react` — React 19
- `bun:bundle` — Bun feature flag system
- `zod/v4` — Schema validation (for FuzzyPicker)

## Implementation Details

### ThemeProvider

Manages the application's theme state with these capabilities:

- **Theme settings**: `'dark' | 'light' | 'auto'`
- **Auto theme**: Watches terminal theme changes via OSC 11 polling when `AUTO_THEME` feature is enabled
- **Preview mode**: Temporary theme preview (used by ThemePicker) with save/cancel
- **Persistence**: Saves to global config on theme change
- **System theme seeding**: Seeds from `$COLORFGBG` environment variable, corrected by OSC 11 watcher
- **Context value**: `{themeSetting, setThemeSetting, setPreviewTheme, savePreview, cancelPreview, currentTheme}`

### ThemedBox

Wraps the base `Box` component with theme resolution:

- **Color props**: `borderColor`, `borderTopColor`, `borderBottomColor`, `borderLeftColor`, `borderRightColor`, `backgroundColor`
- **Resolution**: Checks if value is a raw color (starts with `rgb(`, `#`, `ansi256(`, `ansi:`) — if so, passes through; otherwise looks up in theme
- **Event support**: Full event handler support (onClick, onFocus, onBlur, onKeyDown, onMouseEnter, onMouseLeave)
- **Layout props**: All Box layout props (flexWrap, flexDirection, flexGrow, flexShrink, overflow, etc.)

### ThemedText

Wraps the base `Text` component with theme resolution:

- **Color props**: `color` and `backgroundColor` accept theme keys
- **Hover color context**: `TextHoverColorContext` allows parent components to set a hover color that propagates to uncolored ThemedText in the subtree
- **Precedence**: explicit `color` > hover color > dimColor (uses theme's inactive color)
- **Style props**: `bold`, `italic`, `underline`, `strikethrough`, `inverse`, `dimColor`, `wrap`

### Dialog

Reusable dialog component with:

- **Structure**: Title (bold, themed) → Subtitle (dim) → Content (gap=1) → Input guide
- **Border**: Optional colored border via `Pane` component
- **Cancel handling**: Built-in `confirm:no` keybinding (Esc/n) and app:exit/interrupt (Ctrl+C/D)
- **Configurable cancel**: `isCancelActive` prop to disable cancel keybindings when embedded text field is focused
- **Custom input guide**: `inputGuide` render prop receives `exitState` for Ctrl+C/D pending display
- **Color**: Themed border color (default: `'permission'`)

### FuzzyPicker

Generic fuzzy-search picker component:

- **Generic type**: `<T>` for any item type
- **Features**: Search input, filtered list, keyboard navigation (up/down/Enter), preview panel, action hints (Tab/Shift+Tab)
- **Layout**: Title → SearchBox → List → Preview → Hints
- **Direction**: `'down'` (default) or `'up'` (atuin-style, items[0] at bottom)
- **Visibility**: Adaptive visible count based on terminal height (min 2, default 8, chrome accounts for 10 rows)
- **Actions**: Primary (Enter), secondary (Tab), tertiary (Shift+Tab) with configurable handlers and hints
- **Focus tracking**: `onFocus` callback for async preview loading
- **Empty state**: Customizable empty message
- **Match label**: Status line showing match count

### Tabs

Tabbed interface with keyboard navigation:

- **Structure**: Tab header row → Optional banner → Tab content
- **Navigation**: Left/right arrows or Tab/Shift+Tab switch tabs
- **Modes**: Controlled (`selectedTab` + `onTabChange`) or uncontrolled (`defaultTab`)
- **Content height**: Fixed `contentHeight` prevents layout shifts between tabs (overflow hidden)
- **Header focus**: Header can be focused/blurred; content can opt-in to cede navigation keys back to header
- **Nav from content**: `navFromContent` allows Tab/arrow keys to switch tabs from focused content
- **Context**: `TabsContext` provides `selectedTab`, `width`, `headerFocused`, `focusHeader`, `blurHeader`, `registerOptIn`
- **Hooks**: `useTabsWidth()` for content width, `useTabHeaderFocus()` for header focus gating

### ProgressBar

Unicode block-character progress bar:

- **Characters**: Uses 9 block characters (` `, `▏`, `▎`, `▍`, `▌`, `▋`, `▊`, `▉`, `█`) for sub-character precision
- **Props**: `ratio` (0-1), `width` (character width), `fillColor`, `emptyColor`
- **Rendering**: Filled portion uses theme color, empty portion uses background theme color
- **Precision**: Calculates whole blocks + partial block + empty blocks

## Data Flow

```
ThemeProvider (provides currentTheme)
    ↓
ThemedBox / ThemedText (resolve theme keys → raw colors)
    ↓
Ink Box / Text (render with resolved colors)
    ↓
Terminal output (ANSI escape sequences)
```

## Theme Resolution

Color values follow this resolution chain:

1. **Raw color detection**: If value starts with `rgb(`, `#`, `ansi256(`, or `ansi:` — use as-is
2. **Theme key lookup**: Otherwise, look up value in `getTheme(themeName)[key]`
3. **Fallback**: Undefined colors are passed through (Ink handles defaults)

## Key Design Decisions

1. **Theme keys over raw colors**: Components accept theme keys (`'primary'`, `'permission'`, etc.) for consistent theming
2. **Preview mode**: Theme changes can be previewed before committing, enabling smooth theme picker UX
3. **Auto theme detection**: Watches terminal theme changes via OSC 11 for seamless dark/light transitions
4. **Hover color context**: Cross-component hover coloring without prop drilling
5. **Progressive width gating**: FuzzyPicker and Tabs adapt to available terminal width
6. **React Compiler**: All components use `react/compiler-runtime` for automatic memoization
7. **Controlled/uncontrolled**: Tabs and other components support both patterns for flexibility
