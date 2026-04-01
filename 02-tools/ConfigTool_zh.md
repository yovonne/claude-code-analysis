# ConfigTool

## 目的

ConfigTool 为 Claude 提供在运行时读取和修改自身配置设置的能力。它支持全局设置（存储在 `~/.claude.json`）和项目设置（存储在 `.claude/settings.json`）。该工具处理类型强制转换、异步验证（例如模型 API 检查）、功能标志门控设置（语音、bridge 模式、Kairos）以及用于实时 UI 效果的即时 AppState 同步。读取设置是自动允许的；写入需要用户权限。

## 位置

- `restored-src/src/tools/ConfigTool/ConfigTool.ts` — 主工具定义和读/写逻辑（468 行）
- `restored-src/src/tools/ConfigTool/prompt.ts` — 从设置注册表动态生成提示（94 行）
- `restored-src/src/tools/ConfigTool/supportedSettings.ts` — 带类型、验证和选项的设置注册表（212 行）
- `restored-src/src/tools/ConfigTool/constants.ts` — 工具名称常量（2 行）
- `restored-src/src/tools/ConfigTool/UI.tsx` — 终端 UI 渲染（38 行）

## 关键导出

### 从 ConfigTool.ts

| 导出 | 描述 |
|--------|-------------|
| `ConfigTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | Zod 推断的输入类型：`{ setting, value? }` |
| `Output` | Zod 推断的输出类型：`{ success, operation?, setting?, value?, previousValue?, newValue?, error? }` |

### 从 supportedSettings.ts

| 导出 | 描述 |
|--------|-------------|
| `SUPPORTED_SETTINGS` | 所有可配置设置的注册表，带类型、来源、描述和验证 |
| `isSupported(key)` | 检查设置键是否在注册表中 |
| `getConfig(key)` | 返回设置的 SettingConfig |
| `getAllKeys()` | 返回所有设置键 |
| `getOptionsForSetting(key)` | 返回设置的有效选项值 |
| `getPath(key)` | 返回设置的可嵌套路径数组 |
| `SettingConfig` | 类型：`{ source, type, description, path?, options?, getOptions?, appStateKey?, validateOnWrite?, formatOnRead? }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  setting: string,   // 必需：设置键（例如 "theme"、"model"、"permissions.defaultMode"）
  value?: string | boolean | number,  // 可选：新值。省略以获取当前值。
}
```

### 输出 Schema

```typescript
{
  success: boolean,
  operation?: 'get' | 'set',
  setting?: string,
  value?: unknown,           // 当前值（用于获取操作）
  previousValue?: unknown,   // 更改前的值（用于设置操作）
  newValue?: unknown,        // 更改后的值（用于设置操作）
  error?: string,            // 如果操作失败则返回错误消息
}
```

## 支持的设置

### 全局设置（存储在 `~/.claude.json`）

| 设置 | 类型 | 选项 | 描述 |
|---------|------|---------|-------------|
| `theme` | string | 主题名称（或带 AUTO_THEME 的主题设置） | UI 颜色主题 |
| `editorMode` | string | `EDITOR_MODES` | 键绑定模式（例如 emacs、vim） |
| `verbose` | boolean | true/false | 显示详细调试输出 |
| `preferredNotifChannel` | string | 通知通道 | 首选通知通道 |
| `autoCompactEnabled` | boolean | true/false | 上下文满时自动压缩 |
| `fileCheckpointingEnabled` | boolean | true/false | 启用文件检查点以进行代码回滚 |
| `showTurnDuration` | boolean | true/false | 在响应后显示轮次持续时间 |
| `terminalProgressBarEnabled` | boolean | true/false | 显示 OSC 9;4 进度指示器 |
| `todoFeatureEnabled` | boolean | true/false | 启用待办/任务跟踪 |
| `teammateMode` | string | `TEAMMATE_MODES` | 如何生成队友（tmux、进程内、自动） |

### 项目设置（存储在 `.claude/settings.json`）

| 设置 | 类型 | 选项 | 描述 |
|---------|------|---------|-------------|
| `autoMemoryEnabled` | boolean | true/false | 启用自动内存 |
| `autoDreamEnabled` | boolean | true/false | 启用后台内存整合 |
| `model` | string | 模型选项（sonnet、opus、haiku 等） | 覆盖默认模型 |
| `alwaysThinkingEnabled` | boolean | true/false | 启用扩展思考 |
| `permissions.defaultMode` | string | `default`、`plan`、`acceptEdits`、`dontAsk`、`auto` | 默认权限模式 |
| `language` | string | 任何语言名称 | 响应和语音听写的首选语言 |

### 功能标志门控设置

| 设置 | 功能标志 | 类型 | 描述 |
|---------|-------------|------|-------------|
| `classifierPermissionsEnabled` | `TRANSCRIPT_CLASSIFIER`（仅限 ant） | boolean | 启用基于 AI 的 Bash 权限分类 |
| `voiceEnabled` | `VOICE_MODE` | boolean | 启用语音听写（按住说话） |
| `remoteControlAtStartup` | `BRIDGE_MODE` | boolean | 为所有会话启用远程控制 |
| `taskCompleteNotifEnabled` | `KAIROS` 或 `KAIROS_PUSH_NOTIFICATION` | boolean | 在完成后空闲时推送到移动设备 |
| `inputNeededNotifEnabled` | `KAIROS` 或 `KAIROS_PUSH_NOTIFICATION` | boolean | 在等待输入时推送到移动设备 |
| `agentPushNotifEnabled` | `KAIROS` 或 `KAIROS_PUSH_NOTIFICATION` | boolean | 允许 Claude 在认为适当时推送 |

## 读/写管道

### 执行流程

```
1. 接收输入
   - 模型返回带 { setting, value? } 的 Config tool_use

