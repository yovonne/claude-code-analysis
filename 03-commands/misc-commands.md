# Miscellaneous Commands

## Overview

Miscellaneous commands cover utility, promotional, configuration, and feature-specific functionality that does not fit into other command categories. These include side questions, session tagging, installation diagnostics, editor mode toggles, voice mode, terminal setup, and various promotional or feature-gated commands.

---

## `/btw`

**Purpose:** Ask a quick side question without interrupting the main conversation flow.

**Source:** `restored-src/src/commands/btw/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration as local-jsx type |
| `btw.tsx` | Side question UI with streaming response and scroll support |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'btw',
  description: 'Ask a quick side question without interrupting the main conversation',
  immediate: true,
  argumentHint: '<question>',
  load: () => import('./btw.js'),
}
```

**Behavior:**
- Renders a scrollable dialog that sends the question as a side question via `runSideQuestion()`
- Uses cached system prompt parameters (`getLastCacheSafeParams`) to achieve prompt cache hits on the forked request
- Displays a spinner while the answer streams in, then renders the response as Markdown
- Tracks usage count in global config (`btwUseCount`)
- Dismissed with Escape, Enter, Space, or Ctrl+C/D
- Scrollable with Up/Down arrows or Ctrl+P/Ctrl+N

**Cache-safe parameter building (`buildCacheSafeParams`):**
1. Prefers `getLastCacheSafeParams()` — the exact systemPrompt/userContext/systemContext bytes the main thread last sent, ensuring byte-identical prefix and cache hit
2. Falls back to rebuilding from scratch via `getSystemPrompt()`, `getUserContext()`, `getSystemContext()` — may miss cache if main loop applied buildEffectiveSystemPrompt extras (--agent, --system-prompt, etc.)
3. Strips in-progress assistant messages (those with `stop_reason === null`) before forking
4. Uses `getMessagesAfterCompactBoundary()` to limit fork context

---

## `/summary`

**Purpose:** Stub command (disabled).

**Source:** `restored-src/src/commands/summary/`

| File | Purpose |
|------|---------|
| `index.js` | Stub registration — always disabled and hidden |

**Registration:**
```javascript
{ isEnabled: () => false, isHidden: true, name: 'stub' }
```

This command is a placeholder with no implementation.

---

## `/think-back`

**Purpose:** Generate and play a personalized "2025 Claude Code Year in Review" ASCII animation.

**Source:** `restored-src/src/commands/thinkback/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with feature gate check |
| `thinkback.tsx` | Full installation, menu, and animation flow UI |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'think-back',
  description: 'Your 2025 Claude Code Year in Review',
  isEnabled: () => checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback'),
  load: () => import('./thinkback.js'),
}
```

**Feature gate:** Controlled by `tengu_thinkback` Statsig feature gate.

**Installation flow (`ThinkbackInstaller`):**
1. Checks if the marketplace is installed (`loadKnownMarketplacesConfig`)
2. If marketplace missing, installs it via `addMarketplaceSource()` using the appropriate repo (`anthropics/claude-plugins-official` for external users, `anthropics/claude-code-marketplace` for internal)
3. If marketplace exists but plugin missing, refreshes marketplace via `refreshMarketplace()`
4. Installs the plugin via `installSelectedPlugins([pluginId])`
5. If plugin is installed but disabled, enables it via `enablePluginOp()`
6. Clears all caches after each step

**Menu flow (`ThinkbackMenu`):**
- Shows a dialog titled "Think Back on 2025 with Claude Code"
- If no animation exists yet, shows a single "Let's go!" option to generate
- If animation exists, offers four actions:

| Action | Value | Description |
|--------|-------|-------------|
| Play animation | `play` | Watch the year in review |
| Edit content | `edit` | Modify the animation |
| Fix errors | `fix` | Fix validation or rendering issues |
| Regenerate | `regenerate` | Create a new animation from scratch |

**Generative actions:**
- `edit` → Sends prompt to use Skill tool with `mode=edit`
- `fix` → Sends prompt to use Skill tool with `mode=fix` (runs validator, fixes errors)
- `regenerate` → Sends prompt to use Skill tool with `mode=regenerate` (deletes existing, starts fresh)

**Animation playback (`playAnimation`):**
1. Locates `year_in_review.js` and `player.js` in the installed plugin's skill directory
2. Validates both files exist before proceeding
3. Takes over the terminal via `inkInstance.enterAlternateScreen()`
4. Runs `node player.js` with inherited stdio
5. Restores terminal via `inkInstance.exitAlternateScreen()`
6. Opens `year_in_review.html` in the system browser for video download

---

## `/thinkback-play`

