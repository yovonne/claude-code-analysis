# 组件概述

## 目的

Claude Code 的 React 组件库的完整目录，按领域组织。组件构建于 Ink 框架（自定义 React 终端渲染器）和设计系统（主题感知原语）之上。

## 位置

`restored-src/src/components/` — 主组件目录，包含领域子目录。

## 组件分类

### 布局组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `FullscreenLayout` | `components/FullscreenLayout.tsx` | 替代屏幕模式下的根布局；管理终端尺寸约束布局 |
| `SessionPreview` | `components/SessionPreview.tsx` | 会话预览覆盖层 |
| `VirtualMessageList` | `components/VirtualMessageList.tsx` | 虚拟化消息列表，用于大消息量的性能优化 |
| `Messages` | `components/Messages.tsx` | 编排消息渲染的消息容器 |
| `MessageRow` | `components/MessageRow.tsx` | 单个消息行包装器 |
| `MessageResponse` | `components/MessageResponse.tsx` | 响应消息显示 |
| `MessageModel` | `components/MessageModel.tsx` | 模型信息显示 |
| `MessageSelector` | `components/MessageSelector.tsx` | 消息倒带/选择 UI |
| `MessageTimestamp` | `components/MessageTimestamp.tsx` | 时间戳显示 |

### 输入组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `PromptInput/` | `components/PromptInput/` | 带自动完成和操作按钮的主聊天输入区 |
| `TextInput` | `components/TextInput.tsx` | 基本文本输入 |
| `BaseTextInput` | `components/BaseTextInput.tsx` | 带通用属性的基文本输入 |
| `VimTextInput` | `components/VimTextInput.tsx` | 支持 vim 模式的文本输入 |
| `SearchBox` | `components/SearchBox.tsx` | 带图标的搜索输入 |
| `LanguagePicker` | `components/LanguagePicker.tsx` | 语言选择下拉框 |
| `ModelPicker` | `components/ModelPicker.tsx` | 模型选择对话框 |
| `OutputStylePicker` | `components/OutputStylePicker.tsx` | 输出样式选择 |
| `CustomSelect/` | `components/CustomSelect/` | 可复用的选择/列表组件 |

### 对话框组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `BridgeDialog` | `components/BridgeDialog.tsx` | Claude Code Bridge 连接对话框 |
| `ExportDialog` | `components/ExportDialog.tsx` | 会话导出对话框 |
| `GlobalSearchDialog` | `components/GlobalSearchDialog.tsx` | 全局文件/命令搜索 |
| `QuickOpenDialog` | `components/QuickOpenDialog.tsx` | 快速文件打开对话框 |
| `HistorySearchDialog` | `components/HistorySearchDialog.tsx` | 命令历史搜索 (ctrl+r) |
| `MCPServerDialogCopy` | `components/MCPServerDialogCopy.tsx` | MCP 服务器配置 |
| `MCPServerApprovalDialog` | `components/MCPServerApprovalDialog.tsx` | MCP 服务器审批 |
| `MCPServerMultiselectDialog` | `components/MCPServerMultiselectDialog.tsx` | MCP 服务器多选 |
| `MCPServerDesktopImportDialog` | `components/MCPServerDesktopImportDialog.tsx` | 从桌面导入 MCP 服务器 |
| `WorkflowMultiselectDialog` | `components/WorkflowMultiselectDialog.tsx` | 工作流选择 |
| `TeleportRepoMismatchDialog` | `components/TeleportRepoMismatchDialog.tsx` | Teleport 仓库不匹配警告 |
| `RemoteEnvironmentDialog` | `components/RemoteEnvironmentDialog.tsx` | 远程环境配置 |
| `InvalidConfigDialog` | `components/InvalidConfigDialog.tsx` | 无效配置警告 |
| `InvalidSettingsDialog` | `components/InvalidSettingsDialog.tsx` | 无效设置警告 |
| `ChannelDowngradeDialog` | `components/ChannelDowngradeDialog.tsx` | 频道降级确认 |
| `DevChannelsDialog` | `components/DevChannelsDialog.tsx` | 开发频道选择 |
| `CostThresholdDialog` | `components/CostThresholdDialog.tsx` | 成本阈值警告 |
| `AutoModeOptInDialog` | `components/AutoModeOptInDialog.tsx` | 自动模式选择加入 |
| `BypassPermissionsModeDialog` | `components/BypassPermissionsModeDialog.tsx` | 绕过权限模式 |
| `ClaudeMdExternalIncludesDialog` | `components/ClaudeMdExternalIncludesDialog.tsx` | 外部 CLAUDE.md 包含 |
| `IdeAutoConnectDialog` | `components/IdeAutoConnectDialog.tsx` | IDE 自动连接入门 |
| `IdeOnboardingDialog` | `components/IdeOnboardingDialog.tsx` | IDE 入门 |
| `IdleReturnDialog` | `components/IdleReturnDialog.tsx` | 从空闲状态返回 |
| `WorktreeExitDialog` | `components/WorktreeExitDialog.tsx` | 工作树退出确认 |

