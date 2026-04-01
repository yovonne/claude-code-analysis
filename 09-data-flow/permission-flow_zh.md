# 权限流

## 目的

追踪工具执行权限如何被评估 — 从初始工具调用，通过规则匹配、分类器检查、用户提示，到最终的允许/拒绝决定。

## 位置

主要来源：
- `restored-src/src/types/permissions.ts` — 权限类型定义
- `restored-src/src/Tool.ts` — `ToolPermissionContext`、`checkPermissions`、`PermissionResult`
- `restored-src/src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx` — 权限提示示例
- `restored-src/src/utils/permissions/permissionSetup.ts` — 初始化
- `restored-src/src/utils/permissions/permissions.ts` — 核心权限评估
- `restored-src/src/utils/permissions/denialTracking.ts` — 拒绝计数
- `restored-src/src/hooks/useCanUseTool.ts` — 权限决策入口点

---

## 权限模式

| 模式 | 行为 |
|------|----------|
| `default` | 标准权限检查与规则 |
| `bypassPermissions` | 跳过所有权限提示（危险） |
| `acceptEdits` | 自动批准文件编辑，提示破坏性操作 |
| `dontAsk` | 自动批准所有非破坏性操作 |
| `plan` | 计划模式 — 无工具执行，仅计划 |
| `auto` | AI 分类器评估每个命令的安全性 |
| `bubble` | 权限通过外部 bubble 处理（远程/桥接） |

---

## 权限规则系统

### 规则来源（按优先级排序）

1. `cliArg` — 命令行标志（`--allowedTool`）
2. `session` — 会话范围的规则（从计划模式退出）
3. `flagSettings` — `--settings` 标志覆盖
4. `policySettings` — 企业策略限制
5. `userSettings` — `~/.claude/settings.json`
6. `projectSettings` — 项目根目录的 `.claude/settings.json`
7. `localSettings` — `.claude/settings.local.json`
8. `command` — 每命令内置权限

### 规则结构

```typescript
type PermissionRule = {
  source: PermissionRuleSource    // 规则来自哪里
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string              // 例如 "Bash", "FileWrite"
    ruleContent?: string          // 例如 "ls *"（模式匹配）
  }
}
```

### ToolPermissionContext 中的规则类别

```
ToolPermissionContext {
  mode: PermissionMode
  alwaysAllowRules: ToolPermissionRulesBySource   // 自动批准
  alwaysDenyRules: ToolPermissionRulesBySource    // 始终阻止
  alwaysAskRules: ToolPermissionRulesBySource     // 始终提示
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource  // 在 auto 模式中移除
}
```

---

## 完整权限评估流程

### 阶段 1：工具调用初始化

```
API 返回 tool_use 块
  │
  ▼
canUseTool(tool, input, context, assistantMessage, toolUseID)
  │
  ├─► 1. validateInput(tool, input)
  │     │
  │     ├─► tool.validateInput(input, context)
  │     │    └─► 返回 { result: true } 或 { result: false, message, errorCode }
  │     │
  │     └─► 如果验证失败 → 拒绝并显示错误消息
  │
  └─► 2. checkPermissions(tool, input, context)
```

### 阶段 2：权限决策链

```
checkPermissions(tool, input, context)
  │
  ├─► A. 工具特定的 checkPermissions()
  │     │
  │     ├─► 每个工具可以覆盖 checkPermissions()
  │     ├─► 默认：{ behavior: 'allow', updatedInput }
  │     ├─► AskUserQuestionTool: { behavior: 'ask', message: 'Answer questions?' }
  │     └─► BashTool: 复杂命令分析
  │
  ├─► B. 钩子评估（PreToolUse 钩子）
  │     │
  │     ├─► 按顺序执行 PreToolUse 钩子
  │     ├─► 钩子可以：allow、deny、ask 或修改输入
  │     └─► 第一个有决定性的钩子获胜
  │
  ├─► C. 权限规则匹配
  │     │
  │     ├─► 检查 alwaysDenyRules（工具名称 + 模式）
  │     ├─► 检查 alwaysAllowRules（工具名称 + 模式）
  │     ├─► 检查 alwaysAskRules（工具名称 + 模式）
  │     └─► 应用基于模式的默认值
  │
  ├─► D. 基于模式的评估
  │     │
  │     ├─► bypassPermissions → 允许所有
  │     ├─► dontAsk → 允许非破坏性
  │     ├─► acceptEdits → 允许文件编辑，提示 Bash/Write
  │     ├─► plan → 拒绝所有工具执行
  │     └─► auto → 分类器评估（见下文）
  │
  ├─► E. 安全检查
  │     │
  │     ├─► 工作目录验证
  │     ├─► 沙箱命令排除
  │     ├─► Windows 路径绕过检测
  │     └─► 跨机器桥接消息验证
  │
  └─► F. 最终决定
        │
        ├─► 'allow' → 执行工具
        ├─► 'deny' → 向 API 返回错误
        └─► 'ask' → 显示权限对话框
```