**Purpose:** Hidden command that plays the thinkback animation directly. Called by the thinkback skill after generation is complete.

**Source:** `restored-src/src/commands/thinkback-play/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration — hidden, feature-gated |
| `thinkback-play.ts` | Locates plugin and invokes `playAnimation()` |

**Registration:**
```typescript
{
  type: 'local',
  name: 'thinkback-play',
  description: 'Play the thinkback animation',
  isEnabled: () => checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback'),
  isHidden: true,
  supportsNonInteractive: false,
  load: () => import('./thinkback-play.js'),
}
```

**Behavior:**
- Loads installed plugins via `loadInstalledPluginsV2()`
- Resolves the thinkback plugin installation path
- Calls `playAnimation()` from the thinkback module with the skill directory
- Returns the animation result message

---

## `/stickers`

**Purpose:** Open the Claude Code stickers ordering page in the browser.

**Source:** `restored-src/src/commands/stickers/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration as local type |
| `stickers.ts` | Opens the StickerMule URL in browser |

**Registration:**
```typescript
{
  type: 'local',
  name: 'stickers',
  description: 'Order Claude Code stickers',
  supportsNonInteractive: false,
  load: () => import('./stickers.js'),
}
```

**Behavior:**
- Opens `https://www.stickermule.com/claudecode` via `openBrowser()`
- Shows success message or fallback URL if browser launch fails

---

## `/feedback`

**Purpose:** Submit feedback or report bugs about Claude Code.

**Source:** `restored-src/src/commands/feedback/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with availability checks |
| `feedback.tsx` | Renders the Feedback component |

**Registration:**
```typescript
{
  aliases: ['bug'],
  type: 'local-jsx',
  name: 'feedback',
  description: 'Submit feedback about Claude Code',
  argumentHint: '[report]',
  isEnabled: () => !(
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isEnvTruthy(process.env.DISABLE_FEEDBACK_COMMAND) ||
    isEnvTruthy(process.env.DISABLE_BUG_COMMAND) ||
    isEssentialTrafficOnly() ||
    process.env.USER_TYPE === 'ant' ||
    !isPolicyAllowed('allow_product_feedback')
  ),
  load: () => import('./feedback.js'),
}
```

**Availability conditions (disabled when any are true):**
- Using Bedrock, Vertex, or Foundry providers
- `DISABLE_FEEDBACK_COMMAND` or `DISABLE_BUG_COMMAND` env vars set
- Essential traffic only mode active
- Internal Anthropic user (`USER_TYPE === 'ant'`)
- Policy disallows product feedback

**Behavior:**
- Alias `/bug` provides the same functionality
- Accepts an optional `[report]` argument as initial description
- Renders the `Feedback` component with current messages and abort signal
- Shared `renderFeedbackComponent()` function is also used by other parts of the codebase

---

## `/release-notes`

**Purpose:** View the latest Claude Code release notes and changelog.

**Source:** `restored-src/src/commands/release-notes/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration as local type |
| `release-notes.ts` | Fetches and formats changelog data |

**Registration:**
```typescript
{
  description: 'View release notes',
  name: 'release-notes',
  type: 'local',
  supportsNonInteractive: true,
  load: () => import('./release-notes.js'),
}
```

**Behavior:**
1. Attempts to fetch fresh changelog with a 500ms timeout via `fetchAndStoreChangelog()`
2. If fresh fetch succeeds, formats and displays the new notes
3. If fetch fails or times out, falls back to cached notes via `getStoredChangelog()`
4. If no cached notes available, displays the changelog URL

**Format:**
```
Version X.Y.Z:
· Change item 1
· Change item 2

Version X.Y.Z:
· Change item 1
```

---

## `/passes`

**Purpose:** Share a free week of Claude Code with friends and earn extra usage passes.

**Source:** `restored-src/src/commands/passes/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with dynamic description and visibility |
| `passes.tsx` | Passes UI with first-visit tracking |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'passes',
  get description() {
    const reward = getCachedReferrerReward()
    if (reward) {
      return 'Share a free week of Claude Code with friends and earn extra usage'
    }
    return 'Share a free week of Claude Code with friends'
  },
  get isHidden() {
    const { eligible, hasCache } = checkCachedPassesEligibility()
    return !eligible || !hasCache
  },
  load: () => import('./passes.js'),
}
```

**Visibility:** Hidden when user is not eligible for the passes program or when eligibility cache is unavailable.

**Behavior:**
- On first visit, records `hasVisitedPasses: true` and `passesLastSeenRemaining` in global config
- This stops showing the passes upsell prompt
- Logs `tengu_guest_passes_visited` analytics event with `is_first_visit` flag
- Renders the `Passes` component

