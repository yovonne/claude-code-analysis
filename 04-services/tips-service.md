# Tips Service

## Purpose
Manages contextual tips displayed during the loading spinner. Tips are contextually relevant based on user behavior, session state, and environment. The system selects the tip that hasn't been shown for the longest time to ensure variety.

## Location
`restored-src/src/services/tips/`

### Key Files
| File | Description |
|------|-------------|
| `tipScheduler.ts` | Tip selection logic — find tip to show, record shown tip |
| `tipRegistry.ts` | Tip definitions with relevance checks and content generation |
| `tipHistory.ts` | Tip show history persistence in global config |

## Key Exports

### Functions (tipScheduler.ts)
- `getTipToShowOnSpinner(context)`: Get the next tip to display during spinner
- `selectTipWithLongestTimeSinceShown(availableTips)`: Select tip shown least recently
- `recordShownTip(tip)`: Record tip as shown (history + analytics)

### Functions (tipRegistry.ts)
- `getRelevantTips(context)`: Filter tips by relevance and cooldown, return eligible tips

### Functions (tipHistory.ts)
- `recordTipShown(tipId)`: Record tip shown at current startup count
- `getSessionsSinceLastShown(tipId)`: Get number of sessions since tip was last shown

### Types
- `Tip`: `{ id, content, cooldownSessions, isRelevant(context) }`
- `TipContext`: Context object passed to relevance checks (bashTools, readFileState, theme)

## Dependencies

### Internal Dependencies
- `settings/settings` — Read spinnerTipsEnabled, spinnerTipsOverride
- `config` — Global config for startup count, feature usage tracking
- `analytics` — Telemetry events
- `growthbook` — A/B test variants for tip content

### External Dependencies
- `chalk` — Terminal coloring for tip content

## Implementation Details

### Tip Selection Algorithm
1. Check if tips are disabled (`spinnerTipsEnabled === false`)
2. Get all relevant tips (filtered by `isRelevant()` and cooldown)
3. Among available tips, select the one with the longest time since last shown
4. If only one tip available, return it directly

### Tip Structure
Each tip has:
- `id`: Unique identifier for tracking
- `content`: Async function returning tip text (can be context-aware with theme colors)
- `cooldownSessions`: Minimum sessions between shows
- `isRelevant(context)`: Async function returning whether tip applies

### Tip Categories
The registry includes ~50 tips covering:
- **New user onboarding**: new-user-warmup, plan-mode, permissions
- **Feature discovery**: memory command, theme command, status line, prompt queue
- **Terminal integration**: terminal-setup, shift-enter, vscode-command-install
- **Multi-session**: git-worktrees, color-when-multi-clauding
- **Plugin marketplace**: frontend-design-plugin, vercel-plugin
- **Platform-specific**: powershell-tool-env, colorterm-truecolor, paste-images-mac
- **Conversation management**: continue, rename-conversation, double-esc
- **Advanced features**: custom-commands, custom-agents, agent-flag
- **Upsells**: desktop-app, desktop-shortcut, web-app, mobile-app
- **Growth experiments**: effort-high-nudge, subagent-fanout-nudge, loop-command-nudge
- **Referral/credits**: guest-passes, overage-credit
- **Internal-only**: important-claudemd, skillify (ANT-only)

### Custom Tips
Users can override default tips via `spinnerTipsOverride` in settings:
- `tips`: Array of custom tip strings
- `excludeDefault`: If true and custom tips exist, skip built-in tips entirely

### Cooldown System
- Tips track last shown session via `numStartups` in global config
- Each tip has a `cooldownSessions` value (0-30 sessions)
- Tips are filtered out if shown within their cooldown period
- Selection favors tips not shown for the longest time

### Context-Aware Content
Some tips generate dynamic content based on context:
- Theme colors via `color('suggestion', ctx.theme)`
- Plugin installation status
- A/B test variants via GrowthBook flags

## Data Flow

```
Spinner render
    ↓
getTipToShowOnSpinner(context)
    ↓
Check spinnerTipsEnabled setting
    ↓
getRelevantTips(context)
    ↓
Filter: isRelevant() for each tip
    ↓
Filter: getSessionsSinceLastShown() >= cooldownSessions
    ↓
selectTipWithLongestTimeSinceShown()
    ↓
Return tip or undefined
    ↓
recordShownTip(tip) — update history + log event
```

## Integration Points
- **Spinner UI**: Tips displayed during loading spinner
- **Global Config**: Tip history persisted as `tipsHistory` map
- **Settings**: `spinnerTipsEnabled` and `spinnerTipsOverride` control behavior
- **Analytics**: `tengu_tip_shown` event logged when tip displayed

## Configuration
- Settings: `spinnerTipsEnabled` (default true), `spinnerTipsOverride` (custom tips)
- Global config: `tipsHistory` map (tipId → numStartups when shown)
- Cooldown range: 0-30 sessions per tip

## Error Handling
- Relevance check failures caught and logged, tip treated as not relevant
- Config read failures return false for relevance
- Tip content generation failures would surface as empty strings

## Testing
- `getSessionsSinceLastShown()` returns Infinity for never-shown tips
- History deduplication: same tip at same startup count is not re-recorded

## Related Modules
- [Analytics](./analytics-service.md) — tip shown telemetry
- [Settings](./settings-commands.md) — spinner tips configuration

## Notes
- Tips are displayed during spinner loading to educate users without interrupting workflow
- The "longest time since shown" selection ensures even coverage across all tips
- Internal-only tips are filtered by `USER_TYPE === 'ant'` environment variable
- A/B test variants allow testing different tip copy without code changes
- Custom tips can completely replace built-in tips with `excludeDefault: true`
