# Buddy System (Companion)

## Purpose

The Buddy System adds a persistent animated companion ("buddy") to the Claude Code terminal UI. The companion is a procedurally generated ASCII art character with species, rarity, stats, hats, and personality. It displays idle animations, speech bubbles for reactions, and responds to the `/buddy` command.

## Location

`restored-src/src/buddy/` — 6 files:
- `types.ts` — Type definitions, species, rarities, stats
- `companion.ts` — Companion generation (deterministic PRNG-based rolling)
- `sprites.ts` — ASCII art sprite rendering with animation frames and hats
- `CompanionSprite.tsx` — React component for rendering the companion in the UI
- `prompt.ts` — System prompt injection for companion awareness
- `useBuddyNotification.tsx` — Notification hook and `/buddy` command detection

## Key Exports

### Types (types.ts)

- `Rarity`: `'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'`
- `Species`: 18 species including duck, goose, cat, dragon, octopus, owl, etc.
- `Eye`: 6 eye styles (`·`, `✦`, `×`, `◉`, `@`, `°`)
- `Hat`: 8 hat types (none, crown, tophat, propeller, halo, wizard, beanie, tinyduck)
- `StatName`: 5 stats (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK)
- `CompanionBones`: Deterministic physical traits derived from `hash(userId)`
- `CompanionSoul`: Model-generated name and personality
- `Companion`: Full companion object (bones + soul + hatchedAt)
- `StoredCompanion`: What persists in config (soul only; bones regenerate)

### Constants (types.ts)

- `RARITY_WEIGHTS`: common(60), uncommon(25), rare(10), epic(4), legendary(1)
- `RARITY_STARS`: Star display strings per rarity
- `RARITY_COLORS`: Theme color mapping per rarity
- `SPECIES`: Array of 18 species names

### Functions (companion.ts)

- `roll(userId)`: Deterministic companion roll using Mulberry32 PRNG seeded from hashed userId. Cached per userId.
- `rollWithSeed(seed)`: Non-cached roll with arbitrary seed string.
- `companionUserId()`: Derives the user ID from OAuth account UUID or config userID.
- `getCompanion()`: Merges regenerated bones with stored soul from config. Returns undefined if no companion hatched.

### Functions (sprites.ts)

- `renderSprite(bones, frame)`: Renders a 5-line ASCII art sprite with eye substitution and hat overlay.
- `spriteFrameCount(species)`: Returns number of animation frames for a species.
- `renderFace(bones)`: Returns a compact one-line face representation for narrow terminals.

### Functions (prompt.ts)

- `companionIntroText(name, species)`: Generates the system prompt section explaining the companion to the model.
- `getCompanionIntroAttachment(messages)`: Returns a `companion_intro` attachment if the companion hasn't been announced yet in the current message list.

### Functions (useBuddyNotification.tsx)

- `isBuddyTeaserWindow()`: Returns true during April 1-7, 2026 teaser period.
- `isBuddyLive()`: Returns true after April 2026 (feature permanently live).
- `useBuddyNotification()`: React hook that shows a rainbow `/buddy` teaser notification on startup when no companion exists.
- `findBuddyTriggerPositions(text)`: Finds all `/buddy` command positions in text for syntax highlighting.

### React Components (CompanionSprite.tsx)

- `CompanionSprite`: Main component rendering the companion with idle animation, speech bubbles, and pet effects.
- `CompanionFloatingBubble`: Overlay bubble for fullscreen mode (avoids overflow clipping).
- `companionReservedColumns()`: Calculates how many terminal columns the companion UI occupies.

## Dependencies

### Internal Dependencies

- `../utils/config.js` — `getGlobalConfig()` for companion storage and mute state
- `../state/AppState.js` — `companionReaction`, `companionPetAt`, `footerSelection` state
- `../hooks/useTerminalSize.js` — Terminal dimensions for responsive rendering
- `../ink.js` — Ink framework components (Box, Text)
- `../context/notifications.js` — Notification system for teaser
- `../utils/fullscreen.js` — Fullscreen mode detection
- `../utils/theme.js` — Theme color types
- `../utils/thinking.js` — `getRainbowColor()` for teaser text
- `bun:bundle` — Feature flag system (`feature('BUDDY')`)

### External Dependencies

- `react` — React hooks (useEffect, useRef, useState)
- `react/compiler-runtime` — React Compiler optimizations
- `figures` — Terminal Unicode symbols (hearts, etc.)

## Implementation Details

### Companion Generation

Companions are generated deterministically from the user's ID using a seeded PRNG:

1. **Hash** — `hashString(userId + SALT)` produces a numeric seed (uses `Bun.hash` when available, FNV-1a fallback otherwise)
2. **PRNG** — Mulberry32 seeded PRNG generates the random sequence
3. **Rarity roll** — Weighted selection: common 60%, uncommon 25%, rare 10%, epic 4%, legendary 1%
4. **Species/Eye/Hat** — Uniform random selection from respective arrays
5. **Stats** — One peak stat, one dump stat, rest scattered. Rarity sets the floor (common=5, uncommon=15, rare=25, epic=35, legendary=50)
6. **Shiny** — 1% chance for a shiny variant
7. **Caching** — Result is cached per userId to avoid redundant computation across hot paths (500ms tick, per-keystroke, per-turn observer)

