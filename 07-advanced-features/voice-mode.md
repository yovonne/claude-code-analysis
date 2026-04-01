# Voice Mode

## Purpose

Voice Mode enables real-time voice interaction with Claude Code through the `voice_stream` endpoint on claude.ai. It provides a conversational voice interface as an alternative to text-based terminal interaction, requiring Anthropic OAuth authentication.

## Location

`restored-src/src/voice/voiceModeEnabled.ts`

## Key Exports

### Functions

- `isVoiceGrowthBookEnabled()`: Checks the GrowthBook feature flag and kill-switch. Returns `true` unless the `tengu_amber_quartz_disabled` flag is flipped on (emergency off). Default `false` means missing/stale disk cache reads as "not killed."
- `hasVoiceAuth()`: Checks whether the user has a valid Anthropic OAuth token. Returns `true` only when Anthropic auth is enabled AND an access token exists.
- `isVoiceModeEnabled()`: Full runtime check combining auth + GrowthBook kill-switch. The definitive function for deciding whether voice mode should be active.

## Dependencies

### Internal Dependencies

- `../services/analytics/growthbook.js` — `getFeatureValue_CACHED_MAY_BE_STALE()` for GrowthBook flag checks
- `../utils/auth.js` — `getClaudeAIOAuthTokens()` for OAuth token retrieval and `isAnthropicAuthEnabled()` for auth provider check
- `bun:bundle` — `feature()` for build-time feature flag

## Implementation Details

### Three-Layer Gating

Voice mode uses a three-layer gate system:

```
isVoiceModeEnabled()
    ↓
├── hasVoiceAuth()
│   ├── isAnthropicAuthEnabled() → Anthropic OAuth provider active?
│   └── getClaudeAIOAuthTokens() → accessToken exists?
│
└── isVoiceGrowthBookEnabled()
    ├── feature('VOICE_MODE') → Build-time feature flag?
    └── !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
        → Kill-switch NOT active?
```

All three conditions must be true for voice mode to be enabled.

### Auth Layer

`hasVoiceAuth()` performs two checks:

1. **Provider check** — `isAnthropicAuthEnabled()` verifies that the auth provider is Anthropic (not API key, Bedrock, Vertex, or Foundry). Voice mode uses the `voice_stream` endpoint on claude.ai, which is only available with OAuth authentication.

2. **Token check** — `getClaudeAIOAuthTokens()` retrieves the cached OAuth tokens. The function is memoized — the first call spawns the `security` module on macOS (~20-50ms), subsequent calls are cache hits. The memoize clears on token refresh (~once/hour).

Without the provider check, the voice UI would render but `connectVoiceStream` would fail silently when the user isn't logged in.

### GrowthBook Layer

`isVoiceGrowthBookEnabled()` uses a positive ternary pattern for correct bundling behavior:

```typescript
return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
```

Key design decisions:

- **Default `false` for kill-switch**: A missing or stale disk cache reads as "not killed," so fresh installs get voice working immediately without waiting for GrowthBook initialization.
- **Positive ternary pattern**: The `if (!feature(...)) return` pattern does not eliminate inline string literals from external builds. The ternary ensures proper tree-shaking.
- **Cached may be stale**: The GrowthBook value is cached on disk and may be stale, but this is acceptable for a kill-switch — the worst case is a brief window where a disabled feature remains visible.

### Usage Patterns

| Context | Function to Use | Reason |
|---|---|---|
| Command registration | `isVoiceModeEnabled()` | Full check needed |
| Config UI visibility | `isVoiceModeEnabled()` | Full check needed |
| `/voice` command | `isVoiceModeEnabled()` | Full check needed |
| React render paths | `useVoiceEnabled()` | Memoizes auth check, avoids redundant keychain reads |
| VoiceModeNotice | `isVoiceModeEnabled()` | Command-time path, fresh keychain read acceptable |

## Data Flow

```
User launches Claude Code
    ↓
isVoiceModeEnabled() called during startup
    ↓
├── isAnthropicAuthEnabled()
│   └── Check auth provider configuration
│       ↓
├── getClaudeAIOAuthTokens()
│   └── Read OAuth tokens from keychain (memoized)
│       ↓
├── feature('VOICE_MODE')
│   └── Check build-time feature flag
│       ↓
└── getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled')
    └── Check kill-switch flag (cached from disk)
    ↓
All true → Voice mode enabled
Any false → Voice mode disabled
```

## Integration Points

- **`/voice` command** — Entry point for voice mode, uses `isVoiceModeEnabled()` for validation
- **Config tool** — Can enable/disable voice mode, checks `isVoiceModeEnabled()`
- **VoiceModeNotice** — UI notice component, uses `isVoiceModeEnabled()`
- **React components** — Use `useVoiceEnabled()` hook instead of direct `isVoiceModeEnabled()` calls to memoize the auth check
- **connectVoiceStream** — The actual WebSocket connection to claude.ai voice endpoint

## Configuration

| Feature Gate | Purpose |
|---|---|
| `VOICE_MODE` | Build-time feature flag for voice mode |
| `tengu_amber_quartz_disabled` | GrowthBook kill-switch flag (default: false) |

| Auth Requirement | Value |
|---|---|
| Provider | Anthropic OAuth only (not API keys, Bedrock, Vertex, or Foundry) |
| Token | Valid `accessToken` from `getClaudeAIOAuthTokens()` |
| Endpoint | `voice_stream` on claude.ai |

## Error Handling

- **Missing GrowthBook flag**: Defaults to `false` (not killed) — voice mode works on fresh installs
- **Missing OAuth token**: `hasVoiceAuth()` returns `false` — voice mode hidden rather than showing broken UI
- **Wrong auth provider**: `isAnthropicAuthEnabled()` returns `false` — prevents silent `connectVoiceStream` failures
- **Stale cache**: `getFeatureValue_CACHED_MAY_BE_STALE` may return outdated values, but this is acceptable for a kill-switch (worst case: brief visibility window)

## Performance Considerations

- `getClaudeAIOAuthTokens()` is memoized — first call costs ~20-50ms on macOS (spawns `security` module), subsequent calls are O(1) cache hits
- Token refresh happens ~once/hour, causing one cold spawn per refresh cycle
- For React render paths, `useVoiceEnabled()` hook memoizes the auth check to avoid redundant keychain reads on every render
- Command-time paths (`/voice`, Config tool) use `isVoiceModeEnabled()` directly — a fresh keychain read is acceptable at this frequency

## Related Modules

- [Auth](../05-utils/auth.md) — OAuth token management
- [Config Tool](../02-tools/ConfigTool.md) — Voice mode configuration
- [GrowthBook Service](../04-services/) — Feature flag management

## Notes

- Voice mode is exclusively tied to Anthropic OAuth authentication — it cannot work with API keys, AWS Bedrock, Google Vertex, or Azure Foundry
- The kill-switch flag name `tengu_amber_quartz_disabled` follows the internal naming convention for feature toggles
- The `voice_stream` endpoint is specific to claude.ai and is not available through other Anthropic API surfaces
- The three-layer gating (auth + build flag + kill-switch) ensures voice mode can be disabled at multiple levels: per-user (no auth), per-build (feature flag), or globally (kill-switch)
