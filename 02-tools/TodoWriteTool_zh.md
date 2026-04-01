# TodoWriteTool

## 目的

TodoWriteTool 管理当前编码会话的结构化任务清单。它允许 Claude 创建、更新和跟踪多步骤任务的进度，展示周到性并让用户了解整体进度。该工具将 todos 存储在按会话或 agent ID 键控的 AppState 中，自动清除已完成列表，并包含一个验证提醒机制，在关闭 3+ 个任务而没有验证步骤时提醒模型生成验证 agent。

## 位置

- `restored-src/src/tools/TodoWriteTool/TodoWriteTool.ts` — 主工具定义和状态管理（116 行）
- `restored-src/src/tools/TodoWriteTool/prompt.ts` — 带示例的全面使用指南（185 行）
- `restored-src/src/tools/TodoWriteTool/constants.ts` — 工具名称常量（2 行）

## 关键导出

### 来自 TodoWriteTool.ts

| 导出 | 描述 |
|--------|-------------|
| `TodoWriteTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | Zod 推断的输出类型：`{ oldTodos, newTodos, verificationNudgeNeeded? }` |

## 输入/输出模式

### 输入模式

```typescript
{
  todos: TodoItem[],  // 必需：更新后的完整 todo 列表
}

// TodoItem 结构（来自 TodoListSchema）：
{
  content: string,       // 任务描述（祈使语气）
  activeForm: string,    // 现在进行时形式（例如 "Running tests"）
  status: 'pending' | 'in_progress' | 'completed',
}
```

### 输出模式

```typescript
{
  oldTodos: TodoItem[],              // 更新前的 todo 列表
  newTodos: TodoItem[],              // 更新后的 todo 列表
  verificationNudgeNeeded?: boolean, // 是否应显示验证提醒
}
```

## 任务管理

### 执行流程

```
1. 接收输入
   - Model 返回 TodoWrite tool_use，附带 { todos: [...] }

2. 权限检查
   - 自动允许（todo 操作不需要权限检查）

3. 状态更新
   a. 获取当前 app state
   b. 确定存储键：agentId ?? sessionId
   c. 存储旧 todos 以进行比较
   d. 检查是否所有 todos 都已完成
   e. 如果全部完成 → 清除列表（newTodos = []）
   f. 否则 → 存储提供的 todos
   g. 更新 AppState.todos[todoKey]

4. 验证提醒检查
   条件（全部为真）：
   - VERIFICATION_AGENT 功能标志启用
   - tengu_hive_evidence 功能标志启用
   - 没有 agentId（仅限主线程 agent）
   - 所有 todos 都已完成
   - 列表中有 3+ 项
   - 没有包含 "verif"（不区分大小写）的内容
   → 如果全部为真：设置 verificationNudgeNeeded = true

5. 返回结果
   - oldTodos, newTodos, verificationNudgeNeeded