---

## `/upgrade`

**Purpose:** Upgrade to Claude Max subscription for higher rate limits and more Opus access.

**Source:** `restored-src/src/commands/upgrade/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with availability and enable checks |
| `upgrade.tsx` | Upgrade flow with Max plan check and login redirect |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'upgrade',
  description: 'Upgrade to Max for higher rate limits and more Opus',
  availability: ['claude-ai'],
  isEnabled: () =>
    !isEnvTruthy(process.env.DISABLE_UPGRADE_COMMAND) &&
    getSubscriptionType() !== 'enterprise',
  load: () => import('./upgrade.js'),
}
```

**Availability:** Only available for `claude-ai` auth type. Disabled for enterprise users and when `DISABLE_UPGRADE_COMMAND` env var is set.

**Behavior:**
1. Checks if user is already on the highest Max plan (20x rate limit tier)
   - Uses cached tokens or fetches OAuth profile to determine `subscriptionType` and `rateLimitTier`
   - If already on max plan, informs user and suggests `/login` for API-billed account
2. Opens `https://claude.ai/upgrade/max` in the browser
3. Renders the `Login` component to allow re-authentication after upgrade
4. On successful login, calls `context.onChangeAPIKey()` to refresh the session

---

## `/doctor`

**Purpose:** Diagnose and verify the Claude Code installation and settings.

**Source:** `restored-src/src/commands/doctor/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with enable check |
| `doctor.tsx` | Renders the Doctor diagnostic screen |

**Registration:**
```typescript
{
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

**Behavior:**
- Renders the `Doctor` screen component which performs diagnostic checks
- Can be disabled via `DISABLE_DOCTOR_COMMAND` environment variable

---

## `/keybindings`

**Purpose:** Open or create the user's keybindings configuration file in the editor.

**Source:** `restored-src/src/commands/keybindings/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with customization enable check |
| `keybindings.ts` | Creates template file if needed and opens in editor |

**Registration:**
```typescript
{
  name: 'keybindings',
  description: 'Open or create your keybindings configuration file',
  isEnabled: () => isKeybindingCustomizationEnabled(),
  supportsNonInteractive: false,
  type: 'local',
  load: () => import('./keybindings.js'),
}
```

**Availability:** Only enabled when keybinding customization is enabled (preview feature).

**Behavior:**
1. Resolves the keybindings file path via `getKeybindingsPath()`
2. Creates the parent directory if needed
3. Attempts to create the file with a template via `generateKeybindingsTemplate()` using exclusive write flag (`wx`) — fails silently if file already exists
4. Opens the file in the user's configured editor via `editFileInEditor()`
5. Reports whether the file was created or opened, and any editor errors

---

## `/vim`

**Purpose:** Toggle between Vim and Normal (readline) editing modes.

**Source:** `restored-src/src/commands/vim/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration as local type |
| `vim.ts` | Toggles editor mode in global config |

**Registration:**
```typescript
{
  name: 'vim',
  description: 'Toggle between Vim and Normal editing modes',
  supportsNonInteractive: false,
  type: 'local',
  load: () => import('./vim.js'),
}
```

**Behavior:**
1. Reads current `editorMode` from global config (defaults to `'normal'`)
2. Handles backward compatibility: treats legacy `'emacs'` mode as `'normal'`
3. Toggles between `'normal'` and `'vim'`
4. Saves the new mode to global config
5. Logs `tengu_editor_mode_changed` analytics event with new mode and source
6. Displays confirmation message:
   - Vim mode: "Use Escape key to toggle between INSERT and NORMAL modes."
   - Normal mode: "Using standard (readline) keyboard bindings."

---

## `/voice`

**Purpose:** Toggle voice mode for push-to-talk speech input.

**Source:** `restored-src/src/commands/voice/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with feature gate and visibility checks |
| `voice.ts` | Full voice mode toggle with pre-flight checks |

**Registration:**
```typescript
{
  type: 'local',
  name: 'voice',
  description: 'Toggle voice mode',
  availability: ['claude-ai'],
  isEnabled: () => isVoiceGrowthBookEnabled(),
  get isHidden() {
    return !isVoiceModeEnabled()
  },
  supportsNonInteractive: false,
  load: () => import('./voice.js'),
}
```

**Availability:** Only for `claude-ai` auth type and when voice feature is enabled via GrowthBook.

**Toggle OFF behavior:**
- Sets `voiceEnabled: false` in user settings
- Notifies settings change detector
- Logs `tengu_voice_toggled` with `enabled: false`

**Toggle ON behavior (pre-flight checks):**
1. Checks voice mode is enabled and Anthropic auth is active
2. Checks microphone recording availability via `checkRecordingAvailability()`
3. Checks voice stream API key availability via `isVoiceStreamAvailable()`
4. Checks for audio recording tools via `checkVoiceDependencies()` — suggests install command if missing (e.g., SoX)
5. Probes microphone permission via `requestMicrophonePermission()` — triggers OS permission dialog preemptively
   - Provides platform-specific guidance if denied (Windows: Settings > Privacy > Microphone, macOS: System Settings > Privacy & Security > Microphone, Linux: system audio settings)
6. If all checks pass, sets `voiceEnabled: true` and displays the push-to-talk shortcut key
7. Shows dictation language hint (up to 2 times per language) or fallback warning if the configured language is not supported

---

## `/tag`

**Purpose:** Toggle a searchable tag on the current session.

**Source:** `restored-src/src/commands/tag/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration (internal-only) |
| `tag.tsx` | Tag toggle UI with confirmation dialog for removal |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'tag',
  description: 'Toggle a searchable tag on the current session',
  isEnabled: () => process.env.USER_TYPE === 'ant',
  argumentHint: '<tag-name>',
  load: () => import('./tag.js'),
}
```

**Availability:** Internal Anthropic users only (`USER_TYPE === 'ant'`).

**Behavior:**
- Tag names are sanitized via `recursivelySanitizeUnicode()` and trimmed
- If the tag matches the current session's tag, prompts for confirmation to remove
- If the tag differs, adds or replaces the tag on the session
- Tags are stored via `saveTag()` in the session transcript
- Tags appear after the branch name in `/resume` and are searchable with `/`
- Shows help text for `--help`, `--info`, or empty arguments

**Confirmation dialog (`ConfirmRemoveTag`):**
- Displays current tag name
- Offers "Yes, remove tag" or "No, keep tag" options

**Analytics events:**
- `tengu_tag_command_add` — when adding a tag (with `is_replacing` flag)
- `tengu_tag_command_remove_prompt` — when removal is triggered
- `tengu_tag_command_remove_confirmed` — when removal is confirmed
- `tengu_tag_command_remove_cancelled` — when removal is cancelled

---

## `/rewind`

**Purpose:** Restore the code and/or conversation to a previous point.

**Source:** `restored-src/src/commands/rewind/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with `checkpoint` alias |
| `rewind.ts` | Opens the message selector UI |