### Security Design

Species names are constructed via `String.fromCharCode()` rather than literals to avoid triggering build-time canary checks in `excluded-strings.txt`. This keeps the build pipeline's secret-detection armed for actual codenames while allowing species names in the bundle.

Bones are **never persisted** — they regenerate from `hash(userId)` on every read. Only the soul (name, personality, hatchedAt) is stored. This means:
- Species renames don't break stored companions
- Users can't edit their config to fake a legendary rarity
- The same user always gets the same companion

### Sprite Animation

Each species has 3 animation frames for idle fidget behavior:

- **Idle sequence**: `[0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]` — mostly rest (frame 0), occasional fidget (frames 1-2), rare blink (-1)
- **Tick rate**: 500ms per tick
- **Excited state**: When reacting or being pet, cycles through all frames rapidly
- **Blink**: Frame -1 replaces the eye character with `-` for one tick

### UI Rendering

The companion adapts to terminal width:

- **Wide terminals (≥100 cols)**: Full sprite with optional speech bubble
- **Narrow terminals (<100 cols)**: Compact one-line face + name display
- **Fullscreen mode**: Bubble renders as floating overlay (`CompanionFloatingBubble`) to avoid `overflowY: hidden` clipping
- **Non-fullscreen**: Bubble renders inline beside the sprite, shrinking the input area

Speech bubble behavior:
- Shows for 20 ticks (~10 seconds)
- Fades in the last 6 ticks (~3 seconds) so users know it's about to disappear
- Text wraps at 30 characters width
- Pet effect: 5-frame heart animation floating above the sprite (~2.5 seconds)

### Buddy Teaser

During the teaser window (April 1-7, 2026), users without a companion see a rainbow-colored `/buddy` notification on startup. After April 2026, the feature is permanently live. The `external === 'ant'` check bypasses date restrictions for internal builds.

## Data Flow

```
User launches Claude Code
    ↓
getCompanion() → checks config.companion
    ↓ (if exists)
roll(userId) → regenerate bones from hash
    ↓
merge stored soul + regenerated bones
    ↓
CompanionSprite renders:
    ├── 500ms tick interval → cycle idle animation
    ├── AppState.companionReaction → show speech bubble
    ├── AppState.companionPetAt → show heart animation
    └── footerSelection === 'companion' → focused state
    ↓
User types /buddy → findBuddyTriggerPositions() highlights it
```

## Integration Points

- **AppState** — `companionReaction` (speech bubble text), `companionPetAt` (pet timestamp), `footerSelection` (focus state)
- **System prompt** — `companionIntroText()` injects companion awareness into the model's instructions
- **Message attachments** — `getCompanionIntroAttachment()` adds a `companion_intro` attachment to announce the companion
- **Config storage** — `config.companion` stores the soul; `config.companionMuted` disables display
- **Terminal layout** — `companionReservedColumns()` tells the input area how much width to reserve
- **Notification system** — `useBuddyNotification()` shows the teaser notification

## Configuration

| Config Field | Type | Purpose |
|---|---|---|
| `config.companion` | `StoredCompanion \| undefined` | Persisted companion soul (name, personality, hatchedAt) |
| `config.companionMuted` | `boolean` | Disables companion display when true |
| `config.oauthAccount.accountUuid` | `string` | Used as the deterministic seed for companion generation |

| Feature Gate | Purpose |
|---|---|
| `BUDDY` | Master toggle for all buddy functionality |

## Error Handling

- Missing companion: `getCompanion()` returns `undefined`; components render `null`
- Muted companion: `companionMuted` check causes early return in all rendering paths
- Feature gate disabled: `feature('BUDDY')` check causes early return, zero overhead

## Testing

- `rollWithSeed()` allows deterministic testing with known seeds
- `rollCache` can be cleared between tests for isolation
- The PRNG (Mulberry32) is deterministic, so the same seed always produces the same companion

## Related Modules

- [Config](../05-utils/config.md) — Companion storage in global config
- [State Management](../01-core-modules/state-management.md) — AppState for reactions and pet state
- [Ink Framework](../06-ui/ink-framework.md) — Terminal UI rendering

## Notes

- The companion is a terminal UI feature, not an AI agent — it's an animated ASCII art character with procedurally generated traits
- The 18 species include: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk
- The `/buddy` command triggers the companion hatching flow (handled elsewhere in the command system)
- Pet hearts float in 5 frames over ~2.5 seconds using Unicode heart symbols from the `figures` package
- The companion's rarity determines its color in the terminal via `RARITY_COLORS` mapping to theme colors
