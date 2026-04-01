# 迁移

## 目的

迁移模块包含一次性的启动函数，用于将用户配置从旧格式迁移到新格式。每个迁移都是幂等的——它通过全局配置或设置中的完成标志检查是否已运行，如果是则跳过。迁移处理模型别名更改、设置位置移动和功能标志重置。

## 位置

`restored-src/src/migrations/`

## 关键文件和导出

### 模型迁移

#### `migrateSonnet45ToSonnet46.ts`

将 Pro/Max/Team Premium 第一方用户从显式 Sonnet 4.5 模型字符串迁移到 `sonnet` 别名（现在解析为 Sonnet 4.6）。

- **触发**: `getAPIProvider() === 'firstParty'` AND 订阅者状态
- **迁移**: `claude-sonnet-4-5-20250929`、`sonnet-4-5-20250929` 和 `[1m]` 变体 → `sonnet` / `sonnet[1m]`
- **守卫**: 全局配置中的 `sonnet45To46MigrationTimestamp`
- **通知**: 设置时间戳用于 `useModelMigrationNotifications` 显示"模型已更新到 Sonnet 4.6"

#### `migrateOpusToOpus1m.ts`

当有资格获得合并 Opus 1M 体验时，将 settings 中固定了 `opus` 的用户迁移到 `opus[1m]`。

- **触发**: `isOpus1mMergeEnabled()` AND `userSettings.model === 'opus'`
- **跳过**: Pro 订阅者（保留单独的 Opus/Opus 1M 选项）、3P 用户
- **智能**: 如果迁移值等于默认值则设置 `undefined`（避免不必要的设置写入）

#### `migrateLegacyOpusToCurrent.ts`

将第一方用户从显式 Opus 4.0/4.1 模型字符串迁移到 `opus` 别名。

- **触发**: `getAPIProvider() === 'firstParty'` AND `isLegacyModelRemapEnabled()`
- **迁移**: `claude-opus-4-20250514`、`claude-opus-4-1-20250805` 等 → `opus`
- **通知**: 设置 `legacyOpusMigrationTimestamp` 用于一次性 REPL 通知

#### `migrateSonnet1mToSonnet45.ts`

将 `sonnet[1m]` 用户迁移到显式 `sonnet-4-5-20250929[1m]`（因为 Sonnet 4.6 1M 提供给不同的群体）。

- **触发**: `!config.sonnet1m45MigrationComplete`
- **也迁移**: 如果设置则内存中的 `getMainLoopModelOverride()`
- **守卫**: 全局配置中的 `sonnet1m45MigrationComplete` 标志

#### `migrateFennecToOpus.ts`

将已移除的 fennec 模型别名迁移到 Opus 4.6 别名（仅 Ant 内部）。

- **触发**: `USER_TYPE === 'ant'`
- **迁移**: `fennec-latest` → `opus`、`fennec-latest[1m]` → `opus[1m]`、`fennec-fast-latest` / `opus-4-5-fast` → `opus[1m]` + 快速模式

### 设置位置迁移

#### `migrateAutoUpdatesToSettings.ts`

将自动更新偏好从全局配置移动到 settings.json 环境变量。

- **迁移**: `autoUpdates: false`（仅用户偏好，非原生保护）→ `DISABLE_AUTOUPDATER: '1'` 在 settings env 中
- **清理**: 从全局配置移除 `autoUpdates` 和 `autoUpdatesProtectedForNative`

#### `migrateBypassPermissionsAcceptedToSettings.ts`

将绕过权限接受从全局配置移动到 settings.json。

- **迁移**: `bypassPermissionsModeAccepted` → `skipDangerousModePermissionPrompt`
- **清理**: 从全局配置移除旧键

#### `migrateEnableAllProjectMcpServersToSettings.ts`

将 MCP 服务器批准字段从项目配置移动到本地设置。

