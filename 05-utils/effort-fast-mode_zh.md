# 努力程度和快速模式

## 目的

两个相关但独立的特性工具：**努力程度（Effort）** 控制发送到 API 的推理深度/计算级别（低/中/高/最大），而**快速模式（Fast Mode）**（"penguin mode"）是一种使用更快响应路径的速度优化。两者都是订阅门控和模型依赖的特性，具有复杂的可用性逻辑。

## 位置

- `restored-src/src/utils/effort.ts` — 努力程度管理、模型支持、默认值、持久化
- `restored-src/src/utils/fastMode.ts` — 快速模式可用性、运行时状态、冷却、组织状态、预取

## 主要导出

### 努力程度（`effort.ts`）

#### 类型
- `EffortLevel`: `'low' | 'medium' | 'high' | 'max'`
- `EffortValue`: `EffortLevel | number` — 数字值仅为模型默认值，不会持久化
- `OpusDefaultEffortConfig`: `{ enabled, dialogTitle, dialogDescription }` — Opus 默认努力程度对话框的远程配置

#### 常量
- `EFFORT_LEVELS`: `['low', 'medium', 'high', 'max']`

#### 模型支持
- `modelSupportsEffort(model)`: 检查模型是否支持 effort 参数。Opus 4.6 和 Sonnet 4.6 支持；haiku 和更早变体不支持。未知模型在第一方默认 true，第三方默认 false。
- `modelSupportsMaxEffort(model)`: 检查模型是否支持 `'max'` effort。公共模型仅 Opus 4.6。

#### 解析和转换
- `isEffortLevel(value)`: 有效 effort 级别字符串的类型守卫
- `parseEffortValue(value)`: 将未知输入解析为 EffortValue（处理字符串、数字、null）
- `isValidNumericEffort(value)`: 检查数字是否为有效数值 effort（必须是整数）
- `convertEffortValueToLevel(value)`: 将 EffortValue 转换为 EffortLevel。数字值映射到范围（≤50→低、≤85→中、≤100→高、>100→最大），仅限 ants。
- `toPersistableEffort(value)`: 转换为可持久化形式。仅 `'low'`、`'medium'`、`'high'` 可为外部用户持久化；ants 也可以持久化 `'max'`。

#### 解析
- `getInitialEffortSetting()`: 从初始设置获取 effort，通过 `toPersistableEffort` 过滤
- `resolveAppliedEffort(model, appStateEffortValue)`: 解析实际发送到 API 的 effort 值。优先级：环境 `CLAUDE_CODE_EFFORT_LEVEL` → appState.effortValue → 模型默认值。在非 Opus-4.6 模型上将 `'max'` 降级为 `'high'`。
- `getDisplayedEffortLevel(model, appStateEffort)`: 解析显示给用户的 effort 级别。用 `'high'` 回退包装 `resolveAppliedEffort`（API 在未发送 effort 参数时使用）。
- `getEffortSuffix(model, effortValue)`: 构建显示在 Logo/Spinner 中的 ` with {level} effort` 后缀。如果没有设置 effort 则为空字符串。
- `getDefaultEffortForModel(model)`: 获取模型的默认 effort。Opus 4.6 默认为 Pro/Max/Team 订阅者 `'medium'`；启用 ultrathink 的模型默认为 `'medium'`。
- `resolvePickerEffortPersistence(picked, modelDefault, priorPersisted, toggledInPicker)`: 决定用户在 ModelPicker 中选择模型时要持久化的 effort。保持明确选择 sticky，同时让默认值通过。

#### 环境覆盖
- `getEffortEnvOverride()`: 读取 `CLAUDE_CODE_EFFORT_LEVEL` 环境变量。`'unset'` 或 `'auto'` 返回 `null`。

#### 描述
- `getEffortLevelDescription(level)`: 每个努力程度的可读描述
- `getEffortValueDescription(value)`: 字符串和数字 effort 值的描述
- `getOpusDefaultEffortConfig()`: 获取 Opus 默认努力程度推荐对话框的远程配置

