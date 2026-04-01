# Context Providers

## Purpose

React context providers that supply shared state and utilities to the component tree. These contexts enable cross-component communication without prop drilling, covering notifications, modals, overlays, voice state, mailbox, stats, and FPS metrics.

## Location

`restored-src/src/context/`

## Key Providers

### `notifications.tsx` — Notification System

Manages a priority-based notification queue with folding, invalidation, and timeout support.

**Types:**
- `Priority`: `'low' | 'medium' | 'high' | 'immediate'`
- `Notification`: Either `TextNotification` (text + color) or `JSXNotification` (React node)
- `BaseNotification`: Shared fields — `key`, `invalidates`, `priority`, `timeoutMs`, `fold`

**Key exports:**
- `useNotifications()`: Returns `{ addNotification, removeNotification }`
- `getNext(queue)`: Selects highest-priority notification from queue (immediate > high > medium > low)

**Features:**
- **Priority queue**: Notifications are dequeued by priority, not FIFO
- **Immediate priority**: Bypasses the queue — shows right away and clears current timeout
- **Folding**: Notifications with the same `key` can merge via a `fold` function (like `Array.reduce`)
- **Invalidation**: A notification can invalidate others by `key`, removing them from queue immediately
- **Timeout**: Auto-dismisses after `timeoutMs` (default 8000ms)
- **Deduplication**: Prevents duplicate notifications with the same key

### `modalContext.tsx` — Modal Slot Context

Provides sizing and scroll information for content rendered inside the modal slot (bottom-anchored pane for slash-command dialogs).

**Type `ModalCtx`:**
- `rows`: Available content rows
- `columns`: Available content columns
- `scrollRef`: Ref to modal's ScrollBox

**Key exports:**
- `useIsInsideModal()`: Returns true when component is inside the modal slot
- `useModalOrTerminalSize(fallback)`: Returns modal dimensions or falls back to terminal size
- `useModalScrollRef()`: Returns the modal's scroll ref for scroll reset on tab switch

**Use cases:**
- Suppress top-level framing (Pane skips its Divider)
- Size Select pagination to available rows (not full terminal)
- Reset scroll on tab switch via ScrollBox keying

### `overlayContext.tsx` — Overlay Tracking

Tracks active overlays for Escape key coordination.

**Key exports:**
- `useRegisterOverlay(id, enabled?)`: Auto-registers on mount, unregisters on unmount
- `useIsOverlayActive()`: Returns true if any overlay is active
- `useIsModalOverlayActive()`: Returns true if any modal (non-autocomplete) overlay is active

**Non-modal overlays:** `autocomplete` is excluded from modal overlay detection so it doesn't disable TextInput focus.

### `promptOverlayContext.tsx` — Floating Prompt Overlay

Portal for content that floats above the prompt, escaping FullscreenLayout's `overflowY:hidden` clip.

**Two channels:**
- **Suggestions data**: Structured slash-command suggestion data (written by PromptInputFooter)
- **Dialog node**: Arbitrary dialog nodes like AutoModeOptInDialog (written by PromptInput)

**Key exports:**
- `usePromptOverlay()`: Returns suggestion data
- `usePromptOverlayDialog()`: Returns dialog ReactNode
- `useSetPromptOverlay(data)`: Registers suggestion data (clears on unmount)
- `useSetPromptOverlayDialog(node)`: Registers dialog node (clears on unmount)

**Design:** Split into data/setter context pairs so writers never re-render on their own writes.

### `voice.tsx` — Voice State

Manages voice input state using a custom store pattern with `useSyncExternalStore`.

**Type `VoiceState`:**
- `voiceState`: `'idle' | 'recording' | 'processing'`
- `voiceError`: Error message or null
- `voiceInterimTranscript`: Current transcription
- `voiceAudioLevels`: Audio level array for visualization
- `voiceWarmingUp`: Warm-up indicator

**Key exports:**
- `useVoiceState(selector)`: Subscribe to a slice of voice state (only re-renders when selected value changes)
- `useSetVoiceState()`: Get the state setter (stable reference, never causes re-renders)
- `useGetVoiceState()`: Get synchronous reader for fresh state inside callbacks

### `mailbox.tsx` — Mailbox Context

Provides a singleton `Mailbox` instance for inter-agent communication.

**Key exports:**
- `useMailbox()`: Returns the mailbox instance (throws if used outside provider)

### `stats.tsx` — Stats Store

In-memory metrics store with counters, gauges, histograms, and sets.

**Type `StatsStore`:**
- `increment(name, value?)`: Add to a counter
- `set(name, value)`: Set a gauge value
- `observe(name, value)`: Record a histogram observation (reservoir sampling, size 1024)
- `add(name, value)`: Add to a string set
- `getAll()`: Returns all metrics as a flat record

**Histogram features:**
- Reservoir sampling (Algorithm R) for bounded memory
- Computes count, min, max, avg, p50, p95, p99
- Flushes to project config on process exit

**Key exports:**
- `createStatsStore()`: Factory function
- `useStats()`: Returns the stats store
- `useCounter(name)`: Returns an increment function for a named counter
- `useGauge(name)`: Returns a set function for a named gauge
- `useTimer(name)`: Returns an observe function for a named timer
- `useSet(name)`: Returns an add function for a named set

### `fpsMetrics.tsx` — FPS Metrics

Provides access to FPS tracking metrics.

**Key exports:**
- `useFpsMetrics()`: Returns a getter function for FPS metrics

### `QueuedMessageContext.tsx` — Queued Message Layout

Provides layout context for queued messages in multi-agent scenarios.

**Type `QueuedMessageContextValue`:**
- `isQueued`: Always true for children
- `isFirst`: Whether this is the first queued message
- `paddingWidth`: Width reduction for container padding

**Key exports:**
- `useQueuedMessage()`: Returns the context value
- `QueuedMessageProvider`: Wraps children with padding and context

## Dependencies

### Internal Dependencies

- `state/AppState.js` — App state store and setters
- `utils/config.js` — Project config persistence (stats flush)
- `utils/mailbox.js` — Mailbox implementation
- `utils/fpsTracker.js` — FPS tracking
- `ink/components/ScrollBox.js` — ScrollBox handle type
- `ink.js` — Box component for layout

### External Dependencies

- `react` — Context creation and hooks
- `react/compiler-runtime` — React Compiler optimization

## Implementation Details

### Store Pattern (Voice, Stats)

Both voice and stats contexts use a custom store pattern instead of React state:
- `createStore(initialState)` creates an external store with `getState`, `setState`, `subscribe`
- `useSyncExternalStore` subscribes components to specific slices
- Setters are stable references that never cause re-renders

### Notification Queue Algorithm

```
addNotification(notif):
  if priority == 'immediate':
    Clear current timeout
    Show immediately, re-queue non-immediate items
    Set new timeout
  else:
    Check for fold (same key) → merge
    Check for invalidation → remove invalidated
    Check for duplicate → skip
    Add to queue
  processQueue()

processQueue():
  If nothing current:
    Get highest priority from queue (getNext)
    Set as current
    Start timeout
```

## Related Modules

- [Bootstrap](./bootstrap.md) — Stats store reference stored in bootstrap state
- [Hooks System](./hooks-system.md) — Notification hooks consume the notification context
- [Schemas](./schemas.md) — Hook schemas define hook configuration
