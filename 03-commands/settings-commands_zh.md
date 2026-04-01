# 设置命令

## 概述

设置命令管理运行时行为，包括努力级别、快速模式、工具权限、沙箱、使用限制和速率限制处理。这些命令提供交互式 UI 和直接基于参数的控制。

---

## `/effort`

**用途：** 设置模型使用的努力级别，控制模型推理深度和处理任务的程度。

**来源：** `restored-src/src/commands/effort/`

| 文件 | 用途 |
|------|---------|
| `index.ts` | 带动态即时模式的命令注册 |
| `effort.tsx` | 努力级别逻辑、验证和状态管理 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'effort',
  description: 'Set effort level for model usage',
  argumentHint: '[low|medium|high|max|auto]',
  immediate: shouldInferenceConfigCommandBeImmediate(),
  load: () => import('./effort.js'),
}
```

**努力级别：**

| 级别 | 描述 |
|-------|-------------|
| `low` | 快速、直接的实现 |
| `medium` | 平衡方法，标准测试 |
| `high` | 全面的实现，广泛测试 |
| `max` | 最大能力，最深度推理（仅 Opus 4.6） |
| `auto` | 使用模型的默认努力级别 |

**使用模式：**

| 调用 | 行为 |
|------------|-------------|
| `/effort` | 显示当前努力级别 |
| `/effort current` | 显示当前努力级别 |
| `/effort status` | 显示当前努力级别 |
| `/effort [level]` | 设置为指定级别 |
| `/effort auto` | 重置为 auto/默认 |
| `/effort unset` | 与 `auto` 相同 — 清除努力设置 |
| `/effort --help` | 显示完整帮助和所有级别描述 |

**设置逻辑（`setEffortValue`）：**
1. 通过 `toPersistableEffort` 转换为可持久化形式
2. 通过 `updateSettingsForSource('userSettings', ...)` 更新用户设置
3. 检查环境变量覆盖（`CLAUDE_CODE_EFFORT_LEVEL`）
4. 如果环境变量冲突，警告用户环境变量优先
5. 记录 `tengu_effort_command` 分析事件
6. 如果不可持久化，返回带描述和仅会话后缀的消息

**显示逻辑（`showCurrentEffort`）：**
- 解析有效努力：环境覆盖 > 应用状态 > 模型默认
- 当未设置显式级别时显示 `auto (currently <level>)`
- 当设置时显示 `Current effort level: <value> (<description>)`

**取消设置逻辑（`unsetEffortLevel`）：**
- 从用户设置中清除努力
- 如果 `CLAUDE_CODE_EFFORT_LEVEL` 环境变量仍控制会话则发出警告

**验证（`executeEffort`）：**
- 将输入规范化为小写
- 接受 `auto` 或 `unset` 以清除
- 通过 `isEffortLevel` 针对已知努力级别验证
- 对无效参数返回错误消息

---

## `/fast`

**用途：** 切换快速模式 — 具有独立速率限制和高级计费的高速研究预览模式。

**来源：** `restored-src/src/commands/fast/`

| 文件 | 用途 |
|------|---------|
| `index.ts` | 带动态可用性的命令注册 |
| `fast.tsx` | 快速模式选择器、切换逻辑和状态管理 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'fast',
  description: `Toggle fast mode (${FAST_MODE_MODEL_DISPLAY} only)`,
  availability: ['claude-ai', 'console'],
  isEnabled: () => isFastModeEnabled(),
  isHidden: !isFastModeEnabled(),
  argumentHint: '[on|off]',
  immediate: shouldInferenceConfigCommandBeImmediate(),
  load: () => import('./fast.js'),
}
```

**可用性：** 仅当快速模式启用（`isFastModeEnabled()`）时可见。在 `claude-ai` 和 `console` 上下文中可用。

**使用模式：**

| 调用 | 行为 |
|------------|-------------|
| `/fast` | 打开交互式 `FastModePicker` 对话框 |
| `/fast on` | 直接启用快速模式 |
| `/fast off` | 直接禁用快速模式 |

**快速模式选择器（`FastModePicker`）：**
- 显示当前快速模式状态和切换控制
- 如果快速模式无法启用则显示不可用原因
- 当受限速率时显示带重置计时器的冷却状态
- 显示快速模式模型层级的定价信息
- 确认时：应用设置，记录分析，显示结果消息
- 取消时：保留原始状态
- 键盘绑定：`Enter` 确认，`Tab`/箭头切换，`Esc` 取消

