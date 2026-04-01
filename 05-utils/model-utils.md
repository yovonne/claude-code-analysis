# Model Utilities

**Directory:** `restored-src/src/utils/model/`

## Purpose

Manages model selection, configuration, validation, and provider integration for Claude Code. Handles model aliases, 1M context windows, multi-provider support (first-party, Bedrock, Vertex, Foundry), and user-tier-based defaults.

## model.ts

### Core Model Resolution

- `getMainLoopModel()` - Returns the model for the current session
  - Priority: /model override > --model flag > ANTHROPIC_MODEL env > settings > default
- `getUserSpecifiedModelSetting()` - Gets user-specified model (respects allowlist)
- `getDefaultMainLoopModel()` - Returns built-in default model
- `getDefaultMainLoopModelSetting()` - Returns default based on user tier
- `getBestModel()` - Returns default Opus model
- `getRuntimeMainLoopModel(params)` - Gets model for runtime context (plan mode, token count)

### Default Models by Tier

- **Max/Team Premium**: Opus 4.6 (with 1M if enabled)
- **Pro/Team Standard/Enterprise/PAYG**: Sonnet 4.6
- **Ants**: Configurable via flag config, defaults to Opus 1M

### Model Name Resolution

- `parseUserSpecifiedModel(modelInput)` - Resolves aliases to full model names
  - Supports aliases: sonnet, opus, haiku, best, opusplan
  - Supports [1m] suffix on aliases
  - Legacy Opus 4.0/4.1 remap to current Opus (first-party only)
  - Ant model resolution via config

- `getCanonicalName(fullModelName)` - Maps full name to canonical short name
- `firstPartyNameToCanonical(name)` - Strips date/provider suffixes
- `getPublicModelDisplayName(model)` - Returns human-readable name or null
- `getMarketingNameForModel(modelId)` - Returns marketing name with 1M indicator
- `renderModelName(model)` - Returns display name (public or masked for ant)
- `getPublicModelName(model)` - Safe author name for git commits

### Model Aliases

- `opusplan` - Opus in plan mode, Sonnet otherwise
- `opus[1m]` - Opus with 1M context
- `sonnet[1m]` - Sonnet with 1M context

### 1M Context

- `isOpus1mMergeEnabled()` - Checks if Opus 1M merge is enabled
  - Disabled for Pro subscribers, 3P providers, or when 1M disabled
- `resolveSkillModelOverride(skillModel, currentModel)` - Resolves skill model with 1M carry-over

### Legacy Model Handling

- `isLegacyModelRemapEnabled()` - Checks if legacy Opus remap is enabled
- `isLegacyOpusFirstParty(model)` - Checks if model is legacy Opus 4.0/4.1

## modelOptions.ts

### Model Picker Options

- `getModelOptions(fastMode?)` - Returns full model option list for picker
  - Different lists per user tier (ant, Max/Team Premium, Pro, PAYG 1P, PAYG 3P)
  - Includes custom model from env var or current setting
  - Filters by availableModels allowlist

- `getDefaultOptionForUser(fastMode?)` - Returns default option for user tier
- `getSonnet46Option()` / `getOpus46Option()` / `getHaiku45Option()` - Individual options
- `getSonnet46_1MOption()` / `getOpus46_1MOption()` - 1M context options
- `getOpusPlanOption()` - Opus plan mode option

### Option Filtering

- `filterModelOptionsByAllowlist(options)` - Filters by availableModels setting
- `getKnownModelOption(model)` - Returns option with upgrade hint if newer available

## modelCapabilities.ts

### Model Capability Caching

- `getModelCapability(model)` - Returns cached capability for model
  - Cached in `~/.claude/cache/model-capabilities.json`
  - Matches by exact or substring (longest-id-first)
- `refreshModelCapabilities()` - Fetches and caches model capabilities from API
  - Only for ant users on first-party API
  - Skips if cache unchanged

## modelAllowlist.ts

### Model Allowlist

- `isModelAllowed(model)` - Checks if model is in availableModels allowlist
  - Supports family aliases (opus allows any opus version)
  - Supports exact model IDs

## aliases.ts

