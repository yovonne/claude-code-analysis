# 设置工具

**目录：** `restored-src/src/utils/settings/`

## 目的

跨多个源管理 Claude Code 设置文件，具有基于优先级的合并、模式验证、MDM/配置文件 enforcement 和更改检测。

## 设置源（优先级：低到高）

1. **pluginSettings** - 插件提供的基础设置
2. **userSettings** - `~/.claude/settings.json`（或 `cowork_settings.json`）
3. **projectSettings** - 项目根目录中的 `.claude/settings.json`
4. **localSettings** - 项目根目录中的 `.claude/settings.local.json`
5. **flagSettings** - SDK 内联设置 + 标志文件
6. **policySettings** - 托管设置（远程 > MDM > 文件 > HKCU）

## settings.ts

### 核心函数

- `getSettingsForSource(source)` - 返回特定源的设置（缓存）
- `getInitialSettings()` - 从所有源返回合并的设置
- `getSettings_DEPRECATED()` - getInitialSettings 的别名（向后兼容）
- `getSettingsWithSources()` - 返回有效设置加上按源细分
- `getSettingsWithErrors()` - 返回带验证错误的合并设置
- `updateSettingsForSource(source, settings)` - 更新可编辑源的设置
- `getSettingsFilePathForSource(source)` - 返回源的配置文件路径
- `getSettingsRootPathForSource(source)` - 返回源的根目录

### 文件解析

- `parseSettingsFile(path)` - 解析并验证设置文件（缓存）
- `loadManagedFileSettings()` - 加载 managed-settings.json + drop-in 目录
- `getManagedFileSettingsPresence()` - 检查哪些托管文件源存在

### 合并

- `settingsMergeCustomizer(objValue, srcValue)` - lodash mergeWith 自定义函数
  - 数组：连接和去重
  - 对象：深度合并
  - 未定义：删除键

### 安全检查

- `hasSkipDangerousModePermissionPrompt()` - 检查信任源的绕过接受
- `hasAutoModeOptIn()` - 检查信任源的自动模式选择加入
- `getUseAutoModeDuringPlan()` - 检查计划模式是否使用自动模式语义
- `getAutoModeConfig()` - 返回合并的自动模式分类器配置
- `rawSettingsContainsKey(key)` - 检查任何原始设置文件是否包含键

### 托管设置键

- `getManagedSettingsKeysForLogging(settings)` - 提取用于日志显示的键
  - 将权限、沙箱、hooks 展开到一层深度

## settingsCache.ts

### 缓存层

1. **sessionSettingsCache** - 整个会话的合并设置
2. **perSourceCache** - 按源设置（Map<SettingSource, SettingsJson>）
3. **parseFileCache** - 按文件解析结果（Map<path, ParsedSettings>）
4. **pluginSettingsBase** - 插件设置基础层

- `resetSettingsCache()` - 清除所有缓存（在设置写入时调用）
- `getPluginSettingsBase()` / `setPluginSettingsBase()` - 插件基础层访问

## types.ts

### SettingsSchema

涵盖所有设置字段的综合 Zod v4 模式：

- **认证**：apiKeyHelper、awsCredentialExport、awsAuthRefresh、gcpAuthRefresh、xaaIdp
- **模型**：model、availableModels、modelOverrides、effortLevel、advisorModel
- **权限**：permissions（allow/deny/ask/defaultMode/disableBypassPermissionsMode/disableAutoMode/additionalDirectories）
- **MCP**：enabledMcpjsonServers、disabledMcpjsonServers、allowedMcpServers、deniedMcpServers、enableAllProjectMcpServers
- **插件**：enabledPlugins、extraKnownMarketplaces、strictKnownMarketplaces、blockedMarketplaces
- **Hooks**：hooks（PreToolUse、PostToolUse、Notification 等）、disableAllHooks、allowManagedHooksOnly
- **沙箱**：sandbox 设置
- **环境**：env 变量
- **Git**：worktree（symlinkDirectories、sparsePaths）、respectGitignore
- **UI**：outputStyle、language、terminalTitleFromRename、syntaxHighlightingDisabled
- **特性**：agent、fastMode、alwaysThinkingEnabled、promptSuggestionEnabled
- **远程**：remote.defaultEnvironmentId、sshConfigs
- **自动模式**：autoMode（allow、soft_deny、environment）、skipAutoPermissionPrompt
- **企业**：strictPluginOnlyCustomization、allowManagedMcpServersOnly、allowedHttpHookUrls
- **其他**：cleanupPeriodDays、attribution、plansDirectory、voiceEnabled、assistant

### 子模式

- `PermissionsSchema` - 权限规则和模式配置
- `EnvironmentVariablesSchema` - 环境变量记录
- `AllowedMcpServerEntrySchema` - MCP 服务器允许列表条目
- `DeniedMcpServerEntrySchema` - MCP 服务器拒绝列表条目
- `ExtraKnownMarketplaceSchema` - 自定义市场定义