**Registration:**
```typescript
{
  description: 'Restore the code and/or conversation to a previous point',
  name: 'rewind',
  aliases: ['checkpoint'],
  argumentHint: '',
  type: 'local',
  supportsNonInteractive: false,
  load: () => import('./rewind.js'),
}
```

**Behavior:**
- Invokes `context.openMessageSelector()` to display the message selection UI
- Returns a skip result to avoid appending any output messages

---

## `/onboarding`

**Purpose:** Stub command (disabled).

**Source:** `restored-src/src/commands/onboarding/`

| File | Purpose |
|------|---------|
| `index.js` | Stub registration — always disabled and hidden |

**Registration:**
```javascript
{ isEnabled: () => false, isHidden: true, name: 'stub' }
```

This command is a placeholder with no implementation.

---

## `/terminal-setup`

**Purpose:** Configure terminal keybindings for convenient multi-line prompt input.

**Source:** `restored-src/src/commands/terminalSetup/`

| File | Purpose |
|------|---------|
| `index.ts` | Command registration with terminal detection |
| `terminalSetup.tsx` | Terminal-specific keybinding installation logic |

**Registration:**
```typescript
{
  type: 'local-jsx',
  name: 'terminal-setup',
  description:
    env.terminal === 'Apple_Terminal'
      ? 'Enable Option+Enter key binding for newlines and visual bell'
      : 'Install Shift+Enter key binding for newlines',
  isHidden: env.terminal !== null && env.terminal in NATIVE_CSIU_TERMINALS,
  load: () => import('./terminalSetup.js'),
}
```

**Auto-hidden terminals (native CSI u / Kitty protocol support):**

| Terminal | Display Name |
|----------|-------------|
| `ghostty` | Ghostty |
| `kitty` | Kitty |
| `iTerm.app` | iTerm2 |
| `WezTerm` | WezTerm |
| `WarpTerminal` | Warp |

**Supported terminals for setup:**

