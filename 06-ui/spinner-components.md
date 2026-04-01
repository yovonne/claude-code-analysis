# Spinner Components

## Purpose

The Spinner component system provides animated loading indicators for Claude Code's terminal UI. It includes multiple animation styles (spinning glyphs, shimmer text, glimmer effects, flashing characters), stall detection with color transitions, reduced motion support, and teammate-specific spinner variants — all synchronized to a shared animation clock for performance.

## Location

`restored-src/src/components/Spinner/`

## Key Exports

### Core Components

| Component | Description |
|-----------|-------------|
| `SpinnerGlyph` | Main spinner character with frame-based animation, stall detection, and reduced motion support |
| `GlimmerMessage` | Message text with shimmer/glimmer animation effect |
| `ShimmerChar` | Individual character with shimmer color animation |
| `FlashingChar` | Character with opacity-based flash animation between base and shimmer colors |
| `SpinnerAnimationRow` | Complete animated status row combining spinner, message, timer, tokens, and thinking status |

### Teammate Components

| Component | Description |
|-----------|-------------|
| `TeammateSpinnerLine` | Spinner line for teammate agent display |
| `TeammateSpinnerTree` | Tree of teammate spinners for swarm visualization |

### Hooks

| Hook | Description |
|------|-------------|
| `useShimmerAnimation(mode, message, isStalled)` | Calculates glimmer index for shimmer animation; unsubscribes from clock when stalled |
| `useStalledAnimation(time, responseLength, hasActiveTools, reducedMotion)` | Detects token flow stalls and calculates smooth intensity fade to red |

### Utilities

| Function | Description |
|----------|-------------|
| `getDefaultCharacters()` | Returns platform-appropriate spinner characters (Ghostty, macOS, Linux variants) |
| `interpolateColor(color1, color2, t)` | Linear RGB interpolation between two colors |
| `toRGBColor(color)` | Converts RGB object to `rgb(r,g,b)` string for Text component |
| `parseRGB(colorStr)` | Parses `rgb(r,g,b)` string to RGB object (cached) |
| `hueToRgb(hue)` | HSL hue to RGB conversion (for voice-mode waveform colors) |

### Types

| Type | Description |
|------|-------------|
| `SpinnerMode` | `'requesting' | 'tool-use' | 'tool-input' | 'responding' | 'thinking'` |

## Dependencies

### Internal Dependencies

- `../../ink.js` — Ink framework (Box, Text, useTheme, useAnimationFrame)
- `../../ink/stringWidth.js` — String width calculation
- `../../utils/theme.js` — Theme definitions and `getTheme()`
- `../../utils/format.js` — Duration and number formatting
- `../../utils/ink.js` — `toInkColor()` utility
- `../../tasks/InProcessTeammateTask/types.js` — Teammate task state types
- `../design-system/Byline.js` — Byline component for status hints

### External Dependencies

- `react` — React 19
- `figures` — Unicode symbols (arrows for token count)

## Implementation Details

### SpinnerGlyph

The main spinner character component:

- **Frame animation**: Cycles through `SPINNER_FRAMES` (characters forward + reverse) based on frame index
- **Platform characters**: Ghostty uses `['·', '✢', '✳', '✶', '✻', '*']` (avoids offset `✽`), macOS uses `['·', '✢', '✳', '✶', '✻', '✽']`, Linux uses `['·', '✢', '*', '✶', '✻', '✽']`
- **Reduced motion**: Shows a single `●` dot that toggles dim/visible on a 2-second cycle
- **Stall detection**: Interpolates color from message color to error red (`rgb(171,43,63)`) based on `stalledIntensity` (0-1)
- **Fallback**: If RGB parsing fails, switches to `'error'` theme color when stalled intensity > 0.5

### GlimmerMessage

Message text with a moving shimmer effect:

- **Glimmer index**: Position of the shimmer highlight that sweeps across the message text
- **Speed**: 50ms cycle for `'requesting'` mode, 200ms for other modes
- **Flash opacity**: Sine-wave opacity for tool-use mode flash effect
- **Stall handling**: Glimmer index set to -100 (offscreen) when stalled
- **Color interpolation**: Smooth RGB interpolation between message color and shimmer color

### SpinnerAnimationRow

The complete animated status row (265 lines):

- **Animation isolation**: Owns the `useAnimationFrame(50)` subscription; parent components are freed from the 50ms render loop
- **Progressive width gating**: Dynamically shows/hides thinking, timer, and token count based on available terminal width
- **Components displayed**:
  - SpinnerGlyph (frame-based spinner character)
  - GlimmerMessage (animated message text)
  - Timer (elapsed time, formatted duration)
  - Token count (with mode-specific glyph: arrow down for tool-use/responding, arrow up for requesting)
  - Thinking status (with shimmer color animation)
  - Teammate status (foregrounded teammate name with color)