2. 支持检查
   - isSupported(setting)：对照 SUPPORTED_SETTINGS 注册表检查
   - 语音设置：通过 GrowthBook 功能标志的附加运行时门控
   - 未知设置 → 返回错误

3. 获取操作（value 是 undefined）
   a. 查找设置的配置
   b. 从来源获取当前值（全局或设置）
   c. 如果定义则应用 formatOnRead（例如 model null → "default"）
   d. 返回 { success: true, operation: 'get', setting, value }

4. 设置操作（提供了 value）
   a. 查找设置的配置
   b. 处理"default"特殊情况（remoteControlAtStartup）
   c. 强制转换和验证类型（布尔字符串 → true/false）
   d. 如果定义则检查允许的选项
   e. 如果定义则运行异步验证（例如 validateModel API 检查）
   f. 语音模式的预检检查：
      - GrowthBook 启用检查
      - Anthropic 认证检查
      - 语音流可用性
      - 录制可用性
      - 语音依赖（arecord 工具）
      - 麦克风权限
   g. 写入存储（全局配置或设置文件）
   h. 通知更改检测器（用于语音设置）
   i. 如果定义 appStateKey 则同步到 AppState
   j. 记录分析事件
   k. 返回 { success: true, operation: 'set', setting, previousValue, newValue }
```

### 存储来源

| 来源 | 存储 | 访问 |
|--------|---------|--------|
| `global` | 通过 `getGlobalConfig()` / `saveGlobalConfig()` 存储在 `~/.claude.json` | 单级键 |
| `settings` | 通过 `updateSettingsForSource()` 存储在 `.claude/settings.json` | 支持嵌套路径 |

### 类型强制转换

布尔设置接受字符串值并强制转换它们：

| 输入 | 结果 |
|-------|--------|
| `"true"`（不区分大小写） | `true` |
| `"false"`（不区分大小写） | `false` |
| 其他字符串 | 错误："requires true or false" |
| 实际布尔值 | 原样传递 |

### 异步验证

某些设置有 `validateOnWrite` 函数，运行异步验证：

| 设置 | 验证 |
|---------|-----------|
| `model` | `validateModel()` — 检查模型 API 可用性 |

### AppState 同步

带 `appStateKey` 的设置在写入后立即同步到 AppState：

| 设置 | AppState 键 |
|---------|-------------|
| `verbose` | `verbose` |
| `model` | `mainLoopModel` |
| `alwaysThinkingEnabled` | `thinkingEnabled` |

`remoteControlAtStartup` 设置使用自定义同步路径，因为配置键与 AppState 字段名不同。

## 语音模式预检检查

启用 `voiceEnabled` 时，运行综合验证链：

```
1. GrowthBook 语音模式启用？
   → 否：错误 "Voice mode requires Claude.ai account" 或 "not available"