```

### 任务状态

| 状态 | 含义 |
|-------|---------|
| `pending` | 任务尚未开始 |
| `in_progress` | 正在进行中（限制：同一时间只能有一个） |
| `completed` | 任务已成功完成 |

### 任务描述要求

每个任务必须有两种形式：

| 字段 | 形式 | 示例 |
|-------|------|---------|
| `content` | 祈使语气 | "Run tests"、"Build the project" |
| `activeForm` | 现在进行时 | "Running tests"、"Building the project" |

### 完成时自动清除

当所有 todos 达到 `completed` 状态时，列表会自动清除（设置为 `[]`）。这可以防止陈旧的已完成任务列表持续存在于会话状态中。

## 验证提醒

当模型完成 3+ 个任务而没有包含验证步骤时，工具结果包含提醒：

```
NOTE: You just closed out 3+ tasks and none of them was a verification step.
Before writing your final summary, spawn the verification agent
(subagent_type="<VERIFICATION_AGENT_TYPE>"). You cannot self-assign PARTIAL
by listing caveats in your summary — only the verifier issues a verdict.
```

这在最后一个任务关闭且循环自然退出的精确时刻触发，捕获模型跳过验证的模式。

### 提醒条件

全部为真：

| 条件 | 检查 |
|-----------|-------|
| 功能标志 | `VERIFICATION_AGENT` 启用 |
| 功能标志 | `tengu_hive_evidence` 启用 |
| 仅限主 agent | `context.agentId` 为假 |
| 全部完成 | 每个 todo 的 `status === 'completed'` |
| 列表大小 | `todos.length >= 3` |
| 无验证 | 没有 todo 内容匹配 `/verif/i` |

## 存储

Todos 作为映射存储在 `AppState.todos` 中：

```typescript
AppState.todos = {
  [sessionId]: [...],      // 主会话 todos
  [agentId]: [...],        // agent 特定的 todos（如果生成了）
}
```

存储键是 `context.agentId ?? getSessionId()`，确保：
- 主会话使用会话 ID
-生成的 agent 有自己独立的 todo 列表

## 使用时机

工具提示提供了关于何时使用 todo 列表的详细指导：

### 使用当

1. 复杂的多步骤任务（3+ 个不同步骤）
2. 需要仔细计划的非平凡任务
3. 用户明确请求 todo 列表
4. 用户提供多个任务（编号或逗号分隔）
5. 收到新指令后
6. 开始处理任务时（开始前标记为 `in_progress`）
7. 完成任务后（标记为 `completed`，添加后续任务）

### 跳过当

1. 单一、直接的任务
2. 没有组织收益的平凡任务
3. 可在 3 个平凡步骤内完成的任务
4. 纯对话或信息请求

## 任务管理规则

| 规则 | 描述 |
|------|---------|
| 一次一个 | 任何时候只能有一个任务处于 `in_progress` |
| 立即完成 | 完成后立即标记任务 |
| 不批量处理 | 不要批量完成 |
| 删除无关项 | 从列表中完全删除不再相关的任务 |
| 完全完成 | 仅在完全完成时标记为已完成 |
| 阻塞项 | 如果被阻塞则保持 `in_progress`；创建新任务以解决问题 |
| 具体项目 | 创建具有清晰名称的可操作、具体项目 |

## 权限处理

| 操作 | 权限决定 |
|-----------|-------------------|
| 所有操作 | 自动允许（不需要权限检查） |

该工具被认为是安全的，因为它只修改内存状态，没有对文件系统或系统的副作用。

## 工具可用性

| 条件 | 效果 |
|-----------|---------|
| `isTodoV2Enabled()` 返回 true | 工具**禁用**（V2 取代它） |
| `isTodoV2Enabled()` 返回 false | 工具**启用** |

此工具代表 V1 todo 系统。当 V2 启用时，此工具被禁用，新系统接管。

## 结果消息

工具结果始终包含标准消息：

```
Todos have been modified successfully. Ensure that you continue to use the todo list to track your current tasks if applicable
```

当触发验证提醒时，它将此提醒附加到消息中。

## 配置

### 功能标志

| 标志 | 目的 |
|------|---------|
| `VERIFICATION_AGENT` | 启用验证 agent 功能 |
| `tengu_hive_evidence` | 启用验证提醒 |

### 环境变量

| 变量 | 目的 |
|----------|---------|
| `USER_TYPE` | `'ant'` 用于内部用户（启用额外功能） |

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `bootstrap/state.js` | 会话 ID 访问 |
| `services/analytics/growthbook.js` | 功能标志访问 |
| `utils/tasks.js` | Todo V2 启用检查 |
| `utils/todo/types.js` | TodoListSchema 定义 |
| `tools/AgentTool/constants.js` | 验证 agent 类型常量 |

### 外部

| 包 | 目的 |
|---------|---------|
| `zod/v4` | 输入/输出模式验证 |
| `bun:bundle` | 功能标志 API |

## 数据流

```
Model 请求
    |
    v
TodoWriteTool 输入 { todos: [{ content, activeForm, status }] }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 权限检查                                                         │
│  - 自动允许（不需要权限检查）                                     │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 状态更新                                                         │
│                                                                 │
│  1. 获取 AppState                                               │
│  2. 确定键：agentId ?? sessionId                                 │
│  3. 保存 oldTodos                                               │
│  4. 检查是否全部完成                                             │
│  5. 如果全部完成 → 清除列表                                       │
│  6. 否则 → 存储新 todos                                          │
│  7. 更新 AppState.todos[key]                                    │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 验证提醒检查                                                      │
│                                                                 │
│  全部条件：                                                       │
│  - VERIFICATION_AGENT 标志开启                                   │
│  - tengu_hive_evidence 标志开启                                 │
│  - 主 agent（无 agentId）                                        │
│  - 所有 todos 完成                                              │
│  - 3+ 项                                                        │
│  - 任何内容中没有 "verif"                                        │
│                                                                 │
│  → 如果全部为真：将验证提醒附加到结果                             │
└─────────────────────────────────────────────────────────────────┘
    |
    v
输出 { oldTodos, newTodos, verificationNudgeNeeded? }
    |
    v
Model 响应（带成功消息的工具结果 + 可选提醒）
```
