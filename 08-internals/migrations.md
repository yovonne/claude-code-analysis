# Migrations

## Purpose

The migrations module contains one-shot startup functions that migrate user configuration from old formats to new formats. Each migration is idempotent — it checks whether it has already run (via completion flags in global config or settings) and skips if so. Migrations handle model alias changes, settings location moves, and feature flag resets.

## Location

`restored-src/src/migrations/`

## Key Files & Exports

### Model Migrations

#### `migrateSonnet45ToSonnet46.ts`

Migrates Pro/Max/Team Premium first-party users from explicit Sonnet 4.5 model strings to the `sonnet` alias (now resolves to Sonnet 4.6).

- **Triggers**: `getAPIProvider() === 'firstParty'` AND subscriber status
- **Migrates**: `claude-sonnet-4-5-20250929`, `sonnet-4-5-20250929`, and `[1m]` variants → `sonnet` / `sonnet[1m]`
- **Guard**: `sonnet45To46MigrationTimestamp` in global config
- **Notification**: Sets timestamp for `useModelMigrationNotifications` to display "Model updated to Sonnet 4.6"

#### `migrateOpusToOpus1m.ts`

Migrates users with `opus` pinned in settings to `opus[1m]` when eligible for merged Opus 1M experience.

- **Triggers**: `isOpus1mMergeEnabled()` AND `userSettings.model === 'opus'`
- **Skips**: Pro subscribers (retain separate Opus/Opus 1M options), 3P users
- **Smart**: Sets `undefined` if migrated value equals default (avoids unnecessary settings writes)

#### `migrateLegacyOpusToCurrent.ts`

Migrates first-party users off explicit Opus 4.0/4.1 model strings to the `opus` alias.

- **Triggers**: `getAPIProvider() === 'firstParty'` AND `isLegacyModelRemapEnabled()`
- **Migrates**: `claude-opus-4-20250514`, `claude-opus-4-1-20250805`, etc. → `opus`
- **Notification**: Sets `legacyOpusMigrationTimestamp` for one-time REPL notification

#### `migrateSonnet1mToSonnet45.ts`

Migrates `sonnet[1m]` users to explicit `sonnet-4-5-20250929[1m]` (needed because Sonnet 4.6 1M was offered to a different group).

- **Triggers**: `!config.sonnet1m45MigrationComplete`
- **Also migrates**: In-memory `getMainLoopModelOverride()` if set
- **Guard**: `sonnet1m45MigrationComplete` flag in global config

#### `migrateFennecToOpus.ts`

Migrates removed fennec model aliases to Opus 4.6 aliases (Ant-internal only).

- **Triggers**: `USER_TYPE === 'ant'`
- **Migrates**: `fennec-latest` → `opus`, `fennec-latest[1m]` → `opus[1m]`, `fennec-fast-latest` / `opus-4-5-fast` → `opus[1m]` + fast mode

### Settings Location Migrations

#### `migrateAutoUpdatesToSettings.ts`

Moves auto-updates preference from global config to settings.json env var.

- **Migrates**: `autoUpdates: false` (user preference only, not native protection) → `DISABLE_AUTOUPDATER: '1'` in settings env
- **Cleanup**: Removes `autoUpdates` and `autoUpdatesProtectedForNative` from global config

#### `migrateBypassPermissionsAcceptedToSettings.ts`

Moves bypass permissions acceptance from global config to settings.json.

- **Migrates**: `bypassPermissionsModeAccepted` → `skipDangerousModePermissionPrompt`
- **Cleanup**: Removes old key from global config

#### `migrateEnableAllProjectMcpServersToSettings.ts`

Moves MCP server approval fields from project config to local settings.

- **Migrates**: `enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`
- **Merges**: Server lists with existing local settings (avoids duplicates)
- **Cleanup**: Removes old fields from project config

#### `migrateReplBridgeEnabledToRemoteControlAtStartup.ts`

Renames `replBridgeEnabled` config key to `remoteControlAtStartup`.

- **Simple rename**: Copies value, deletes old key
- **Guard**: Only acts when old key exists and new key doesn't

### Feature Reset Migrations

#### `resetAutoModeOptInForDefaultOffer.ts`

One-shot migration to re-surface the auto mode dialog with the new "make it my default mode" option.

- **Triggers**: `TRANSCRIPT_CLASSIFIER` feature flag AND `getAutoModeEnabledState() === 'enabled'`
- **Resets**: `skipAutoPermissionPrompt` when `defaultMode !== 'auto'`
- **Guard**: `hasResetAutoModeOptInForDefaultOffer` in global config

#### `resetProToOpusDefault.ts`

Auto-migrates Pro subscribers to Opus 4.5 default on firstParty.

- **Triggers**: `apiProvider === 'firstParty'` AND `isProSubscriber()`
- **Notification**: Sets `opusProMigrationTimestamp` for users without custom model setting
- **Guard**: `opusProMigrationComplete` flag

## Dependencies

### Internal Dependencies

- `bootstrap/state.js` — Model override access (`migrateSonnet1mToSonnet45`)
- `services/analytics/index.js` — `logEvent` for all migrations
- `utils/config.js` — `getGlobalConfig`, `saveGlobalConfig`, `getCurrentProjectConfig`
- `utils/settings/settings.js` — `getSettingsForSource`, `updateSettingsForSource`
- `utils/model/model.js` — Model parsing and feature flags
- `utils/model/providers.js` — `getAPIProvider()` for first-party checks
- `utils/auth.js` — Subscriber status checks

### External Dependencies

- `bun:bundle` — `feature()` for feature flag checks

## Implementation Details

### Idempotency Patterns

Each migration uses one of these patterns to ensure it runs exactly once:

1. **Completion flag**: Check/set a boolean in global config (e.g., `sonnet1m45MigrationComplete`)
2. **Timestamp check**: Check if a timestamp field exists (e.g., `legacyOpusMigrationTimestamp`)
3. **Value comparison**: Check if the current value already matches the target (e.g., model string comparison)
4. **Source-specific**: Only read/write `userSettings` to avoid promoting project-scoped values to global

### Analytics

Every migration logs a `tengu_*` analytics event:
- `tengu_sonnet45_to_46_migration`
- `tengu_opus_to_opus1m_migration`
- `tengu_legacy_opus_migration`
- `tengu_migrate_autoupdates_to_settings`
- `tengu_migrate_bypass_permissions_accepted`
- `tengu_migrate_mcp_approval_fields_success/error`
- `tengu_migrate_reset_auto_opt_in_for_default_offer`
- `tengu_reset_pro_to_opus_default`

### Error Handling

Migrations that could fail (settings writes) wrap in try/catch and log errors without throwing — a migration failure should not prevent app startup.

## Data Flow

```
main.tsx startup
    ↓
Run all migrations (order matters for model migrations)
    ↓
Each migration checks its guard
    ↓
If guard passes: migrate → log event → set guard
    ↓
App continues with migrated config
```

## Related Modules

- [Bootstrap](./bootstrap.md) — Migrations read/write global config and settings
- [Hooks - Notifications](./hooks-system.md) — `useModelMigrationNotifications` displays migration notifications
- [Schemas](./schemas.md) — Settings schemas validate migrated data
