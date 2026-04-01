# 模型工具

**目录：** `restored-src/src/utils/model/`

## 目的

管理 Claude Code 的模型选择、配置、验证和提供商集成。处理模型别名、1M 上下文窗口、多提供商支持（第一方、Bedrock、Vertex、Foundry）以及基于用户层级的默认值。

## model.ts

### 核心模型解析

- `getMainLoopModel()` - 返回当前会话的模型
  - 优先级：/model 覆盖 > --model 标志 > ANTHROPIC_MODEL 环境 > 设置 > 默认值
- `getUserSpecifiedModelSetting()` - 获取用户指定的模型（遵循允许列表）
- `getDefaultMainLoopModel()` - 返回内置默认模型
- `getDefaultMainLoopModelSetting()` - 基于用户层级返回默认值
- `getBestModel()` - 返回默认 Opus 模型
- `getRuntimeMainLoopModel(params)` - 获取运行时上下文的模型（计划模式、令牌计数）

### 按层级的默认模型

- **Max/Team Premium**: Opus 4.6（如果启用则带 1M）
- **Pro/Team Standard/Enterprise/PAYG**: Sonnet 4.6
- **Ants**: 通过标志配置可配置，默认为 Opus 1M

### 模型名称解析

- `parseUserSpecifiedModel(modelInput)` - 将别名解析为完整模型名称
  - 支持别名：sonnet、opus、haiku、best、opusplan
  - 支持别名上的 [1m] 后缀
  - 遗留 Opus 4.0/4.1 重新映射到当前 Opus（仅第一方）
  - Ant 模型解析通过配置
- `getCanonicalName(fullModelName)` - 将全名映射到规范短名称
- `firstPartyNameToCanonical(name)` - 剥离日期/提供商后缀
- `getPublicModelDisplayName(model)` - 返回人类可读名称或 null
- `getMarketingNameForModel(modelId)` - 返回带 1M 指示器的营销名称
- `renderModelName(model)` - 返回显示名称（公共或为 ant 隐藏）
- `getPublicModelName(model)` - 用于 git 提交的安全作者名称

### 模型别名

- `opusplan` - 计划模式下的 Opus，否则为 Sonnet
- `opus[1m]` - 带 1M 上下文的 Opus
- `sonnet[1m]` - 带 1M 上下文的 Sonnet

### 1M 上下文

- `isOpus1mMergeEnabled()` - 检查 Opus 1M 合并是否启用
  - 为 Pro 订阅者、3P 提供商或 1M 禁用时禁用
- `resolveSkillModelOverride(skillModel, currentModel)` - 解析带 1M 延续的 skill 模型

### 遗留模型处理

- `isLegacyModelRemapEnabled()` - 检查遗留 Opus 重新映射是否启用
- `isLegacyOpusFirstParty(model)` - 检查模型是否为遗留 Opus 4.0/4.1

## modelOptions.ts

### 模型选择器选项

- `getModelOptions(fastMode?)` - 返回选择器的完整模型选项列表
  - 每个用户层级不同的列表（ant、Max/Team Premium、Pro、PAYG 1P、PAYG 3P）
  - 包括来自环境变量或当前设置的自定义模型
  - 按 availableModels 允许列表过滤
- `getDefaultOptionForUser(fastMode?)` - 返回用户层级的默认选项
- `getSonnet46Option()` / `getOpus46Option()` / `getHaiku45Option()` - 单个选项
- `getSonnet46_1MOption()` / `getOpus46_1MOption()` - 1M 上下文选项
- `getOpusPlanOption()` - Opus 计划模式选项

### 选项过滤

- `filterModelOptionsByAllowlist(options)` - 按允许列表过滤选项
- `getKnownModelOption(model)` - 如果有更新的选项可用则返回带升级提示的选项

## modelCapabilities.ts

### 模型能力缓存

- `getModelCapability(model)` - 返回模型的缓存能力
  - 缓存在 `~/.claude/cache/model-capabilities.json`
  - 按精确或子字符串匹配（最长 ID 优先）
- `refreshModelCapabilities()` - 从 API 获取并缓存模型能力
  - 仅适用于第一方 API 上的 ant 用户
  - 如果缓存未更改则跳过

## modelAllowlist.ts

### 模型允许列表

- `isModelAllowed(model)` - 检查模型是否在 availableModels 允许列表中
  - 支持系列别名（opus 允许任何 opus 版本）
  - 支持精确模型 ID

