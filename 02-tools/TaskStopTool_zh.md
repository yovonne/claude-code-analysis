# TaskStopTool

## 目的

TaskStopTool 通过其 ID 停止（终止）运行中的后台任务。它支持所有后台任务类型，包括 shell 任务、agent 任务和远程会话。该工具还通过别名维护与已弃用 `KillShell` 工具名称的向后兼容性，确保现有 transcript 和 SDK 用户继续工作。

## 位置

- `restored-src/src/tools/TaskStopTool/TaskStopTool.ts` — 主工具定义（约 132 行）
- `restored-src/src/tools/TaskStopTool/prompt.ts` — 工具名称常量和描述（9 行）
- `restored-src/src/tools/TaskStopTool/UI.tsx` — UI 渲染函数（41 行）

## 关键导出

| 导出 | 描述 |
|--------|-------------|
| `TaskStopTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `TASK_STOP_TOOL_NAME` | 常量：`'TaskStop'`（来自 prompt.ts） |
| `DESCRIPTION` | 工具描述字符串（来自 prompt.ts） |
| `renderToolUseMessage` | 返回空字符串（无工具使用消息） |
| `renderToolResultMessage` | 渲染已停止任务的显示，附带截断的命令 |
| `Output` | Zod 推断的输出类型：`{ message, task_id, task_type, command? }` |

## 输入/输出模式

### 输入模式

```typescript
{
  task_id?: string,   // 可选：要停止的后台任务的 ID
  shell_id?: string,  // 可选：已弃用，请改用 task_id（KillShell 兼容）
}
```

必须提供 `task_id` 或 `shell_id` 中的至少一个。

### 输出模式

```typescript
{
  message: string,     // 关于操作的状态消息
  task_id: string,     // 被停止的任务的 ID
  task_type: string,   // 被停止的任务的类型
  command?: string,    // 被停止任务的命令或描述
}
```

## 工具定义属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'TaskStop'` | 内部工具标识符 |
| `searchHint` | `'kill a running background task'` | 工具搜索提示 |
| `aliases` | `['KillShell']` | 向后兼容的已弃用名称 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `userFacingName()` | ant 为 `''`，否则为 `'Stop Task'` | UI 中的显示名称 |
| `shouldDefer` | `true` | 工具执行可以延迟 |
| `isConcurrencySafe()` | `true` | 可以安全并发运行 |
| `toAutoClassifierInput()` | `input.task_id ?? input.shell_id ?? ''` | 用于自动模式分类的 ID |

## 任务停止流程

```
1. 输入验证
   - validateInput()：检查提供了 task_id 或 shell_id
   - 解析 id = task_id ?? shell_id
   - 如果 ID 缺失：返回错误代码 1
   - 如果任务未找到：返回错误代码 1
   - 如果任务未运行：返回错误代码 3

2. 任务停止
   - 调用 stopTask(id, { getAppState, setAppState })
   - 委托给任务的特定终止机制
   - 返回 { taskId, taskType, command }

3. 返回结果
   - { message, task_id, task_type, command }
```

## 实现细节

### ID 解析

该工具接受 `task_id` 和 `shell_id` 两个参数：
```typescript
const id = task_id ?? shell_id
```

`shell_id` 被接受是为了与已弃用的 `KillShell` 工具保持向后兼容。新代码应使用 `task_id`。

### 输入验证

`validateInput` 方法执行三项检查：
1. **缺失 ID**：必须提供 `task_id` 或 `shell_id` 之一（错误代码 1）
2. **任务未找到**：ID 必须对应 app state 中存在的任务（错误代码 1）
3. **任务未运行**：任务状态必须为 `'running'`（错误代码 3）

拒绝停止非运行中的任务（pending、completed 等）。

### 任务停止

该工具委托给 `tasks/stopTask.ts` 中的 `stopTask()`：
```typescript
const result = await stopTask(id, {
  getAppState,
  setAppState,
})
```

