# 权限组件

## 目的

权限组件系统渲染交互式权限请求对话框，允许用户批准、拒绝或配置各种操作的访问权限（文件系统、shell 命令、网络访问、沙箱操作等）。每种权限类型都有专门的组件，带有上下文适当的解释和操作按钮。

## 位置

`restored-src/src/components/permissions/`

## 关键组件

### 权限请求组件

| 组件 | 描述 |
|-----------|-------------|
| `BashPermissionRequest/` | Shell 命令执行权限 |
| `FileEdit.PermissionRequest/` | 文件编辑操作权限 |
| `FileWritePermissionRequest/` | 文件写入操作权限 |
| `FilesystemPermissionRequest/` | 文件系统访问权限 |
| `SandboxPermissionRequest.tsx` | 沙箱操作权限 |
| `SedEditPermissionRequest/` | Sed 编辑操作权限 |
| `NotebookEditPermissionRequest/` | Notebook 编辑权限 |
| `SkillPermissionRequest/` | 技能执行权限 |
| `WebFetchPermissionRequest/` | Web 获取/网络权限 |
| `AskUserQuestionPermissionRequest/` | 向用户提问权限 |
| `EnterPlanModePermissionRequest/` | 进入计划模式权限 |
| `ExitPlanModePermissionRequest/` | 退出计划模式权限 |
| `PowerShellPermissionRequest/` | PowerShell 执行权限 |
| `ComputerUseApproval/` | 计算机使用批准 |
| `FilePermissionDialog/` | 通用文件权限对话框 |

### 核心权限组件

| 组件 | 描述 |
|-----------|-------------|
| `PermissionPrompt.tsx` | 主要权限提示协调器 |
| `PermissionDialog.tsx` | 基础权限对话框包装器 |
| `PermissionRequest.tsx` | 权限请求容器 |
| `PermissionRequestTitle.tsx` | 权限请求标题显示 |
| `PermissionExplanation.tsx` | 权限解释文本 |
| `PermissionRuleExplanation.tsx` | 基于规则的权限解释 |
| `PermissionDecisionDebugInfo.tsx` | 权限决策调试信息 |
| `FallbackPermissionRequest.tsx` | 未知权限类型的回退 |

### Worker/队友组件

| 组件 | 描述 |
|-----------|-------------|
| `WorkerBadge.tsx` | Worker 身份徽章 |
| `WorkerPendingPermission.tsx` | Worker 的待处理权限指示器 |

### Hooks 和工具函数

| 导出 | 描述 |
|--------|-------------|
| `hooks.ts` | 权限相关的 React hooks |
| `utils.ts` | 权限工具函数 |
| `shellPermissionHelpers.tsx` | Shell 特定的权限帮助器 |
| `useShellPermissionFeedback.ts` | Shell 权限反馈 hook |
| `rules/` | 权限规则定义 |

## 依赖

### 内部依赖

- `../ink.js` — Ink 框架（Box、Text 等）
- `../design-system/` — Dialog、Pane 和其他原语
- `../TextInput.tsx` / `../VimTextInput.tsx` — 用于权限响应的文本输入
- `../ConfigurableShortcutHint.tsx` — 快捷键提示
- `../design-system/KeyboardShortcutHint.tsx` — 键盘快捷键显示
- `../design-system/Byline.tsx` — 操作提示
- `../../keybindings/useKeybinding.js` — 快捷键集成
- `../../utils/` — 权限工具、格式化、主题
- `../../hooks/` — 共享 hooks（useExitOnCtrlCDWithKeybindings）
- `../../bootstrap/` — 引导状态
- `../../permissions/` — 权限逻辑和类型

### 外部依赖

- `react` — React 19
- `figures` — Unicode 符号

## 权限架构

### 权限流程

```
工具执行请求
    ↓
权限系统 (确定是否需要权限)
    ↓
PermissionPrompt (协调器)
    ↓
[按类型的特定 PermissionRequest 组件]
    ↓
PermissionDialog (基础对话框包装器)
    ↓
PermissionExplanation (上下文 + 推理)
    ↓
操作按钮 (允许、拒绝、始终允许等)
    ↓
用户决策 → 工具执行或拒绝
```

### 权限请求模式

每个权限请求组件遵循此模式：

1. **标题**: 描述正在请求的操作
2. **解释**: 为什么需要权限及其允许的内容
3. **详情**: 操作特定详情（文件路径、命令、URL）
4. **操作**: 允许、拒绝、始终允许（取决于上下文）
5. **规则信息**: 显示权限规则是否适用及其原因

### 权限类型

- **文件操作**: 读取、写入、编辑特定文件或目录
- **Shell 命令**: 使用特定参数执行 bash/PowerShell 命令
- **网络访问**: 获取 URL、发出 HTTP 请求
- **沙箱操作**: 需要沙箱逃逸的操作
- **技能执行**: 使用特定权限运行技能
- **计划模式**: 进入/退出计划模式
- **计算机使用**: 屏幕/键盘/鼠标控制批准

### 决策选项

权限对话框通常提供：
- **允许一次**: 单次执行批准
- **拒绝**: 拒绝此请求
- **始终允许**: 创建永久规则（适用时）
- **会话允许**: 仅允许当前会话
- **记住决策**: 保存为未来类似请求的规则

### Shell 权限处理

Shell 权限有专门的处理：
- **命令显示**: 显示要执行的确切命令
- **参数高亮**: 高亮潜在危险的参数
- **反馈 hook**: `useShellPermissionFeedback` 跟踪用户决策
- **帮助器函数**: `shellPermissionHelpers.tsx` 提供常见 shell 权限逻辑

### Worker 权限

当 worker 代理需要权限时：
- **WorkerBadge**: 显示哪个 worker 正在请求权限
- **WorkerPendingPermission**: 待处理 worker 权限的视觉指示器
- **委托**: 权限可以从领导者委托给 workers

## 关键设计决策

1. **类型特定组件**: 每种权限类型都有自己的组件，以提供适当的上下文和解释
2. **规则透明度**: 显示权限规则适用或不适用的原因
3. **渐进式披露**: 首先显示基本信息，为高级用户提供可展开的详情
4. **调试信息**: `PermissionDecisionDebugInfo` 帮助诊断权限决策
5. **回退处理**: `FallbackPermissionRequest` 优雅地处理未知权限类型
6. **Worker 感知**: Worker 与领导者权限请求的独特渲染
7. **快捷键集成**: 权限对话框使用上下文感知的键盘快捷键
8. **会话持久化**: "始终允许" 创建存储在配置中的持久规则
