# 权限工具

**目录：** `restored-src/src/utils/permissions/`

## 目的

实现工具执行的权限系统，包括规则解析、分类、自动模式 AI 决策、路径验证和拒绝跟踪。

## 核心类型

### PermissionRule
```typescript
{
  source: PermissionRuleSource
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue  // { toolName, ruleContent? }
}
```

### PermissionMode
- `default` - 标准权限提示
- `acceptEdits` - 自动接受工作目录中的文件编辑
- `plan` - 计划模式（无工具执行）
- `bypassPermissions` - 跳过权限提示
- `dontAsk` - 自动拒绝一切
- `auto` - AI 分类器决定（仅 ant）

## permissions.ts

### 主要入口点

- `hasPermissionsToUseTool(tool, input, context, assistantMessage, toolUseID)` - 主要权限检查函数
  - 步骤 1：检查拒绝规则、询问规则、工具实现检查、安全检查
  - 步骤 2a：检查绕过权限模式
  - 步骤 2b：检查始终允许规则
  - 步骤 3：将直通转换为询问
  - 自动模式集成：工具决策的 AI 分类器
  - 无头 agent 支持：非交互式上下文的 PermissionRequest hooks

### 规则提取

- `getAllowRules(context)` - 从所有源提取所有允许规则
- `getDenyRules(context)` - 提取所有拒绝规则
- `getAskRules(context)` - 提取所有询问规则
- `toolAlwaysAllowedRule(context, tool)` - 检查整个工具是否允许
- `getDenyRuleForTool(context, tool)` - 检查工具是否被拒绝
- `getRuleByContentsForTool(context, tool, behavior)` - 将规则内容映射到规则

### 自动模式集成

- 在自动模式时使用 AI 对工具操作进行分类
- 快速路径：acceptEdits 模式检查、安全工具允许列表
- 拒绝跟踪：3 次连续或 20 次总拒绝后回退到提示
- 无头模式：提示不可用时自动拒绝

### 权限规则编辑

- `deletePermissionRule()` - 从适当的目标删除规则
- `applyPermissionRulesToPermissionContext()` - 将规则应用到上下文

## permissionRuleParser.ts

### 字符串到规则值转换

- `permissionRuleValueFromString(ruleString)` - 解析 "Tool(content)" 格式
  - 处理转义括号
  - 规范化遗留工具名称（Task 到 Agent、KillShell 到 TaskStop）
  - 空或通配符内容作为工具范围规则处理

- `permissionRuleValueToString(ruleValue)` - 转换回字符串格式
  - 转义内容中的括号

- `escapeRuleContent(content)` / `unescapeRuleContent(content)` - 转义工具

## permissionsLoader.ts

### 从磁盘加载规则

- `loadAllPermissionRulesFromDisk()` - 从所有启用的源加载规则
  - 遵循 allowManagedPermissionRulesOnly 标志
- `getPermissionRulesForSource(source)` - 从特定源加载规则
- `addPermissionRulesToSettings()` - 添加规则到设置文件
- `deletePermissionRuleFromSettings()` - 从设置中删除规则
- `shouldAllowManagedPermissionRulesOnly()` - 检查是否仅应使用管理的规则
- `shouldShowAlwaysAllowOptions()` - 上面的反义

## PermissionMode.ts

### 模式配置

- `permissionModeTitle(mode)` - 人类可读标题
- `permissionModeShortTitle(mode)` - UI 短标题
- `permissionModeSymbol(mode)` - 显示符号
- `permissionModeFromString(str)` - 将字符串解析为模式枚举
- `toExternalPermissionMode(mode)` - 转换为外部兼容模式
- `isExternalPermissionMode(mode)` - 类型守卫

## permissionExplainer.ts

### AI 驱动的解释

- `generatePermissionExplanation()` - 使用 Haiku 解释命令
  - 返回风险级别（LOW/MEDIUM/HIGH）、解释、推理、风险
  - 使用强制工具选择进行结构化输出
  - 提取对话上下文用于推理
- `isPermissionExplainerEnabled()` - 检查全局配置标志

## pathValidation.ts

### 路径权限检查