- **迁移**: `enableAllProjectMcpServers`、`enabledMcpjsonServers`、`disabledMcpjsonServers`
- **合并**: 与现有本地设置合并服务器列表（避免重复）
- **清理**: 从项目配置移除旧字段

#### `migrateReplBridgeEnabledToRemoteControlAtStartup.ts`

将 `replBridgeEnabled` 配置键重命名为 `remoteControlAtStartup`。

- **简单重命名**: 复制值，删除旧键
- **守卫**: 仅在旧键存在且新键不存在时行动

### 功能重置迁移

#### `resetAutoModeOptInForDefaultOffer.ts`

一次性迁移，重新显示带有新的"使其成为我的默认模式"选项的自动模式对话框。

- **触发**: `TRANSCRIPT_CLASSIFIER` 功能标志 AND `getAutoModeEnabledState() === 'enabled'`
- **重置**: 当 `defaultMode !== 'auto'` 时的 `skipAutoPermissionPrompt`
- **守卫**: 全局配置中的 `hasResetAutoModeOptInForDefaultOffer`

#### `resetProToOpusDefault.ts`

在第一方上自动将 Pro 订阅者迁移到 Opus 4.5 默认值。

- **触发**: `apiProvider === 'firstParty'` AND `isProSubscriber()`
- **通知**: 为没有自定义模型设置的用户设置 `opusProMigrationTimestamp`
- **守卫**: `opusProMigrationComplete` 标志

## 依赖

### 内部依赖

- `bootstrap/state.js` — 模型覆盖访问（`migrateSonnet1mToSonnet45`）
- `services/analytics/index.js` — 所有迁移的 `logEvent`
- `utils/config.js` — `getGlobalConfig`、`saveGlobalConfig`、`getCurrentProjectConfig`
- `utils/settings/settings.js` — `getSettingsForSource`、`updateSettingsForSource`
- `utils/model/model.js` — 模型解析和功能标志
- `utils/model/providers.js` — 用于第一方检查的 `getAPIProvider()`
- `utils/auth.js` — 订阅者状态检查

### 外部依赖

- `bun:bundle` — 功能标志检查的 `feature()`

## 实现细节

### 幂等性模式

每个迁移使用以下模式之一以确保恰好运行一次：

1. **完成标志**: 在全局配置中检查/设置布尔值（例如 `sonnet1m45MigrationComplete`）
2. **时间戳检查**: 检查时间戳字段是否存在（例如 `legacyOpusMigrationTimestamp`）
3. **值比较**: 检查当前值是否已匹配目标（例如模型字符串比较）
4. **源特定**: 仅读取/写入 `userSettings` 以避免将项目作用域的值提升为全局

### 分析

每个迁移记录一个 `tengu_*` 分析事件：
- `tengu_sonnet45_to_46_migration`
- `tengu_opus_to_opus1m_migration`
- `tengu_legacy_opus_migration`
- `tengu_migrate_autoupdates_to_settings`
- `tengu_migrate_bypass_permissions_accepted`
- `tengu_migrate_mcp_approval_fields_success/error`
- `tengu_migrate_reset_auto_opt_in_for_default_offer`
- `tengu_reset_pro_to_opus_default`

### 错误处理

可能失败的迁移（设置写入）包装在 try/catch 中并记录错误而不抛出——迁移失败不应阻止应用启动。

## 数据流

```
main.tsx 启动
    ↓
运行所有迁移（顺序对模型迁移很重要）
    ↓
每个迁移检查其守卫
    ↓
如果守卫通过: 迁移 → 记录事件 → 设置守卫
    ↓
应用继续使用迁移后的配置
```

## 相关模块

- [引导](./bootstrap.md) — 迁移读取/写入全局配置和设置
- [Hooks - 通知](./hooks-system.md) — `useModelMigrationNotifications` 显示迁移通知
- [Schemas](./schemas.md) — 设置 schemas 验证迁移后的数据