2. 语音流可用（isVoiceStreamAvailable）？
   → 否：错误 "requires Claude.ai account"

3. 录制可用（checkRecordingAvailability）？
   → 否：带原因的错误

4. 语音依赖（checkVoiceDependencies）？
   → 否：带可用安装命令的错误

5. 麦克风权限（requestMicrophonePermission）？
   → 否：平台特定指导（Windows/Linux/macOS）
```

## 权限处理

| 操作 | 权限决策 |
|-----------|-------------------|
| GET（省略 value） | 自动允许 |
| SET（提供了 value） | 询问（用户必须批准） |

## 提示生成

提示从设置注册表动态生成：

1. 迭代 `SUPPORTED_SETTINGS`
2. 跳过 `model`（有其自己的带动态选项的部分）
3. 当标志关闭时跳过功能标志门控设置
4. 按来源分组（全局 vs 项目）
5. 从 `getModelOptions()` 生成带当前可用选项的模型部分
6. 包含使用示例

## 错误处理

| 错误条件 | 响应 |
|----------------|----------|
| 未知设置 | `{ success: false, error: 'Unknown setting: "<setting>"' }` |
| 无效布尔值 | `{ success: false, error: '<setting> requires true or false.' }` |
| 无效选项值 | `{ success: false, error: 'Invalid value "<value>". Options: <list>' }` |
| 异步验证失败 | `{ success: false, error: <validation error> }` |
| 语音预检失败 | `{ success: false, error: <specific voice error> }` |
| 写入错误 | `{ success: false, error: <error message> }` |

## 配置

### 功能标志

| 标志 | 门控设置 |
|------|---------------|
| `VOICE_MODE` | `voiceEnabled` |
| `BRIDGE_MODE` | `remoteControlAtStartup` |
| `KAIROS` / `KAIROS_PUSH_NOTIFICATION` | `taskCompleteNotifEnabled`、`inputNeededNotifEnabled`、`agentPushNotifEnabled` |
| `TRANSCRIPT_CLASSIFIER`（仅限 ant） | `classifierPermissionsEnabled` |
| `AUTO_THEME` | 扩展主题选项 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/config.js` | 全局配置获取/保存 |
| `utils/settings/settings.js` | 设置管理 |
| `utils/model/modelOptions.js` | 模型选项列表 |
| `utils/model/validateModel.js` | 模型验证 |
| `utils/theme.js` | 主题常量 |
| `utils/configConstants.js` | 编辑器模式、通知通道、队友模式 |
| `voice/voiceModeEnabled.js` | 语音功能门控 |
| `services/voiceStreamSTT.js` | 语音流可用性 |
| `services/voice.js` | 录制和依赖检查 |
| `utils/auth.js` | 认证状态检查 |
| `services/analytics/` | 事件日志 |

### 外部

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入/输出 schema 验证 |
| `react` | UI 渲染 |
| `bun:bundle` | 功能标志 API |

## 数据流

```
Model Request
    │
    ▼
ConfigTool Input { setting, value? }
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 支持检查                                                   │
│ - isSupported(setting)                                          │
│ - 功能标志门控（语音、bridge、kairos）                    │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 权限检查                                                │
│  GET → 自动允许                                               │
│  SET → 询问用户                                             │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 操作                                                       │
│                                                                 │
│  GET：                                                           │
│    - 从来源读取（全局/设置）                         │
│    - 应用 formatOnRead                                         │
│    - 返回值                                               │
│                                                                 │
│  SET：                                                           │
│    - 强制转换类型（布尔字符串）                              │
│    - 验证选项                                  │
│    - 异步验证（模型 API 检查）                        │
│    - 语音预检检查（如果 voiceEnabled）                 │
│    - 写入存储                                          │
│    - 同步到 AppState                                          │
│    - 记录分析事件                                       │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Output { success, operation, setting, value/previousValue/newValue, error? }
    │
    ▼
Model Response (tool_result block)
```
