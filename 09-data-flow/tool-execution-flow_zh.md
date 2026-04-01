# 工具执行流

## 目的

追踪工具执行的完整生命周期 — 从模型的 tool_use API 响应，通过验证、权限检查、执行、进度报告，到结果处理。

## 位置

主要来源：
- `restored-src/src/Tool.ts` — `Tool` 接口、`buildTool`、`ToolUseContext`、`ToolResult`
- `restored-src/src/QueryEngine.ts` — 工具执行编排
- `restored-src/src/query.ts` — 调度工具调用的查询循环
- `restored-src/src/services/tools/toolExecution.ts` — 执行引擎
- `restored-src/src/services/tools/toolOrchestration.ts` — 多工具协调

---

## 工具接口

系统中每个工具实现 `Tool<Input, Output, Progress>` 接口：

```typescript
type Tool<Input, Output, Progress> = {
  // 标识
  name: string
  aliases?: string[]
  searchHint?: string           // ToolSearch 的关键字搜索提示

  // Schema
  inputSchema: z.ZodType<Input>
  inputJSONSchema?: ToolInputJSONSchema  // 用于 MCP 工具
  outputSchema?: z.ZodType<Output>

  // 核心执行
  call(args: Input, context: ToolUseContext, canUseTool, parentMessage, onProgress?)
    : Promise<ToolResult<Output>>

  // 描述（动态、上下文感知）
  description(input: Input, options): Promise<string>
  prompt(options): Promise<string>

  // 权限和验证
  validateInput?(input: Input, context): Promise<ValidationResult>
  checkPermissions(input: Input, context): Promise<PermissionResult>
  preparePermissionMatcher?(input: Input): Promise<(pattern: string) => boolean>

  // 安全性和行为
  isEnabled(): boolean
  isConcurrencySafe(input: Input): boolean
  isReadOnly(input: Input): boolean
  isDestructive?(input: Input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  requiresUserInteraction?(): boolean

  // UI 渲染
  renderToolUseMessage(input, options): React.ReactNode
  renderToolUseProgressMessage(progressMessages, options): React.ReactNode
  renderToolResultMessage(content, progressMessages, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null

  // 显示帮助器
  userFacingName(input): string
  userFacingNameBackgroundColor?(input): string
  getActivityDescription?(input): string | null
  getToolUseSummary?(input): string | null
  isResultTruncated?(output): boolean
  renderToolUseTag?(input): React.ReactNode
  renderToolUseQueuedMessage?(): React.ReactNode

  // 数据转换
  mapToolResultToToolResultBlockParam(content: Output, toolUseID: string)
    : ToolResultBlockParam
  toAutoClassifierInput(input: Input): unknown
  extractSearchText?(out: Output): string

  // 高级
  shouldDefer?: boolean         // 通过 ToolSearch 延迟加载
  alwaysLoad?: boolean          // 从不延迟
  strict?: boolean              // 严格模式执行
  isTransparentWrapper?(): boolean
  isSearchOrReadCommand?(input): { isSearch, isRead, isList? }
  isOpenWorld?(input): boolean
  backfillObservableInput?(input): void
  getPath?(input: Input): string
  isMcp?: boolean
  isLsp?: boolean
  mcpInfo?: { serverName, toolName }

  // 大小限制
  maxResultSizeChars: number
}
```

---

## 工具执行生命周期

### 阶段 1：工具发现和加载

```
QueryEngine.start()
  │
  ├─► getTools()                 — 构建初始工具注册表
  │    │
  │    ├─► 内置工具（Bash、Read、Edit、Write 等）
  │    ├─► MCP 工具（来自连接的服务器）
  │    ├─► 代理工具（如启用）
  │    └─► 插件工具（如安装）
  │
  ├─► 延迟工具（shouldDefer: true）
  │    └─► 不包含在初始 API 调用中
  │    └─► 通过 ToolSearch 工具发现
  │
  └─► 工具过滤
       ├─► 每个工具的 isEnabled() 检查
       ├─► MCP 权限过滤
       └─► 沙箱命令排除
```

### 阶段 2：工具调用调度

```
API 返回带 tool_use 内容块的 assistant 消息
  │
  ▼
query.ts: handleToolUse()
  │
  ├─► 对于每个 tool_use 块（可能并行）：
  │    │
  │    ├─► 1. 按名称查找工具（或别名）
  │    │    └─► toolMatchesName(tool, name)
  │    │
  │    ├─► 2. 验证输入 schema
  │    │    └─► inputSchema.parse(input)
  │    │
  │    ├─► 3. backfillObservableInput(input)
  │    │    └─► 为钩子/transcript 添加遗留/派生字段
  │    │
  │    ├─► 4. validateInput(input, context)
  │    │    ├─► 工具特定验证
  │    │    └─► 如果失败 → 返回错误 tool_result
  │    │
  │    └─► 5. canUseTool(tool, input, context, toolUseID)
  │         └─► 见权限流文档
  │
  └─► 6. 根据权限结果执行或拒绝
```

### 阶段 3：工具执行