### 阶段 3：Auto 模式分类器

```
Auto 模式启用？
  │
  ▼
构建分类器 transcript
  │
  ├─► 系统提示（分类器指令）
  ├─► 工具调用描述（toAutoClassifierInput）
  ├─► 最近的用户提示
  └─► 对话上下文
  │
  ▼
调用分类器 API（快速阶段）
  │
  ├─► 如果快速阶段有信心 → 返回决定
  └─► 如果不确定 → 思考阶段（更深层分析）
  │
  ▼
ClassifierResult {
  matches: boolean
  matchedDescription?: string
  confidence: 'high' | 'medium' | 'low'
  reason: string
}
  │
  ▼
YoloClassifierResult {
  shouldBlock: boolean
  reason: string
  thinking?: string
  unavailable?: boolean
  transcriptTooLong?: boolean
  model: string
  usage?: ClassifierUsage
  stage?: 'fast' | 'thinking'
}
  │
  ├─► shouldBlock === false → 允许
  ├─► shouldBlock === true → 拒绝
  └─► unavailable → 回退到正常提示
```

### 阶段 4：用户权限对话框

```
决定 === 'ask'
  │
  ▼
构建权限提示
  │
  ├─► tool.userFacingName(input)
  ├─► tool.getActivityDescription(input)
  ├─► 风险评估（LOW/MEDIUM/HIGH）
  ├─► 权限解释
  └─► 建议更新（保存为 allow/deny 规则）
  │
  ▼
显示权限对话框（REPL UI）
  │
  ├─► 用户选择：Allow / Allow & Remember / Deny / Deny & Remember
  │
  ├─► Allow → 执行工具
  ├─► Allow & Remember → 添加到 alwaysAllowRules
  ├─► Deny → 向 API 返回错误
  └─► Deny & Remember → 添加到 alwaysDenyRules
  │
  ▼
跟踪拒绝计数（用于分类器回退）
  │
  └─► 如果拒绝超出阈值 → 下次强制提示
```

---

## 权限决策类型

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?; decisionReason?; toolUseID? }
  | { behavior: 'deny'; message: string; decisionReason: PermissionDecisionReason }
  | { behavior: 'ask'; message: string; suggestions?: PermissionUpdate[]; blockedPath? }
  | { behavior: 'passthrough'; message: string; pendingClassifierCheck? }

type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'workingDir'; reason: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'other'; reason: string }
```

---

## AskUserQuestionTool 权限流（示例）

```
模型调用 AskUserQuestion 工具
  │
  ▼
validateInput()
  │
  ├─► 检查 HTML 预览有效性（如果 previewFormat === 'html'）
  ├─► 验证无 <html>、<body>、<!DOCTYPE>、<script>、<style>
  └─► 返回 { result: true/false, message, errorCode }
  │
  ▼
checkPermissions()
  │
  └─► 始终返回 { behavior: 'ask', message: 'Answer questions?' }
      （此工具始终需要用户交互）
  │
  ▼
权限对话框显示：
  ├─► 问题标题（chips）
  ├─► 带标签和描述的选项
  ├─► 可选预览内容（markdown 或 HTML）
  └─► "Other" 选项用于自定义输入
  │
  ▼
用户响应 → 记录答案
  │
  └─► ToolResult: { questions, answers, annotations? }
```

---

## 拒绝跟踪

```
DenialTrackingState {
  count: number          // 连续拒绝数
  threshold: number      // 何时强制提示
  lastToolName: string   // 上次拒绝的工具
}
```

- 每次拒绝递增计数器
- 超出阈值时 → 即使在 auto 模式也强制权限提示
- 成功工具执行时重置
- 用作分类器模式（YOLO、headless）的安全回退

---

## 集成点

| 组件 | 在权限流中的角色 |
|-----------|------------------------|
| `Tool.checkPermissions()` | 工具特定权限逻辑 |
| `PermissionMode` | 全局权限行为选择器 |
| `ToolPermissionContext` | 在执行过程中携带规则和模式 |
| `PreToolUse hooks` | 外部权限拦截器 |
| `Classifier` | AI 驱动的安全评估（auto 模式） |
| `DenialTrackingState` | 回退安全机制 |
| `AskUserQuestionTool` | 多选用户权限提示 |

## 相关文档

- [Message Flow](./message-flow.md)
- [Tool Execution Flow](./tool-execution-flow.md)
- [Session Lifecycle](./session-lifecycle.md)
- [Permissions Utils](../05-utils/permissions-utils.md)
- [AskUserQuestionTool](../02-tools/AskUserQuestionTool.md)