### 快速模式（`fastMode.ts`）

#### 类型
- `FastModeRuntimeState`: `{ status: 'active' } | { status: 'cooldown', resetAt, reason }`
- `FastModeDisabledReason`: `'free' | 'preference' | 'extra_usage_disabled' | 'network_error' | 'unknown'`
- `CooldownReason`: `'rate_limit' | 'overloaded'`

#### 常量
- `FAST_MODE_MODEL_DISPLAY`: `'Opus 4.6'`

#### 可用性检查
- `isFastModeEnabled()`: 通过环境变量（`CLAUDE_CODE_DISABLE_FAST_MODE`）检查快速模式是否启用
- `isFastModeAvailable()`: 检查快速模式是否可用（启用 + 无不可用原因）
- `getFastModeUnavailableReason()`: 获取快速模式不可用的原因，或 null。检查：Statsig 门控、捆绑模式要求、SDK 会话限制、第一方提供商要求、组织状态。
- `isFastModeSupportedByModel(modelSetting)`: 检查当前模型是否支持快速模式（仅 Opus 4.6）
- `getFastModeModel()`: 获取快速模式模型字符串（如果启用 Opus 1m 合并则包含 `[1m]` 后缀）
- `getInitialFastModeModel(model)`: 基于模型、可用性和用户偏好获取初始快速模式设置

#### 运行时状态
- `getFastModeRuntimeState()`: 获取当前运行时状态。冷却过期时自动重置。
- `triggerFastModeCooldown(resetTimestamp, reason)`: 在速率限制或过载后进入冷却。发出分析并通知监听器。
- `clearFastModeCooldown()`: 手动清除冷却。
- `isFastModeCooldown()`: 检查当前是否处于冷却状态。
- `getFastModeState(model, fastModeUserEnabled)`: 获取聚合状态：`'off' | 'cooldown' | 'on'`

#### API 拒绝处理
- `handleFastModeRejectedByAPI()`: 在 API 拒绝快速模式时调用（例如 400 "your organization not enabled"）。永久禁用快速模式。
- `handleFastModeOverageRejection(reason)`: 在 429 表示额外使用不可用时调用。除非信用额度用完，否则永久禁用。
- `onFastModeOverageRejection`: 订阅超额拒绝事件
- `onOrgFastModeChanged`: 订阅组织级快速模式状态更改
- `onCooldownTriggered`、`onCooldownExpired`: 订阅冷却生命周期事件

#### 组织状态预取
- `prefetchFastModeStatus()`: 从 API 获取组织快速模式状态，最小间隔 30 秒。处理带令牌刷新的认证错误。失败时回退到缓存。
- `resolveFastModeStatusFromCache()`: 从持久化缓存解析 orgStatus，无需网络调用（在启动预取被限制时使用）

#### 超额消息
- `getOverageDisabledMessage(reason)`: 获取各种超额拒绝原因的用户面向消息

## 设计注意事项

- **Effort 优先级链**：环境 `CLAUDE_CODE_EFFORT_LEVEL` → `appState.effortValue` → 模型默认值。环境覆盖可以用 `'unset'` 或 `'auto'` 显式取消设置 effort。
- **最大 effort 是会话范围的**：对于外部用户，它不能持久化到 settings.json。只有内部用户（ants）可以持久化 `'max'`。
- **数值 effort 值**：仅为模型默认值，永不持久化。它们通过 `convertEffortValueToLevel` 转换为命名级别用于显示。
- **快速模式冷却**：与用户偏好分开跟踪 — 它反映实际运营状态（主动发送快速速度 vs 速率限制后的冷却）。
- **快速模式组织状态**：从 API 获取并缓存。失败时，ants 默认为启用；外部用户回退到缓存值或以 `network_error` 原因禁用。
- **快速模式预取**：使用 30 秒最小间隔以避免过多 API 调用。飞行中的请求被去重。
- **超额 vs 组织禁用**：是区分的：超额意味着额外使用计费不可用；组织禁用意味着组织已完全关闭快速模式。
