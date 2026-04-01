# EnterPlanModeTool

## 目的

EnterPlanModeTool 请求用户许可以将会话从默认/代码模式转换到计划模式。在计划模式下，模型专注于探索代码库、理解现有模式和在编写任何代码之前设计实现方法。该工具强制执行只读探索阶段，其中文件编辑仅限于计划文件。

## 位置

- `restored-src/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` — 主工具定义和调用逻辑（127 行）
- `restored-src/src/tools/EnterPlanModeTool/prompt.ts` — 带使用指南的工具提示（171 行）
- `restored-src/src/tools/EnterPlanModeTool/UI.tsx` — 终端 UI 渲染（33 行）
- `restored-src/src/tools/EnterPlanModeTool/constants.ts` — 工具名称常量（2 行）

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `EnterPlanModeTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | 输出类型：`{ message: string }` |
| `ENTER_PLAN_MODE_TOOL_NAME` | 常量字符串 `'EnterPlanMode'` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  // 不需要参数 — 该工具是简单的模式转换请求
}
```

输入 schema 是一个严格空对象。不需要配置，因为该工具的唯一目的是请求模式转换。

### 输出 Schema

```typescript
{
  message: string,  // 确认计划模式已输入
}
```

## 计划模式激活

### 激活流程

```
1. 模型调用 EnterPlanMode
   - 工具无参数调用

2. 上下文验证
   - 如果从代理上下文调用则抛出（存在 agentId）
   - 计划模式不能从子代理进入

3. 模式转换（handlePlanModeTransition）
   - handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')
   - 验证从当前模式转换是否合法

4. 状态更新
   - context.setAppState() 包装 prepareContextForPlanMode()
   - applyPermissionUpdate() 将模式设置为 'plan'，目标为 'session'
   - prepareContextForPlanMode() 在用户的 defaultMode 为 'auto' 时运行分类器激活副作用

5. 返回结果
   - message: "Entered plan mode. You should now focus on exploring..."
   - mapToolResultToToolResultBlockParam() 附加工作流指令
```

### 状态更改

该工具修改 app state 中的 `toolPermissionContext`：

| 字段 | 之前 | 之后 |
|-------|--------|-------|
| `mode` | `'default'` / `'auto'` 等 | `'plan'` |
| `prePlanMode` | undefined | （由 prepareContextForPlanMode 设置，保留） |
| 权限规则 | 完全访问 | 剥离为只读（计划文件除外） |

### 分类器副作用

当 `defaultMode` 为 `'auto'` 时，`prepareContextForPlanMode()` 触发自动模式分类器激活副作用。这确保分类器状态与模式转换一致。完整生命周期在 `permissionSetup.ts` 中管理。

## 模式门控

### 可用性条件

该工具在以下条件下被禁用：

```typescript
isEnabled() {
  if (
    (feature('KAIROS') || feature('KAIROS_CHANNELS')) &&
    getAllowedChannels().length > 0
  ) {
    return false
  }
  return true
}
```

当 `--channels` 功能标志激活时（Telegram/Discord 集成），EnterPlanMode 和 ExitPlanMode 都被禁用。这防止计划模式成为陷阱 — 模型可以进入计划模式，但审批对话框会挂起，因为消息平台上没有终端 UI 可用。

### 工具属性

| 属性 | 值 | 理由 |
|----------|-------|-----------|
| `shouldDefer` | `true` | 执行前需要用户批准 |
| `isConcurrencySafe` | `true` | 模式转换是原子的 |
| `isReadOnly` | `true` | 不修改文件 |
| `userFacingName` | `''`（空） | 仅内部工具名称 |

## 计划模式工作流指令

### 结果块内容

工具成功后，`mapToolResultToToolResultBlockParam()` 返回指导模型在计划模式中行为的指令。内容根据面试阶段功能是否启用而不同：

**无面试阶段**（标准模式）：
```
Entered plan mode. You should now focus on exploring the codebase and designing an implementation approach.

In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.
```

**带面试阶段**（`isPlanModeInterviewPhaseEnabled()`）：
```
Entered plan mode. You should now focus on exploring the codebase and designing an implementation approach.

DO NOT write or edit any files except the plan file. Detailed workflow instructions will follow.
```