`stopTask` 函数处理实际终止逻辑，因任务类型而异：
- **Shell 任务**：终止底层 shell 进程
- **Agent 任务**：中止 agent 执行
- **远程任务**：终止远程会话

### 结果格式化

`mapToolResultToToolResultBlockParam` 方法返回 JSON 字符串化的输出：
```typescript
jsonStringify(output)
```

这会产生结果对象的紧凑 JSON 表示。

### UI 渲染

**工具使用消息：**
返回空字符串 — 不显示特殊工具使用消息。

**工具结果消息：**
渲染带有 "· stopped" 后缀的停止命令：
- 命令在非 verbose 模式下截断为最多 2 行和 160 字符
- Verbose 模式显示完整命令
- 截断的命令显示 `… · stopped`，非截断的显示 ` · stopped`

示例输出：
```
npm run build · stopped
```

## 错误处理

| 错误条件 | 处理 |
|----------------|----------|
| 缺少 task_id 和 shell_id | validateInput 返回错误代码 1 |
| 任务未找到 | validateInput 返回错误代码 1 |
| 任务未运行 | validateInput 返回错误代码 3 |
| call() 缺少 ID | 抛出 `Error('Missing required parameter: task_id')` |

## 配置

### 环境变量

| 变量 | 目的 |
|----------|---------|
| `USER_TYPE` | `'ant'` 用于内部用户（影响 userFacingName） |

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `tasks/stopTask.ts` | `stopTask` — 实际任务终止逻辑 |
| `utils/lazySchema.ts` | 延迟模式评估 |
| `utils/slowOperations.ts` | 用于结果格式化的 `jsonStringify` |
| `ink.js` | `Text` 组件用于 UI |
| `ink/stringWidth.js` | 用于命令截断的 `stringWidth` |
| `utils/format.js` | 用于命令截断的 `truncateToWidthNoEllipsis` |
| `components/MessageResponse.js` | `MessageResponse` 组件 |

### 外部

| 包 | 目的 |
|---------|---------|
| `zod/v4` | 输入/输出模式验证 |
| `react` | UI 渲染 |

## 数据流

```
Model 请求
    |
    v
TaskStopTool 输入 { task_id?, shell_id? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 输入验证                                                         │
│ - 解析 id = task_id ?? shell_id                                 │
│ - 检查提供了 ID（错误 1）                                        │
│ - 检查任务存在于 app state（错误 1）                             │
│ - 检查任务状态为 'running'（错误 3）                              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 任务停止                                                         │
│ - stopTask(id, { getAppState, setAppState })                    │
│ - 委托给特定于任务类型的终止机制                                   │
│ - 返回 { taskId, taskType, command }                           │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 结果格式化                                                       │
│ - message: "Successfully stopped task: ${id} (${command})"       │
│ - JSON 字符串化用于模型响应                                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
输出 { message, task_id, task_type, command? }
    |
    v
Model 响应：JSON 结果
UI 显示："${截断的命令} · stopped"
```

## 与任务系统的集成

TaskStopTool 在后台任务管理系统中提供任务终止：

1. **后台任务生命周期**：通过提供在自然完成前取消它们的方法来完成后台任务的生命周期
2. **KillShell 兼容性**：维护 `KillShell` 别名，以便引用旧工具名称的现有会话 transcript 在重放期间继续工作
3. **任务状态管理**：使用 `getAppState` 和 `setAppState` 在终止期间读取和更新任务状态
4. **类型无关的停止**：通过统一的 `stopTask()` 函数处理所有后台任务类型，该函数根据任务类型分派到适当的终止机制
5. **仅运行中保护**：只允许停止正在运行的任务 — 无法停止 pending 或已完成的任务

该工具将所有实际终止逻辑委托给 `tasks/stopTask.ts` 中的 `stopTask()`，它处理特定于平台和任务类型的进程终止细节。