**应用逻辑（`applyFastMode`）：**
1. 清除快速模式冷却计时器
2. 更新用户设置（`fastMode: true` 或清除为 `undefined`）
3. 启用时：
   - 检查当前模型是否支持快速模式
   - 如果不支持，自动切换到快速模式模型
   - 在应用状态中设置 `fastMode: true`
4. 禁用时：
   - 在应用状态中设置 `fastMode: false`

**快捷方式处理器（`handleFastModeShortcut`）：**
- 用于直接的 `/fast on` 和 `/fast off` 调用
- 应用前检查可用性
- 返回带图标、模型更改通知和定价的格式化结果消息

**预检：**
- 在显示选择器前调用 `prefetchFastModeStatus()` 检查组织级快速模式状态

---

## `/permissions`（别名：`/allowed-tools`）

**用途：** 管理允许和拒绝工具权限规则。

**来源：** `restored-src/src/commands/permissions/`

| 文件 | 用途 |
|------|---------|
| `index.ts` | 带 `allowed-tools` 别名的命令注册 |
| `permissions.tsx` | 渲染权限规则列表的入口点 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'permissions',
  aliases: ['allowed-tools'],
  description: 'Manage allow & deny tool permission rules',
  load: () => import('./permissions.js'),
}
```

**行为：**
- 渲染 `PermissionRuleList` 组件用于管理工具权限
- 退出时：调用 `onDone` 关闭
- 重试拒绝时：通过 `context.setMessages` 创建重试消息并添加到消息队列

**使用的组件：**
- `PermissionRuleList` — 带添加/编辑/删除功能的交互式权限规则列表

---

## `/sandbox`

**用途：** 配置命令执行的沙箱化 — 控制 shell 命令是否在隔离环境中运行。

**来源：** `restored-src/src/commands/sandbox-toggle/`

| 文件 | 用途 |
|------|---------|
| `index.ts` | 带动态状态描述的命令注册 |
| `sandbox-toggle.tsx` | 沙箱设置 UI 和 exclude 子命令 |

**注册：**
```typescript
{
  name: 'sandbox',
  description: `${icon} ${statusText} (⏎ to configure)`,
  argumentHint: 'exclude "command pattern"',
  isHidden: !isSupportedPlatform() || !isPlatformInEnabledList(),
  immediate: true,
  type: 'local-jsx',
  load: () => import('./sandbox-toggle.js'),
}
```

**动态描述** 反映当前状态：
- `⚠ sandbox disabled` — 缺少依赖
- `✓ sandbox enabled` — 沙箱化活动
- `✓ sandbox enabled (auto-allow)` — 带自动允许 bash 的沙箱化
- `○ sandbox disabled` — 沙箱化关闭
- `, fallback allowed` — 当允许非沙箱命令时附加
- `(managed)` — 当设置被策略锁定时附加

**可用性：** 在不支持的平台或平台不在启用列表中时隐藏。

**使用模式：**

| 调用 | 行为 |
|------------|-------------|
| `/sandbox` | 打开交互式 `SandboxSettings` 对话框 |
| `/sandbox exclude "pattern"` | 将命令模式添加到排除命令列表 |

**预检检查（按顺序）：**
1. 平台支持 — 在非 macOS/Linux/WSL2 平台上出错
2. 依赖检查 — 验证沙箱化依赖已安装
3. 平台启用列表 — 检查企业 `enabledPlatforms` 设置
4. 策略锁定 — 检查沙箱设置是否被更高优先级配置覆盖

**Exclude 子命令：**
- 从参数中解析 `exclude "command pattern"`
- 去除模式周围的引号
- 调用 `addToExcludedCommands(pattern)` 持久化
- 报告本地设置文件的相对路径

**使用的组件：**
- `SandboxSettings` — 交互式沙箱配置对话框

---

## `/reset-limits`

**用途：** 重置使用限制。

**来源：** `restored-src/src/commands/reset-limits/`

| 文件 | 用途 |
|------|---------|
| `index.js` | 存根实现（禁用和隐藏） |

**状态：** 此命令当前是存根 — `isEnabled: () => false` 和 `isHidden: true`。它在当前代码库中不生效。定义了两个导出（`resetLimits` 和 `resetLimitsNonInteractive`），但都指向同一个存根。

---

## `/mock-limits`

**用途：** 模拟使用限制（可能用于测试）。

**来源：** `restored-src/src/commands/mock-limits/`

| 文件 | 用途 |
|------|---------|
| `index.js` | 存根实现（禁用和隐藏） |

**状态：** 此命令当前是存根 — `isEnabled: () => false` 和 `isHidden: true`。它在当前代码库中不生效。

---

## `/rate-limit-options`

**用途：** 达到速率限制时显示选项 — 升级计划、启用额外使用或等待。

**来源：** `restored-src/src/commands/rate-limit-options/`

| 文件 | 用途 |
|------|---------|
| `index.ts` | 命令注册（仅限 Claude AI 订阅者，隐藏） |
| `rate-limit-options.tsx` | 带动态选项的速率限制选项菜单 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'rate-limit-options',
  description: 'Show options when rate limit is reached',
  isEnabled: () => isClaudeAISubscriber(),
  isHidden: true, // 从帮助中隐藏 — 仅在内部使用
  load: () => import('./rate-limit-options.js'),
}
```

