# TaskOutputTool

## 用途

TaskOutputTool 从运行中或已完成的后台任务（shell、代理、远程会话）检索输出/日志。它支持阻塞（等待完成）和非阻塞（检查当前状态）模式，以及可配置的超时。该工具已被弃用，推荐直接使用 Read 工具读取任务输出文件路径，但为保持向后兼容仍可用。

## 位置

- `restored-src/src/tools/TaskOutputTool/TaskOutputTool.tsx` — 主要工具定义和 UI（约 584 行）
- `restored-src/src/tools/TaskOutputTool/constants.ts` — 工具名称常量（2 行）

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `TaskOutputTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `TASK_OUTPUT_TOOL_NAME` | 常量：`'TaskOutput'` |
| `Progress` | 重导出的 `TaskOutputProgress` 类型，用于流式更新 |
| `TaskOutput` | 内部类型：统一覆盖所有任务类型的输出 |
| `TaskOutputToolOutput` | 输出类型：`{ retrieval_status, task }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  task_id: string,     // 必需：要获取输出的任务 ID
  block?: boolean,     // 可选：是否等待完成（默认：true）
  timeout?: number,    // 可选：最大等待时间（毫秒，0-600000，默认：30000）
}
```

### 输出 Schema

```typescript
{
  retrieval_status: 'success' | 'timeout' | 'not_ready',
  task: {
    task_id: string,
    task_type: TaskType,        // 'local_bash' | 'local_agent' | 'remote_agent' | ...
    status: string,
    description: string,
    output: string,
    exitCode?: number | null,   // 对于 local_bash 任务
    error?: string,
    prompt?: string,            // 对于代理任务
    result?: string,            // 对于代理任务（干净结果）
  } | null,
}
```

## 工具定义属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'TaskOutput'` | 内部工具标识符 |
| `searchHint` | `'read output/logs from a background task'` | 工具搜索提示 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `userFacingName()` | `'Task Output'` | UI 中的显示名称 |
| `shouldDefer` | `true` | 工具执行可以延迟 |
| `aliases` | `['AgentOutputTool', 'BashOutputTool']` | 向后兼容的重命名工具别名 |
| `description()` | `'[Deprecated] — prefer Read on the task output file path'` | 标记为已弃用 |
| `isConcurrencySafe()` | `true` | 只读，可安全并发运行 |
| `isEnabled()` | `"external" !== 'ant'` | 对外部用户禁用 |
| `isReadOnly()` | `true` | 只读操作 |
| `toAutoClassifierInput()` | `input.task_id` | 使用 task_id 进行自动模式分类 |

## 任务输出检索流程

```
1. 输入验证
   - validateInput()：检查提供了 task_id 且任务存在
   - 如果 task_id 缺失返回错误代码 1
   - 如果任务未找到返回错误代码 2

2. 非阻塞模式（block: false）
   - 如果任务未运行/未决：标记为已通知，返回当前输出
   - 如果任务仍在运行：返回 retrieval_status 'not_ready'

3. 阻塞模式（block: true，默认）
   - 显示进度："Waiting for task..."
   - 轮询任务完成（100ms 间隔）
   - 检查中止控制器是否取消
   - 超时：返回 retrieval_status 'timeout'
   - 完成：标记为已通知，返回输出

4. 输出提取（按任务类型）
   - local_bash：来自 shell 命令输出的 stdout + stderr
   - local_agent：首选内存中的干净结果（优于磁盘）
   - remote_agent：来自远程会话的命令和输出
```

## 实现细节

### 输入验证

`validateInput` 方法执行预执行检查：
- 需要任务 ID（错误代码 1）
- 任务必须存在于应用状态中（错误代码 2）

### 非阻塞模式

当 `block: false` 时：
- 立即返回当前任务状态
- 如果任务完成（未运行/未决）：标记任务为 `notified: true` 并返回 `retrieval_status: 'success'`
- 如果任务仍在运行：返回 `retrieval_status: 'not_ready'`

### 阻塞模式

当 `block: true`（默认）时：
- 显示进度消息："Waiting for task (esc to give additional instructions)"
- 通过 `waitForTaskCompletion()` 每 100ms 轮询应用状态
- 遵守 `timeout` 参数（默认 30s，最大 600s）
- 检查 `abortController.signal` 是否取消（抛出 `AbortError`）
- 超时：返回 `retrieval_status: 'timeout'`，如有则包含当前任务状态
- 完成：标记任务为 `notified: true`，返回 `retrieval_status: 'success'`

### 按任务类型的输出提取

`getTaskOutputData()` 函数处理不同的任务类型：

**local_bash（shell 任务）：**
- 从 `shellCommand.taskOutput` 获取 stdout 和 stderr
- 如果 taskOutput 不可用则回退到磁盘输出
- 包含任务结果中的 `exitCode`

**local_agent（子代理任务）：**
- 首选内存中 `result.content` 的干净结果，优于磁盘输出
- 磁盘输出是指向完整会话 transcripts 的符号链接（每条消息、工具使用）
- 内存中的结果仅包含最终的助手文本内容块
- 包含 `prompt`、`result` 和 `error` 字段

**remote_agent（远程会话）：**
- 将 `command` 作为 prompt 字段返回
- 包含基本输出字段

### 输出格式化