```
权限 === 'allow'
  │
  ▼
tool.call(args, context, canUseTool, parentMessage, onProgress)
  │
  ├─► 工具实现执行：
  │    │
  │    ├─► BashTool: 生成子进程，流式输出
  │    ├─► FileReadTool: 读取文件，应用限制
  │    ├─► FileEditTool: 应用 diff/replace
  │    ├─► FileWriteTool: 写入文件内容
  │    ├─► MCPTool: 调用 MCP 服务器方法
  │    ├─► AgentTool: 生成子代理
  │    └─► ...（每个工具有独特实现）
  │
  ├─► 进度报告（可选）：
  │    │
  │    ├─► onProgress({ toolUseID, data: ProgressData })
  │    ├─► ProgressMessage yield 到查询循环
  │    ├─► UI 更新带进度显示
  │    └─► 进度类型：
  │         ├─► BashProgress（stdout/stderr 块）
  │         ├─► WebSearchProgress（搜索阶段）
  │         ├─► MCPProgress（MCP 调用阶段）
  │         ├─► AgentToolProgress（子代理状态）
  │         ├─► SkillToolProgress（技能执行）
  │         ├─► TaskOutputProgress（任务输出）
  │         └─► REPLToolProgress（REPL 执行）
  │
  ├─► 中断处理：
  │    │
  │    ├─► interruptBehavior() === 'cancel' → 中止工具
  │    └─► interruptBehavior() === 'block' → 等待完成
  │
  └─► 返回 ToolResult<Output>
       │
       ├─► data: Output
       ├─► newMessages?: Message[]  （要注入的其他消息）
       ├─► contextModifier?: (context) => ToolUseContext
       └─► mcpMeta?: { _meta, structuredContent }
```

### 阶段 4：结果处理

```
ToolResult 接收
  │
  ▼
mapToolResultToToolResultBlockParam(result, toolUseID)
  │
  ├─► 转换工具输出为 API 格式
  ├─► 处理过大结果：
  │    ├─► 如果 result > maxResultSizeChars
  │    └─► 持久化到临时文件，返回预览 + 路径
  │
  ▼
createUserMessage({
  content: tool_result_block,
  toolUseResult: true,
  parentUuid: assistantMessage.uuid,
})
  │
  ▼
推送到 mutableMessages
  │
  ▼
记录 transcript
  │
  └─► 继续查询循环（模型看到工具结果）
```

---

## 工具执行模式

### 顺序 vs 并行执行

```
模型可以在一个响应中返回多个 tool_use 块：
  │
  ├─► 顺序工具（isConcurrencySafe === false）：
  │    └─► 一次执行一个，按顺序
  │    └─► 每个结果对下一个工具可见
  │
  └─► 并行工具（isConcurrencySafe === true）：
       └─► 通过 Promise.all 同时执行
       └─► 结果收集并一起返回
```

### 延迟工具加载（ToolSearch）

```
带有 shouldDefer: true 的工具
  │
  ├─► 不包含在初始 API 工具列表中
  ├─► 模型调用 ToolSearch 查找它
  ├─► ToolSearch 返回工具名称 + 描述
  ├─► 模型然后按名称调用工具
  └─► 工具在该轮按需加载
```

### 子代理工具执行（AgentTool）

```
AgentTool.call()
  │
  ├─► 创建子代理上下文（克隆或分叉）
  ├─► 生成子代理或子进程
  ├─► 子代理运行自己的查询循环
  ├─► 进度流回父代理
  ├─► 结果作为 ToolResult 返回
  └─► 子代理消息可选保留在 transcript 中
```

---

## ToolResult 结构

```typescript
type ToolResult<T> = {
  data: T                                    // 实际输出
  newMessages?: Message[]                    // 要注入的其他消息
  contextModifier?: (context) => ToolUseContext  // 修改执行上下文
  mcpMeta?: {                                // MCP 协议直通
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

---

## 工具默认值（通过 buildTool）

`buildTool()` 工厂填充安全默认值：

| 方法 | 默认值 | 理由 |
|--------|---------|-----------|
| `isEnabled()` | `true` | 工具可用 |
| `isConcurrencySafe()` | `false` | 假设不安全（fail-closed） |
| `isReadOnly()` | `false` | 假设写入（fail-closed） |
| `isDestructive()` | `false` | 假设非破坏性 |
| `checkPermissions()` | `{ behavior: 'allow' }` | 推迟到通用权限系统 |
| `toAutoClassifierInput()` | `''` | 跳过分类器（安全工具覆盖） |
| `userFacingName()` | `name` | 使用工具名称 |

---

## 错误处理

```
工具执行错误
  │
  ├─► API 错误 → 带错误文本的合成 assistant 消息
  ├─► 验证错误 → 带错误消息的 tool_result
  ├─► 权限拒绝 → 带拒绝原因的 tool_result
  ├─► 超时 → 带超时消息的 tool_result
  ├─► 子进程崩溃 → 带 stderr 的 tool_result
  └─► 未处理异常 → 带错误堆栈的 tool_result
  │
  └─► 所有错误成为带 tool_result 内容的用户消息
       └─► 模型看到错误并可以重试或适应
```

---

## 集成点

| 组件 | 在工具执行中的角色 |
|-----------|----------------------|
| `Tool.call()` | 核心工具实现 |
| `canUseTool` | 权限门 |
| `ToolUseContext` | 执行上下文（状态、中止等） |
| `onProgress` | 进度报告回调 |
| `mapToolResultToToolResultBlockParam` | API 格式转换 |
| `maxResultSizeChars` | 结果大小管理 |
| `buildTool()` | 默认填充工厂 |

## 相关文档

- [Message Flow](./message-flow.md)
- [Permission Flow](./permission-flow.md)
- [Session Lifecycle](./session-lifecycle.md)
- [Tool System](../01-core-modules/tool-system.md)
- [BashTool](../02-tools/BashTool.md)
- [FileEditTool](../02-tools/FileEditTool.md)