**可用性：** 仅对 Claude AI 订阅者启用。从帮助中隐藏 — 在达到速率限制时在内部触发。

**菜单选项（动态组装）：**

| 选项 | 条件 | 值 |
|--------|-----------|-------|
| 请求额外使用 / 切换到额外使用 / 添加资金 | 额外使用启用，账单访问可用 | `extra-usage` |
| 请求更多 | 团队/企业无账单访问，组织支出上限用尽 | `extra-usage` |
| 升级您的计划 | 不是 Max 20x，不是团队/企业，升级功能启用 | `upgrade` |
| 停止并等待限制重置 | 始终可用 | `cancel` |

**选项排序：**
- 由 `buyFirst` 功能标志（`tengu_jade_anvil_4`）控制
- 当 `true` 时：操作选项出现在取消之前
- 当 `false` 时：取消首先出现

**行为（`RateLimitOptionsMenu`）：**
1. 获取订阅类型、速率限制层级和账户信息
2. 检查 Claude AI 限制的超额状态和禁用原因
3. 基于订阅和账单状态动态构建可用选项
4. 渲染带 `Select` 组件的 `Dialog` 用于选项选择
5. 选择时：
   - `upgrade` — 启动升级命令 JSX
   - `extra-usage` — 启动额外使用命令 JSX
   - `cancel` — 以 `display: "skip"` 关闭
6. 记录菜单取消和每个选择类型的分析事件

**使用的状态检查：**
- `subscriptionType` — `"max"`、`"team"`、`"enterprise"` 等
- `rateLimitTier` — 例如 `"default_claude_max_20x"`
- `claudeAiLimits.overageStatus` — `"rejected"`、`"allowed_warning"` 等
- `claudeAiLimits.overageDisabledReason` — `"out_of_credits"`、`"org_level_disabled_until"` 等
- `hasClaudeAiBillingAccess()` — 用户是否可以管理账单
- `hasExtraUsageEnabled` — 额外使用是否已激活

**使用的组件：**
- `Dialog` — 模态容器，标题为 "What do you want to do?"
- `Select` — 带可见选项计数的选项选择器

---

## 组件依赖

设置命令依赖这些共享组件和工具：

| 组件/工具 | 来源 | 使用者 |
|-------------------|--------|---------|
| `PermissionRuleList` | `components/permissions/rules/PermissionRuleList.js` | `/permissions` |
| `SandboxSettings` | `components/sandbox/SandboxSettings.js` | `/sandbox` |
| `Dialog` | `components/design-system/Dialog.js` | `/fast`、`/rate-limit-options` |
| `Select` | `components/CustomSelect/select.js` | `/rate-limit-options` |
| `FastIcon`、`getFastIconString` | `components/FastIcon.js` | `/fast` |
| `useAppState`、`useSetAppState` | `state/AppState.js` | `/effort`、`/fast` |
| `useMainLoopModel` | `hooks/useMainLoopModel.js` | `/effort` |
| `SandboxManager` | `utils/sandbox/sandbox-adapter.js` | `/sandbox` |
| `isFastModeEnabled`、`isFastModeSupportedByModel`、`getFastModeModel` | `utils/fastMode.js` | `/fast` |
| `updateSettingsForSource` | `utils/settings/settings.js` | `/effort`、`/fast` |
| `getEffortEnvOverride`、`toPersistableEffort`、`isEffortLevel` | `utils/effort.js` | `/effort` |
| `useClaudeAiLimits` | `services/claudeAiLimitsHook.js` | `/rate-limit-options` |
| `getSubscriptionType`、`getRateLimitTier`、`getOauthAccountInfo` | `utils/auth.js` | `/rate-limit-options` |
| `extraUsage`、`upgrade` | `commands/extra-usage/`、`commands/upgrade/` | `/rate-limit-options` |