`mapToolResultToToolResultBlockParam` 方法将输出格式化为类似 XML 的标签：
```xml
<retrieval_status>success</retrieval_status>

<task_id>...</task_id>
<task_type>...</task_type>
<status>...</status>
<exit_code>0</exit_code>
<output>
...
</output>
<error>...</error>
```

### 弃用通知

该工具已弃用。提示指示模型：
- 使用 Read 工具读取任务输出文件路径
- 后台任务在工具结果中返回其输出文件路径
- 任务完成发送带有相同路径的 `<task-notification>`
- 直接读取文件而不是使用此工具

### UI 组件

该工具包含丰富的 UI 渲染：
- `renderToolUseTag`：以暗淡颜色显示 task_id
- `renderToolUseProgressMessage`：显示任务描述和等待状态
- `renderToolResultMessage`：使用 `TaskOutputResultDisplay` 组件进行特定类型渲染
- `renderToolUseRejectedMessage`：后备拒绝消息
- `renderToolUseErrorMessage`：后备错误消息

`TaskOutputResultDisplay` 组件按任务类型不同渲染：
- **local_bash**：使用 `BashToolResultMessage` 组件
- **local_agent**：显示 prompt、result 和 error，带展开快捷方式提示
- **remote_agent**：显示 description、status 和可选输出
- **其他**：显示 description、status 和截断输出（前 500 个字符）

## 错误处理

| 错误条件 | 处理 |
|----------------|----------|
| 缺少 task_id | validateInput 返回错误代码 1 |
| 任务未找到 | validateInput 返回错误代码 2；call() 抛出 Error |
| 任务超时 | 返回 `{ retrieval_status: 'timeout', task: null }` |
| 任务仍在运行（非阻塞） | 返回 `{ retrieval_status: 'not_ready', task: current }` |
| 中止/取消 | 抛出 `AbortError` |
| 工具禁用 | 对外部用户不可用 |

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `USER_TYPE` | `'ant'` 表示内部用户（启用工具） |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/task/diskOutput.ts` | `getTaskOutput` — 从磁盘读取任务输出 |
| `utils/task/framework.ts` | `updateTaskState` — 标记任务为已通知 |
| `utils/task/outputFormatting.ts` | `formatTaskOutput` — 格式化输出以显示 |
| `utils/semanticBoolean.ts` | `block` 参数的语义布尔解析 |
| `utils/sleep.ts` | 轮询间隔睡眠 |
| `utils/slowOperations.ts` | `jsonParse` 用于结果解析 |
| `utils/stringUtils.ts` | `countCharInString` 用于行计数 |
| `utils/errors.ts` | `AbortError` 用于取消 |
| `utils/messages.ts` | `extractTextContent` 用于代理结果提取 |
| `tasks/types.ts` | `TaskState` 类型定义 |
| `tasks/LocalShellTask/guards.ts` | `LocalShellTaskState` 类型 |
| `tasks/LocalAgentTask/LocalAgentTask.ts` | `LocalAgentTaskState` 类型 |
| `tasks/RemoteAgentTask/RemoteAgentTask.ts` | `RemoteAgentTaskState` 类型 |

### 外部

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入 schema 验证 |
| `react` | UI 渲染组件 |
| `react/compiler-runtime` | React 编译器记忆化 |

## 数据流

```
Model Request
    |
    v
TaskOutputTool Input { task_id, block?, timeout? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ INPUT VALIDATION                                                 │
│ - validateInput(): task_id required, task must exist             │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ MODE CHECK                                                       │
│                                                                  │
│  block: false → Non-blocking                                    │
│    - Task complete → success + current output                   │
│    - Task running  → not_ready + current output                 │
│                                                                  │
│  block: true → Blocking                                         │
│    - Show progress: "Waiting for task..."                        │
│    - Poll every 100ms                                           │
│    - Check abort signal                                          │
│    - On timeout → timeout status                                │
│    - On complete → success + output                              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ OUTPUT EXTRACTION (getTaskOutputData)                            │
│                                                                  │
│  local_bash:   stdout + stderr + exitCode                       │
│  local_agent:  clean result (in-memory preferred) + prompt      │
│  remote_agent: command + output                                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ NOTIFICATION                                                     │
│ - Mark task as notified (updateTaskState)                        │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { retrieval_status, task: { task_id, task_type, status, ... } }
    |
    v
Model Response: XML-formatted task output
```

## 与任务系统的集成

TaskOutputTool 桥接任务执行和结果检索层：

1. **后台任务监控**：提供检查由 BashTool（后台模式）、AgentTool 或远程会话产生的任务的方式
2. **任务通知抑制**：在检索输出后将任务标记为 `notified: true`，防止冗余的完成通知
3. **类型无关输出**：通过统一接口处理所有任务类型（`local_bash`、`local_agent`、`remote_agent` 等）
4. **阻塞/非阻塞灵活性**：支持等待完成和检查当前状态两种模式
5. **弃用路径**：该工具正在被直接文件读取取代。任务现在在工具结果中返回其输出文件路径，完成通知包含相同路径，允许模型通过 Read 工具直接读取输出

该工具从内存中的任务状态和磁盘输出文件两者读取，优先选择代理任务的干净内存结果（避免原始 transcripts 噪音）。