| Terminal | Configuration |
|----------|--------------|
| `Apple_Terminal` | Enables Option as Meta key and visual bell via PlistBuddy |
| `vscode` | Installs Shift+Enter keybinding in VSCode keybindings.json |
| `cursor` | Installs Shift+Enter keybinding in Cursor keybindings.json |
| `windsurf` | Installs Shift+Enter keybinding in Windsurf keybindings.json |
| `alacritty` | Appends Shift+Return binding to alacritty.toml |
| `zed` | Adds Shift+Enter binding to Zed keymap.json |

**VSCode/Cursor/Windsurf keybinding installation:**
1. Detects VSCode Remote SSH sessions — if detected, provides manual instructions instead
2. Resolves platform-specific config directory (Windows: `AppData/Roaming`, macOS: `Library/Application Support`, Linux: `.config`)
3. Reads existing `keybindings.json` (preserving JSONC comments)
4. Creates a timestamped backup before modifying
5. Checks for existing Shift+Enter binding to avoid duplicates
6. Adds the keybinding: `shift+enter` → `workbench.action.terminal.sendSequence` with `\u001b\r`
7. Uses `addItemToJSONCArray()` to preserve formatting and comments

**Apple Terminal setup:**
1. Creates a backup of Terminal preferences via `backupTerminalPreferences()`
2. Reads default and startup window profiles via `defaults read`
3. Enables `useOptionAsMetaKey` for each profile via PlistBuddy
4. Disables audio bell (switches to visual bell) for each profile
5. Flushes preferences cache via `killall cfprefsd`
6. On failure, attempts to restore from backup

**Alacritty setup:**
1. Checks config paths in order: `$XDG_CONFIG_HOME/alacritty/alacritty.toml`, `~/.config/alacritty/alacritty.toml`, Windows `%APPDATA%/alacritty/alacritty.toml`
2. Checks for existing Shift+Return binding
3. Creates timestamped backup if config exists
4. Appends the TOML keybinding block

**Zed setup:**
1. Uses `~/.config/zed/keymap.json`
2. Parses existing keymap (defaults to empty array)
3. Checks for existing `shift-enter` binding
4. Creates timestamped backup if file exists
5. Pushes a new binding: `shift-enter` → `['terminal::SendText', '\u001b\r']` in Terminal context

**Shell completion:** Installed for internal (`ant`) users only via `setupShellCompletion()`.

**Config flags set after successful setup:**
- `shiftEnterKeyBindingInstalled: true` — for VSCode/Cursor/Windsurf/Alacritty/Zed
- `optionAsMetaKeyInstalled: true` — for Apple Terminal

---

## `/break-cache`

**Purpose:** Stub command (disabled).

**Source:** `restored-src/src/commands/break-cache/`

| File | Purpose |
|------|---------|
| `index.js` | Stub registration — always disabled and hidden |

**Registration:**
```javascript
{ isEnabled: () => false, isHidden: true, name: 'stub' }
```

This command is a placeholder with no implementation.

---

## `/ctx-viz`

**Purpose:** Stub command (disabled).

**Source:** `restored-src/src/commands/ctx_viz/`

| File | Purpose |
|------|---------|
| `index.js` | Stub registration — always disabled and hidden |

**Registration:**
```javascript
{ isEnabled: () => false, isHidden: true, name: 'stub' }
```

This command is a placeholder with no implementation.

---

## Summary

| Command | Type | Visibility | Key Feature |
|---------|------|-----------|-------------|
| `/btw` | local-jsx | Always visible | Side questions with prompt cache reuse |
| `/summary` | stub | Hidden | Placeholder |
| `/think-back` | local-jsx | Feature-gated | 2025 Year in Review animation |
| `/thinkback-play` | local | Hidden | Animation playback helper |
| `/stickers` | local | Always visible | Opens sticker ordering page |
| `/feedback` | local-jsx | Conditional | Bug reports and feedback (alias: `/bug`) |
| `/release-notes` | local | Always visible | Changelog with 500ms fresh fetch |
| `/passes` | local-jsx | Conditional | Referral passes program |
| `/upgrade` | local-jsx | Conditional | Upgrade to Max subscription |
| `/doctor` | local-jsx | Conditional | Installation diagnostics |
| `/keybindings` | local | Conditional | Open keybindings config file |
| `/vim` | local | Always visible | Toggle Vim/Normal editing mode |
| `/voice` | local | Feature-gated | Toggle push-to-talk voice mode |
| `/tag` | local-jsx | Internal only | Session tagging |
| `/rewind` | local | Always visible | Restore to previous point (alias: `/checkpoint`) |
| `/onboarding` | stub | Hidden | Placeholder |
| `/terminal-setup` | local-jsx | Conditional | Terminal keybinding configuration |
| `/break-cache` | stub | Hidden | Placeholder |
| `/ctx-viz` | stub | Hidden | Placeholder |