## aliases.ts

### 模型别名

- `MODEL_ALIASES` - `['sonnet', 'opus', 'haiku', 'best', 'sonnet[1m]', 'opus[1m]', 'opusplan']`
- `MODEL_FAMILY_ALIASES` - `['sonnet', 'opus', 'haiku']`（允许列表中的通配符）
- `isModelAlias(modelInput)` - 别名的类型守卫
- `isModelFamilyAlias(model)` - 检查模型是否为系列别名

## providers.ts

### API 提供商检测

- `getAPIProvider()` - 返回 `'firstParty' | 'bedrock' | 'vertex' | 'foundry'`
  - 基于环境变量：CLAUDE_CODE_USE_BEDROCK、CLAUDE_CODE_USE_VERTEX、CLAUDE_CODE_USE_FOUNDRY
- `isFirstPartyAnthropicBaseUrl()` - 检查 ANTHROPIC_BASE_URL 是否为第一方
  - 允许 api.anthropic.com 和 api-staging.anthropic.com（ant）

## validateModel.ts

### 模型验证

- `validateModel(model)` - 通过 API 调用验证模型
  - 首先检查允许列表
  - 检查别名和自定义模型环境变量
  - 进行最小 API 调用（1 个令牌）进行验证
  - 缓存有效模型
  - 提供 3P 回退建议（Opus 4.6 -> 4.1、Sonnet 4.6 -> 4.5）

## check1mAccess.ts

### 1M 上下文访问

- `checkOpus1mAccess()` - 检查用户是否有 Opus 1M 访问权限
- `checkSonnet1mAccess()` - 检查用户是否有 Sonnet 1M 访问权限
- `modelSupports1M(model)` - 检查模型是否支持 1M 上下文
- `has1mContext(model)` - 检查模型字符串是否有 [1m] 后缀
- `is1mContextDisabled()` - 检查 1M 上下文是否被禁用

## contextWindowUpgradeCheck.ts

### 升级提示

- `getUpgradeMessage(context)` - 返回上下文警告或提示的升级消息
  - 检测可用升级（opus -> opus[1m]、sonnet -> sonnet[1m]）
  - 返回 `/model opus[1m]` 或关于 5 倍更多上下文的提示

## bedrock.ts

### AWS Bedrock 集成

- `getBedrockInferenceProfiles()` - 列出 Bedrock 推理配置文件（记忆化）
- `getInferenceProfileBackingModel(profileId)` - 获取配置文件的支撑模型
- `findFirstMatch(profiles, substring)` - 查找第一个匹配的配置
- `createBedrockClient()` - 使用凭证创建 Bedrock 客户端
- `createBedrockRuntimeClient()` - 创建 Bedrock Runtime 客户端

### Bedrock 模型 ID 处理

- `isFoundationModel(modelId)` - 检查模型是否为 foundation 模型（anthropic.*）
- `extractModelIdFromArn(modelId)` - 从 ARN 提取模型 ID
- `getBedrockRegionPrefix(modelId)` - 提取区域前缀（us、eu、apac、global）
- `applyBedrockRegionPrefix(modelId, prefix)` - 应用/替换区域前缀

## modelStrings.ts

### 模型 ID 字符串

- `getModelStrings()` - 返回当前提供商的模型 ID 字符串
  - 第一方：claude-opus-4-6-20250514 等
  - Bedrock：us.anthropic.claude-opus-4-6-v1:0 等
  - Vertex：claude-opus-4-6@20250514 等
  - Foundry：部署 ID
- `resolveOverriddenModel(modelId)` - 解析模型覆盖（Bedrock ARN 等）

## configs.ts

### 模型配置

- 模型特定配置和特性标志

## deprecation.ts

### 模型弃用

- 处理弃用模型警告和迁移

## agent.ts

### Agent 模型配置

- agent/subagent 执行的模型配置

## antModels.ts

### 仅 Ant 模型配置

- Ant 特定的模型别名和配置
- 将 ant 模型别名解析为实际模型 ID

## modelSupportOverrides.ts

### 模型支持覆盖

- 基于上下文的模型支持覆盖

## 依赖

- `auth.js` - 订阅类型检查
- `config.js` - 全局配置访问
- `envUtils.js` - 环境变量检查
- `settings/settings.js` - 设置访问
- `sideQuery.js` - 侧通道 API 查询
- `modelCost.js` - 模型定价
- `context.js` - 上下文窗口检查