面试阶段变体将详细指令推迟到系统消息中的单独 `plan_mode` 附件。

## 提示指导

### 使用时机（外部用户变体）

提示提供了七个应该优先使用 EnterPlanMode 的条件：

1. **新功能实现** — 添加有意义的功能
2. **多种有效方法** — 几种不同的解决方案可能
3. **代码修改** — 影响现有行为的更改
4. **架构决策** — 在模式/技术之间选择
5. **多文件更改** — 涉及超过 2-3 个文件的任务
6. **需求不明确** — 在理解范围之前需要探索
7. **用户偏好重要** — 实现可以多种方式

### 不使用时机

- 单行或少数行修复（拼写错误、明显 bug、小调整）
- 添加具有清晰需求的单个函数
- 具有非常具体详细指令的任务
- 纯研究/探索任务（使用 Agent 工具）

### Ant 变体

当 `USER_TYPE === 'ant'` 时，使用更保守的提示，要求真正的架构模糊性才能建议计划模式。Ant 变体：
- 将标准缩小到仅重要的模糊性
- 将"我们可以做 X 吗"视为开始信号，而非计划
- 对于次要问题更喜欢使用 AskUserQuestion 而非完整规划

### 发生什么部分

描述计划模式行为的 `WHAT_HAPPENS_SECTION` 有条件地包含：
- 当 `isPlanModeInterviewPhaseEnabled()` 为 true 时省略（指令通过附件到达）
- 否则作为工具提示的一部分包含

## UI 渲染

### 工具使用消息

`renderToolUseMessage()` 返回 `null` — 不显示中间"using"消息。

### 工具结果消息

```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
    <Text> Entered plan mode</Text>
  </Box>
  <Box paddingLeft={2}>
    <Text dimColor>
      Claude is now exploring and designing an implementation approach.
    </Text>
  </Box>
</Box>
```

显示带状态文本的彩色指示圆点（计划模式颜色）和暗淡描述。

### 工具使用拒绝消息

```tsx
<Box flexDirection="row" marginTop={1}>
  <Text color={getModeColor('default')}>{BLACK_CIRCLE}</Text>
  <Text> User declined to enter plan mode</Text>
</Box>
```

当用户拒绝计划模式请求时，以默认模式颜色显示。

## 集成点

### `bootstrap/state.js`

- `handlePlanModeTransition(from, to)` — 验证和记录模式转换
- `getAllowedChannels()` — 返回用于门控的配置通道列表

### `utils/permissions/permissionSetup.js`

- `prepareContextForPlanMode(context)` — 为计划模式准备权限上下文，剥离危险规则，保留计划前模式

### `utils/permissions/PermissionUpdate.js`

- `applyPermissionUpdate(context, update)` — 原子应用 `{ type: 'setMode', mode: 'plan', destination: 'session' }` 更新

### `utils/planModeV2.js`

- `isPlanModeInterviewPhaseEnabled()` — 面试阶段行为的功能标志检查

### `utils/plans.js`

- 计划文件路径解析和管理（由 ExitPlanMode 使用，在工作流指令中引用）

## 数据流

```
Model decides task needs planning
    │
    ▼
EnterPlanMode tool call (no params)
    │
    ▼
┌─────────────────────────────────────────┐
│ 验证                              │
│ - 检查：不在代理上下文中           │
│ - 检查：工具已启用（无通道）  │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 模式转换                         │
│                                         │
│ handlePlanModeTransition(current, plan) │
│                                         │
│ prepareContextForPlanMode():            │
│   - 剥离危险权限         │
│   - 保留 prePlanMode        │
│   - 运行分类器副作用         │
│                                         │
│ applyPermissionUpdate():                │
│   - 设置模式为 'plan'                  │
│   - 目标：'session'              │
└─────────────────────────────────────────┘
    │
    ▼
State updated: mode = 'plan'
    │
    ▼
带工作流指令的结果块：
  - 无面试阶段：6 步指南
  - 带面试阶段：简要 + 推迟
    │
    ▼
Model enters read-only exploration phase
```