- **Elapsed time**: Wall-clock calculation respecting pause state (`pauseStartTimeRef`, `totalPausedMsRef`)
- **Token counter animation**: Smooth increment animation (3/8/50 tokens per frame depending on gap size)
- **Thinking shimmer**: Separate sine-wave opacity animation with 3-second delay and 2-second glow period
- **Status composition**: Builds parts array conditionally based on available space, wraps in Byline or parentheses

### useShimmerAnimation

Calculates shimmer position for text animation:

- **Glimmer speed**: 50ms for `'requesting'` mode, 200ms for others
- **Clock subscription**: Passes `null` to `useAnimationFrame` when stalled — unsubscribes from clock to save CPU
- **Cycle calculation**: `cyclePosition = floor(time / glimmerSpeed)`, `cycleLength = messageWidth + 20`
- **Direction**: `'requesting'` mode sweeps left-to-right, other modes sweep right-to-left
- **Offscreen**: Returns -100 when stalled (glimmer index outside visible range)

### useStalledAnimation

Detects when token flow has stalled and calculates smooth intensity:

- **Stall detection**: 3 seconds without new tokens (reset when `responseLength` increases)
- **Active tool override**: `hasActiveTools=true` resets stall timer (tools are still working)
- **Intensity fade**: Fades from 0 to 1 over 2 seconds after stall detection
- **Smooth transitions**: Interpolates intensity at 50ms intervals with 0.1 smoothing factor
- **Reduced motion**: Instant intensity change (no smoothing)
- **Mount handling**: Uses `mountTime` for initial stall calculation when no tokens yet

### Color Utilities

- **`interpolateColor(color1, color2, t)`**: Linear RGB interpolation with rounding
- **`parseRGB(colorStr)`**: Regex-based `rgb(r,g,b)` parser with Map cache
- **`toRGBColor(color)`**: Formats RGB object as `rgb(r,g,b)` string
- **`hueToRgb(hue)`**: Full HSL-to-RGB conversion (s=0.7, l=0.6) for voice-mode waveform colors

## Animation Architecture

```
Shared Clock (ClockContext)
    ↓
useAnimationFrame(50) — single subscriber drives the clock
    ↓
┌─────────────────────────────────────────────┐
│ SpinnerAnimationRow (50ms render loop)      │
│  ├── SpinnerGlyph (frame-based character)   │
│  ├── GlimmerMessage (shimmer sweep)         │
│  ├── Timer (elapsed time)                   │
│  ├── Token counter (smooth increment)       │
│  ├── Thinking shimmer (sine-wave opacity)   │
│  └── Teammate status                        │
└─────────────────────────────────────────────┘
    ↓
useStalledAnimation (stall detection + intensity)
    ↓
Color interpolation (message color → error red)
```

## Performance Optimizations

1. **Animation isolation**: Only `SpinnerAnimationRow` re-renders at 50ms; parent components render ~25x/turn instead of ~383x
2. **Clock unsubscription**: `useShimmerAnimation` passes `null` to `useAnimationFrame` when stalled — no ticks when animation is invisible
3. **Memoized stringWidth**: `stringWidth(message)` memoized across 50ms loop (Bun native call is expensive)
4. **Ref-based state**: Timer refs (`loadingStartTimeRef`, `totalPausedMsRef`) avoid re-renders on state changes
5. **Smooth intensity**: 50ms-step interpolation avoids jarring color jumps
6. **Reduced motion**: Disables shimmer, uses simple dot toggle, instant intensity changes
7. **Dead code elimination**: Teammate components use dynamic `require()` for tree-shaking when not needed

## Spinner Modes

| Mode | Description | Glimmer Speed | Glyph |
|------|-------------|---------------|-------|
| `requesting` | Waiting for API response | 50ms (fast) | ↑ arrow |
| `tool-use` | Executing a tool | 200ms (slow) | ↓ arrow |
| `tool-input` | Providing input to tool | 200ms | ↓ arrow |
| `responding` | Generating response | 200ms | ↓ arrow |
| `thinking` | Extended thinking mode | 200ms | — |

## Key Design Decisions

1. **Shared animation clock**: All spinners subscribe to a single ClockContext — animations stay in sync, one wake-up
2. **Frame-based animation**: Spinner character uses frame index (not time) for consistent animation speed
3. **Stall detection via token count**: Detects stalls by monitoring response length changes, not time alone
4. **Progressive width gating**: Status elements (thinking, timer, tokens) shown/hidden based on available terminal width
5. **Color interpolation over theme switching**: Smooth RGB interpolation for stall color transition instead of abrupt theme change
6. **Platform-specific characters**: Different spinner characters for Ghostty (avoids rendering offset), macOS, and Linux
7. **Ref-based timer state**: Uses refs instead of state for timer values to avoid unnecessary re-renders
8. **Teammate isolation**: Teammate spinner components dynamically imported for dead code elimination