### 状态和进度组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `Spinner/` | `components/Spinner/` | 旋转动画（详见单独文档） |
| `AgentProgressLine` | `components/AgentProgressLine.tsx` | 代理进度指示器 |
| `BashModeProgress` | `components/BashModeProgress.tsx` | Bash 模式进度 |
| `CoordinatorAgentStatus` | `components/CoordinatorAgentStatus.tsx` | 协调器代理状态 |
| `EffortIndicator` | `components/EffortIndicator.ts` | 努力程度指示器 |
| `EffortCallout` | `components/EffortCallout.tsx` | 努力程度提示 |
| `TeleportProgress` | `components/TeleportProgress.tsx` | Teleport 操作进度 |
| `ToolUseLoader` | `components/ToolUseLoader.tsx` | 工具执行加载指示器 |
| `FastIcon` | `components/FastIcon.tsx` | 快速模式指示器 |
| `StatusLine` | `components/StatusLine.tsx` | 状态行显示 |
| `StatusNotices` | `components/StatusNotices.tsx` | 状态通知 |
| `MemoryUsageIndicator` | `components/MemoryUsageIndicator.tsx` | 内存使用显示 |
| `IdeStatusIndicator` | `components/IdeStatusIndicator.tsx` | IDE 连接状态 |

### 入门和功能组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `Onboarding` | `components/Onboarding.tsx` | 首次运行入门流程 |
| `ClaudeInChromeOnboarding` | `components/ClaudeInChromeOnboarding.tsx` | Chrome 集成入门 |
| `DesktopHandoff` | `components/DesktopHandoff.tsx` | 桌面应用交接 |
| `DesktopUpsell/` | `components/DesktopUpsell/` | 桌面应用推广 |
| `PrBadge` | `components/PrBadge.tsx` | PR 徽章指示器 |
| `CtrlOToExpand` | `components/CtrlOToExpand.tsx` | Ctrl+O 展开提示 |
| `PressEnterToContinue` | `components/PressEnterToContinue.tsx` | 按 Enter 继续提示 |
| `RemoteCallout` | `components/RemoteCallout.tsx` | 远程环境提示 |
| `SessionBackgroundHint` | `components/SessionBackgroundHint.tsx` | 会话后台提示 |
| `ShowInIDEPrompt` | `components/ShowInIDEPrompt.tsx` | 在 IDE 中显示提示 |

### 开发调试组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `DevBar` | `components/DevBar.tsx` | 开发者工具栏 |
| `DiagnosticsDisplay` | `components/DiagnosticsDisplay.tsx` | 诊断信息显示 |
| `SentryErrorBoundary` | `components/SentryErrorBoundary.ts` | Sentry 错误边界包装器 |
| `KeybindingWarnings` | `components/KeybindingWarnings.tsx` | 快捷键冲突警告 |
| `ValidationErrorsList` | `components/ValidationErrorsList.tsx` | 验证错误列表 |

### 导航组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `TagTabs` | `components/TagTabs.tsx` | 基于标签的标签导航 |
| `TaskListV2` | `components/TaskListV2.tsx` | 任务列表显示 |
| `LogSelector` | `components/LogSelector.tsx` | 日志选择 UI |
| `TeammateViewHeader` | `components/TeammateViewHeader.tsx` | 队友视图头部 |

### 退出和流程组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `ExitFlow` | `components/ExitFlow.tsx` | 会话退出流程 |
| `InterruptedByUser` | `components/InterruptedByUser.tsx` | 用户中断显示 |
| `ResumeTask` | `components/ResumeTask.tsx` | 任务恢复提示 |
| `TeleportStash` | `components/TeleportStash.tsx` | Teleport 暂存显示 |
| `TeleportResumeWrapper` | `components/TeleportResumeWrapper.tsx` | Teleport 恢复包装器 |
| `OffscreenFreeze` | `components/OffscreenFreeze.tsx` | 屏幕外冻结指示器 |

### 主题和设置组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `ThemePicker` | `components/ThemePicker.tsx` | 主题选择 UI |
| `Settings/` | `components/Settings/` | 设置面板组件 |
| `ManagedSettingsSecurityDialog` | `components/ManagedSettingsSecurityDialog/` | 托管设置安全 |

### 反馈组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `Feedback` | `components/Feedback.tsx` | 反馈提交 |
| `FeedbackSurvey/` | `components/FeedbackSurvey/` | 反馈调查组件 |
| `SkillImprovementSurvey` | `components/SkillImprovementSurvey.tsx` | 技能改进调查 |

