# 配置命令

## 概述

配置命令提供用于管理 Claude Code 持久设置、环境变量、模型选择、主题、输出样式和隐私控制的交互界面。这些命令在终端中渲染基于 JSX 的 UI 组件。

---

## `/config`（别名：`/settings`）

**目的：** 打开用于管理所有设置的主配置面板。

**来源：** `restored-src/src/commands/config/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带 `settings` 别名的命令注册 |
| `config.tsx` | 渲染 `Settings` 组件的入口点 |

**行为：**
- 通过 `Settings` 组件打开交互式设置面板
- 默认为"Config"标签页
- 接受 `onClose` 回调以关闭

**注册：**
```typescript
{
  aliases: ['settings'],
  type: 'local-jsx',
  name: 'config',
  description: 'Open config panel',
  load: () => import('./config.js'),
}
```

---

## `/env`

**目的：** 环境变量管理。

**来源：** `restored-src/src/commands/env/`

| 文件 | 目的 |
|------|---------|
| `index.js` | 存根实现（禁用并隐藏） |

**状态：** 此命令当前是一个存根 — `isEnabled: () => false` 和 `isHidden: true`。它在当前代码库中不工作。

---

## `/model`

**目的：** 通过交互式选择器或直接参数设置 Claude Code 的 AI 模型。

**来源：** `restored-src/src/commands/model/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 带动态描述的命令注册，显示当前模型 |
| `model.tsx` | 模型选择逻辑、验证和选择器 UI |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'model',
  description: `Set the AI model for Claude Code (currently ${renderModelName(getMainLoopModel())})`,
  argumentHint: '[model]',
  immediate: shouldInferenceConfigCommandBeImmediate(),
  load: () => import('./model.js'),
}
```

**使用模式：**

| 调用 | 行为 |
|------------|---------|
| `/model` | 打开 `ModelPicker` 交互式 UI |
| `/model [modelName]` | 直接设置模型，不使用 UI |
| `/model default` | 重置为默认模型 |
| `/model --help` | 显示内联帮助文本 |
| `/model -h` / `/model help` | 显示使用信息 |

**模型选择流程（`SetModelAndClose`）：**
1. 检查组织策略是否允许模型（`isModelAllowed`）
2. 检查 Opus 和 Sonnet 模型的 1M 上下文访问
3. 跳过默认模型（`null`）的验证
4. 通过 `isKnownAlias` 识别已知别名
5. 通过 `validateModel` 验证自定义模型
6. 更新 app 状态（`mainLoopModel`，清除 `mainLoopModelForSession`）
7. 处理快速模式兼容性 — 如果新模型不支持则自动禁用
8. 为额外使用模型显示计费通知

**模型选择器流程（`ModelPickerWrapper`）：**
- 使用当前模型和会话覆盖渲染 `ModelPicker` 组件
- 选择时：更新模型，处理快速模式切换，记录分析
- 取消时：显示当前模型并关闭
- 显示快速模式通知（如果适用）

**显示逻辑（`ShowModelAndClose`）：**
- 显示当前模型和努力级别
- 如果存在会话覆盖（来自计划模式），显示会话和基础模型

**模型验证助手：**
- `isKnownAlias(model)` — 对照 `MODEL_ALIASES` 检查
- `isOpus1mUnavailable(model)` — 检查 Opus 4.6 1M 上下文访问
- `isSonnet1mUnavailable(model)` — 检查 Sonnet 4.6 1M 上下文访问
- `renderModelLabel(model)` — 格式化带"(default)"后缀的模型名称

---

## `/theme`

**目的：** 通过交互式选择器更改终端主题。

**来源：** `restored-src/src/commands/theme/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册 |
| `theme.tsx` | 主题选择器 UI 包装器 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'theme',
  description: 'Change the theme',
  load: () => import('./theme.js'),
}
```

**行为：**
- 在带 `color="permission"` 的 `Pane` 组件内渲染 `ThemePicker`
- 选择主题时：通过 `setTheme()` 应用主题并确认
- 取消时：关闭并显示"Theme picker dismissed"消息
- 使用 `skipExitHandling={true}` 管理自己的退出行为

---

## `/output-style`

**目的：** 已弃用 — 用户应改用 `/config`。

**来源：** `restored-src/src/commands/output-style/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册（隐藏） |
| `output-style.tsx` | 弃用通知 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'output-style',
  description: 'Deprecated: use /config to change output style',
  isHidden: true,
  load: () => import('./output-style.js'),
}
```

**行为：**
- 显示弃用消息，引导用户使用 `/config` 或设置文件
- 更改在下次会话时生效

---

## `/privacy-settings`

**目的：** 查看和更新隐私设置（Grove 数据共享控制）。

**来源：** `restored-src/src/commands/privacy-settings/`

| 文件 | 目的 |
|------|---------|
| `index.ts` | 命令注册（仅限消费者订阅者） |
| `privacy-settings.tsx` | 隐私设置对话框逻辑 |

**注册：**
```typescript
{
  type: 'local-jsx',
  name: 'privacy-settings',
  description: 'View and update your privacy settings',
  isEnabled: () => isConsumerSubscriber(),
  load: () => import('./privacy-settings.js'),
}
```

**可用性：** 仅对消费者订阅者（`isConsumerSubscriber()`）启用。

**行为：**
1. 检查用户是否有资格使用 Grove（`isQualifiedForGrove`）
2. 如果没有资格，显示指向 `https://claude.ai/settings/data-privacy-controls` 的回退消息
3. 并行获取当前 Grove 设置和通知配置
4. 如果用户已接受条款（`grove_enabled !== null`），直接显示 `PrivacySettingsDialog`
5. 如果没有，显示 `GroveDialog` 用于首次条款接受
6. 完成后，获取更新后的设置并报告新状态
7. 设置更改时记录 `tengu_grove_policy_toggled` 分析事件

**使用的组件：**
- `GroveDialog` — 新用户的条款接受流程
- `PrivacySettingsDialog` — 回访用户的管理设置

---

## 组件依赖

配置命令依赖这些共享组件和工具：

| 组件/工具 | 来源 | 使用者 |
|-------------------|--------|---------|
| `Settings` | `components/Settings/Settings.js` | `/config` |
| `ModelPicker` | `components/ModelPicker.js` | `/model` |
| `ThemePicker` | `components/ThemePicker.js` | `/theme` |
| `Pane` | `components/design-system/Pane.js` | `/theme` |
| `GroveDialog`, `PrivacySettingsDialog` | `components/grove/Grove.js` | `/privacy-settings` |
| `useAppState`, `useSetAppState` | `state/AppState.js` | `/model`, `/privacy-settings` |
| `validateModel` | `utils/model/validateModel.js` | `/model` |
| `MODEL_ALIASES` | `utils/model/aliases.js` | `/model` |
| `isFastModeEnabled`, `isFastModeSupportedByModel` | `utils/fastMode.js` | `/model` |
| `getGroveSettings`, `isQualifiedForGrove` | `services/api/grove.js` | `/privacy-settings` |