### Model Aliases

- `MODEL_ALIASES` - `['sonnet', 'opus', 'haiku', 'best', 'sonnet[1m]', 'opus[1m]', 'opusplan']`
- `MODEL_FAMILY_ALIASES` - `['sonnet', 'opus', 'haiku']` (wildcard in allowlist)
- `isModelAlias(modelInput)` - Type guard for alias
- `isModelFamilyAlias(model)` - Checks if model is family alias

## providers.ts

### API Provider Detection

- `getAPIProvider()` - Returns `'firstParty' | 'bedrock' | 'vertex' | 'foundry'`
  - Based on env vars: CLAUDE_CODE_USE_BEDROCK, CLAUDE_CODE_USE_VERTEX, CLAUDE_CODE_USE_FOUNDRY
- `isFirstPartyAnthropicBaseUrl()` - Checks if ANTHROPIC_BASE_URL is first-party
  - Allows api.anthropic.com and api-staging.anthropic.com (ant)

## validateModel.ts

### Model Validation

- `validateModel(model)` - Validates model via API call
  - Checks allowlist first
  - Checks alias and custom model env var
  - Makes minimal API call (1 token) to verify
  - Caches valid models
  - Provides 3P fallback suggestions (Opus 4.6 -> 4.1, Sonnet 4.6 -> 4.5)

## check1mAccess.ts

### 1M Context Access

- `checkOpus1mAccess()` - Checks if user has Opus 1M access
- `checkSonnet1mAccess()` - Checks if user has Sonnet 1M access
- `modelSupports1M(model)` - Checks if model supports 1M context
- `has1mContext(model)` - Checks if model string has [1m] suffix
- `is1mContextDisabled()` - Checks if 1M context is disabled

## contextWindowUpgradeCheck.ts

### Upgrade Prompts

- `getUpgradeMessage(context)` - Returns upgrade message for context warning or tip
  - Detects available upgrade (opus -> opus[1m], sonnet -> sonnet[1m])
  - Returns `/model opus[1m]` or tip about 5x more context

## bedrock.ts

### AWS Bedrock Integration

- `getBedrockInferenceProfiles()` - Lists Bedrock inference profiles (memoized)
- `getInferenceProfileBackingModel(profileId)` - Gets backing model for profile
- `findFirstMatch(profiles, substring)` - Finds first matching profile
- `createBedrockClient()` - Creates Bedrock client with credentials
- `createBedrockRuntimeClient()` - Creates Bedrock Runtime client

### Bedrock Model ID Handling

- `isFoundationModel(modelId)` - Checks if model is foundation model (anthropic.*)
- `extractModelIdFromArn(modelId)` - Extracts model ID from ARN
- `getBedrockRegionPrefix(modelId)` - Extracts region prefix (us, eu, apac, global)
- `applyBedrockRegionPrefix(modelId, prefix)` - Applies/replaces region prefix

## modelStrings.ts

### Model ID Strings

- `getModelStrings()` - Returns model ID strings for current provider
  - First-party: claude-opus-4-6-20250514, etc.
  - Bedrock: us.anthropic.claude-opus-4-6-v1:0, etc.
  - Vertex: claude-opus-4-6@20250514, etc.
  - Foundry: deployment IDs
- `resolveOverriddenModel(modelId)` - Resolves model overrides (Bedrock ARNs, etc.)

## configs.ts

### Model Configuration

- Model-specific configuration and feature flags

## deprecation.ts

### Model Deprecation

- Handles deprecated model warnings and migrations

## agent.ts

### Agent Model Configuration

- Model configuration for agent/subagent execution

## antModels.ts

### Ant-Only Model Configuration

- Ant-specific model aliases and configurations
- Resolves ant model aliases to actual model IDs

## modelSupportOverrides.ts

### Model Support Overrides

- Overrides for model support based on context

## Dependencies

- `auth.js` - Subscription type checks
- `config.js` - Global config access
- `envUtils.js` - Environment variable checks
- `settings/settings.js` - Settings access
- `sideQuery.js` - Side-channel API queries
- `modelCost.js` - Model pricing
- `context.js` - Context window checks