### 类型守卫

- `isMcpServerNameEntry()` / `isMcpServerCommandEntry()` / `isMcpServerUrlEntry()`

## validation.ts

### 验证函数

- `formatZodError(error, filePath)` - 将 Zod 错误转换为 ValidationError 数组
- `validateSettingsFileContent(content)` - 验证设置字符串内容
- `filterInvalidPermissionRules(data, filePath)` - 在模式验证之前移除无效的权限规则
- `getValidationTip()` - 返回常见错误的修复建议

### ValidationError 类型

```typescript
{
  file?: string
  path: FieldPath  // 点表示法
  message: string
  expected?: string
  invalidValue?: unknown
  suggestion?: string
  docLink?: string
  mcpErrorMetadata?: { scope, serverName?, severity? }
}
```

## managedPath.ts

### 平台特定路径

- `getManagedFilePath()` - 返回托管设置目录：
  - macOS: `/Library/Application Support/ClaudeCode`
  - Windows: `C:\Program Files\ClaudeCode`
  - Linux: `/etc/claude-code`
- `getManagedSettingsDropInDir()` - 返回 `managed-settings.d` 子目录

## mdm/settings.ts

### MDM 设置管理

- `startMdmSettingsLoad()` - 在启动时启动异步 MDM/HKCU 读取
- `ensureMdmSettingsLoaded()` - 等待飞行中的 MDM 加载
- `getMdmSettings()` - 返回管理员控制的 MDM 设置（缓存）
- `getHkcuSettings()` - 返回 HKCU 注册表设置（Windows，缓存）
- `refreshMdmSettings()` - 触发新的子进程读取
- `clearMdmSettingsCache()` - 清除 MDM 和 HKCU 缓存
- `setMdmSettingsCache(mdm, hkcu)` - 更新会话缓存

### 策略优先级（第一源优先）

1. 远程托管设置（最高）
2. 管理员 MDM（HKLM / macOS plist）
3. managed-settings.json + drop-ins
4. HKCU 注册表（最低，用户可写）

## mdm/constants.ts

### MDM 常量

- `MACOS_PREFERENCE_DOMAIN` - `com.anthropic.claudecode`
- `WINDOWS_REGISTRY_KEY_PATH_HKLM` - `HKLM\SOFTWARE\Policies\ClaudeCode`
- `WINDOWS_REGISTRY_KEY_PATH_HKCU` - `HKCU\SOFTWARE\Policies\ClaudeCode`
- `WINDOWS_REGISTRY_VALUE_NAME` - `Settings`
- `PLUTIL_PATH` - `/usr/bin/plutil`
- `MDM_SUBPROCESS_TIMEOUT_MS` - 5000ms
- `getMacOSPlistPaths()` - 按优先级顺序返回 plist 路径

## mdm/rawRead.ts

### 子进程 I/O

- `fireRawRead()` - 为 MDM 设置触发新的子进程读取
  - macOS: 并行为每个 plist 路径调用 plutil
  - Windows: 并行为 HKLM 和 HKCU 调用 reg query
  - Linux: 返回空
- `startMdmRawRead()` - 在启动时触发一次
- `getMdmRawReadPromise()` - 获取启动承诺

## constants.ts

- `CLAUDE_CODE_SETTINGS_SCHEMA_URL` - JSON Schema 引用 URL
- `SettingSource` 类型联合
- `EditableSettingSource` 类型（排除 policySettings、flagSettings）
- `getEnabledSettingSources()` - 基于上下文返回活动源

## changeDetector.ts

- 检测磁盘上的设置文件更改
- MDM 设置刷新的 30 分钟轮询间隔
- 更改时触发缓存失效

## applySettingsChange.ts

- 将设置更改应用到运行中的会话
- 处理设置修改后的上下文更新

## allErrors.ts

- 聚合所有设置错误用于显示
- 由 /status 和诊断使用

## validationTips.ts

- 提供修复验证错误的人类可读提示
- 将错误代码映射到建议和文档链接

## schemaOutput.ts

- 从 SettingsSchema 生成 JSON Schema 输出
- 用于验证错误报告

## internalWrites.ts

- 跟踪内部文件写入以防止更改检测器误报
- `markInternalWrite(path)` - 将路径标记为内部写入

## permissionValidation.ts

- `validatePermissionRule(ruleString)` - 验证单个权限规则字符串
- `PermissionRuleSchema` - 权限规则验证的 Zod 模式

## toolValidationConfig.ts / validateEditTool.ts

- 工具特定验证配置
- Edit 工具验证辅助函数

## pluginOnlyPolicy.ts

- 强制执行 strictPluginOnlyCustomization 策略
- 为锁定表面阻止非插件自定义源