- `isPathAllowed(resolvedPath, context, operationType)` - 核心路径权限检查
  - 步骤 1：检查拒绝规则
  - 步骤 2：检查内部可编辑路径
  - 步骤 2.5：安全检查（危险文件、Claude 配置）
  - 步骤 3：检查工作目录加 acceptEdits 模式
  - 步骤 3.5：检查内部可读路径
  - 步骤 3.7：检查沙箱写入允许列表
  - 步骤 4：检查允许规则

- `validatePath(path, cwd, context, operationType)` - 带波浪号/glob 处理的完整验证
- `validateGlobPattern(cleanPath, cwd, context, operationType)` - 验证 glob 基础目录
- `isDangerousRemovalPath(resolvedPath)` - 检查危险 rm 目标（/、~、驱动器根）
- `expandTilde(path)` - 将波浪号展开到主目录
- `getGlobBaseDirectory(path)` - 从 glob 模式提取基础目录
- `isPathInSandboxWriteAllowlist(resolvedPath)` - 检查沙箱写入允许列表

## getNextPermissionMode.ts

### 模式循环

- `getNextPermissionMode(context)` - 确定 Shift+Tab 循环中的下一个模式
- `cyclePermissionMode(context)` - 计算下一个模式并准备上下文

## dangerousPatterns.ts

### 危险命令模式

- `DANGEROUS_BASH_PATTERNS` - 危险 bash 命令前缀列表
  - 跨平台：python、node、npx、bash、ssh 等
  - Unix 特定：zsh、fish、eval、exec、sudo 等
  - 仅 ant：fa run、coo、gh、curl、kubectl、aws、gcloud

## autoModeState.ts

### 状态管理

- `setAutoModeActive(active)` / `isAutoModeActive()` - 活动状态标志
- `setAutoModeFlagCli(passed)` / `getAutoModeFlagCli()` - CLI 标志状态
- `setAutoModeCircuitBroken(broken)` / `isAutoModeCircuitBroken()` - 断路器

## denialTracking.ts

### 拒绝限制

- `createDenialTrackingState()` - 初始状态（0 连续、0 总计）
- `recordDenial(state)` - 增加两个计数器
- `recordSuccess(state)` - 重置连续计数器
- `shouldFallbackToPrompting(state)` - 如果 3 次连续或 20 次总计拒绝则返回 true

## bypassPermissionsKillswitch.ts

### 运行时致命开关

- `checkAndDisableBypassPermissionsIfNeeded()` - 基于 Statsig 门控禁用旁路
- `checkAndDisableAutoModeIfNeeded()` - 基于门控/设置禁用自动模式
- 用于组件集成的 React hooks

## filesystem.ts

### 文件权限实现

- `checkWritePermissionForTool()` - 工具级写权限检查
- `checkReadPermissionForTool()` - 工具级读权限检查
- `pathInWorkingPath()` - 检查路径是否在工作目录内
- `pathInAllowedWorkingPath()` - 带附加目录检查
- `matchingRuleForInput()` - 查找路径输入的匹配规则
- `checkEditableInternalPath()` - 检查内部可编辑路径
- `checkReadableInternalPath()` - 检查内部可读路径
- `checkPathSafetyForAutoEdit()` - 全面的安全验证

## 类型定义文件

### PermissionResult.ts / PermissionRule.ts / PermissionUpdate.ts / PermissionUpdateSchema.ts

- 权限结果、规则和更新的模式定义
- `applyPermissionUpdate()` / `applyPermissionUpdates()` - 将更新应用到上下文
- `persistPermissionUpdates()` - 将更新持久化到设置

## AI 分类（特性门控）

### classifierDecision.ts / classifierShared.ts / yoloClassifier.ts / bashClassifier.ts

- `classifyYoloAction()` - 为自动模式分类操作
- `classifyBashCommand()` - 为权限规则分类 bash 命令
- `formatActionForClassifier()` - 格式化分类器输入的操作

## 其他文件

- `shadowedRuleDetection.ts` - 检测重叠的权限规则
- `permissionSetup.ts` - 模式管理和转换
- `PermissionPromptToolResultSchema.ts` - 提示工具结果模式