### 专用组件

| 组件 | 位置 | 描述 |
|-----------|----------|-------------|
| `ConsoleOAuthFlow` | `components/ConsoleOAuthFlow.tsx` | Console OAuth 认证流程 |
| `ApproveApiKey` | `components/ApproveApiKey.tsx` | API 密钥审批对话框 |
| `AwsAuthStatusBox` | `components/AwsAuthStatusBox.tsx` | AWS 认证状态 |
| `ContextVisualization` | `components/ContextVisualization.tsx` | 上下文窗口可视化 |
| `ContextSuggestions` | `components/ContextSuggestions.tsx` | 上下文建议 |
| `TokenWarning` | `components/TokenWarning.tsx` | Token 使用警告 |
| `Stats` | `components/Stats.tsx` | 会话统计 |
| `ThinkingToggle` | `components/ThinkingToggle.tsx` | 扩展思考切换 |
| `AutoUpdater` | `components/AutoUpdater.tsx` | 自动更新检查器 |
| `AutoUpdaterWrapper` | `components/AutoUpdaterWrapper.tsx` | 自动更新包装器 |
| `NativeAutoUpdater` | `components/NativeAutoUpdater.tsx` | 原生自动更新器 |
| `PackageManagerAutoUpdater` | `components/PackageManagerAutoUpdater.tsx` | 包管理器更新检查器 |
| `ClickableImageRef` | `components/ClickableImageRef.tsx` | 可点击图片引用 |
| `FilePathLink` | `components/FilePathLink.tsx` | 文件路径链接 |
| `ConfigurableShortcutHint` | `components/ConfigurableShortcutHint.tsx` | 可配置快捷方式显示 |
| `ScrollKeybindingHandler` | `components/ScrollKeybindingHandler.tsx` | 滚动快捷键处理器 |
| `SandboxViolationExpandedView` | `components/SandboxViolationExpandedView.tsx` | 沙箱违规详情 |
| `TeleportError` | `components/TeleportError.tsx` | Teleport 错误显示 |

## 领域子目录

| 目录 | 用途 |
|-----------|---------|
| `agents/` | 代理相关组件 |
| `design-system/` | 主题感知 UI 原语（Box、Text、Dialog、Tabs 等） |
| `diff/` | Diff 显示组件 |
| `grove/` | Grove 相关组件 |
| `hooks/` | 共享 React Hooks |
| `mcp/` | MCP 服务器 UI 组件 |
| `memory/` | 内存相关组件 |
| `messages/` | 消息类型组件（详见单独文档） |
| `permissions/` | 权限请求组件（详见单独文档） |
| `sandbox/` | 沙箱相关组件 |
| `shell/` | Shell 输出组件 |
| `skills/` | 技能相关组件 |
| `tasks/` | 任务相关组件 |
| `teams/` | 团队相关组件 |
| `ui/` | 通用 UI 组件 |
| `wizard/` | 向导流程组件 |
| `ClaudeCodeHint/` | Claude Code 提示组件 |
| `HelpV2/` | 帮助覆盖层 V2 |
| `HighlightedCode/` | 语法高亮代码显示 |
| `LogoV2/` | Logo V2 组件 |
| `LspRecommendation/` | LSP 推荐 UI |
| `Passes/` | Pass 相关组件 |
| `StructuredDiff/` | 结构化 Diff 显示 |
| `TrustDialog/` | 信任对话框组件 |

## 依赖

### 内部依赖

- `../ink.js` — Ink 框架（Box、Text、useInput、useTheme 等）
- `../keybindings/` — 快捷键系统（useKeybinding）
- `../utils/` — 工具函数（theme、format、platform 等）
- `../hooks/` — 共享 hooks
- `../bootstrap/` — 引导状态
- `../tasks/` — 任务类型和状态
- `../services/` — 服务层

### 外部依赖

- `react` — React 19
- `figures` — 终端 UI 的 Unicode 符号
- `strip-ansi` — ANSI 转义序列移除

## 关键设计模式

1. **主题感知原语**：所有 UI 组件使用设计系统中的 `ThemedBox` 和 `ThemedText`，接受主题键而非原始颜色
2. **快捷键集成**：对话框和交互组件使用 `useKeybinding` 实现上下文感知的键盘快捷键
3. **渐进式披露**：复杂对话框先显示基本信息，为高级用户提供可展开的详情
4. **虚拟渲染**：大列表（VirtualMessageList）使用视口裁剪提升性能
5. **动画隔离**：旋转组件将动画循环与父组件渲染分离，以最小化重渲染范围
6. **上下文提供者**：领域特定的上下文（TabsContext、TextHoverColorContext）用于跨组件协调
7. **功能标志**：组件根据 `bun:bundle` 功能标志和 Statsig 门控条件渲染
