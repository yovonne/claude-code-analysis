# 消息流

## 目的

追踪消息在 Claude Code 系统中的完整生命周期 — 从用户输入，通过查询引擎和 API，到工具执行和 transcript 持久化。

## 位置

主要来源：
- `restored-src/src/main.tsx` — 入口点和会话设置
- `restored-src/src/QueryEngine.ts` — 消息生命周期编排
- `restored-src/src/utils/messages.ts` — 消息创建和规范化
- `restored-src/src/utils/processUserInput/processUserInput.ts` — 用户输入处理
- `restored-src/src/query.ts` — API 查询循环（由 QueryEngine 引用）

---

## 消息类型

系统使用带以下变体的可辨识联合 `Message` 类型：

| 类型 | 子类型（如适用） | 用途 |
|------|------------------------|---------|
| `user` | — | 用户提示、工具结果、斜杠命令输出 |
| `assistant` | — | 带文本、思考或 tool_use 块的 AI 响应 |
| `system` | `compact_boundary` | 标记对话被压缩的位置 |
| `system` | `api_error` / `api_retry` | API 错误和重试通知 |
| `system` | `local_command` | 本地命令输出（不发送到 API） |
| `system` | `informational` | UI 的状态/信息消息 |
| `system` | `warning` | 警告消息（例如权限模式更改） |
| `system` | `permission_retry` | 权限重试标记 |
| `progress` | — | 实时工具执行进度更新 |
| `attachment` | — | 系统附件（技能、延迟工具、MCP 增量） |
| `stream_event` | — | 原始 API 流事件（message_start、delta、stop） |
| `stream_request_start` | — | 标记 API 请求的开始 |
| `tombstone` | — | 消息移除的控制信号 |
| `tool_use_summary` | — | SDK 使用者的工具使用摘要 |

关键消息属性：
- `uuid` — 用于 transcript 去重的唯一标识符
- `parentUuid` — 将子消息链接到父消息（用于分支）
- `toolUseResult` — 用户消息上的工具执行结果
- `isMeta` — 标记合成/系统生成的用户消息
- `isCompactSummary` — 标记压缩摘要消息

---

## 完整消息流

### 阶段 1：会话初始化

```
main.tsx
  │
  ├─► runMigrations()            — 应用设置迁移
  ├─► init()                     — 加载设置、认证、MDM
  ├─► initializeToolPermissionContext() — 设置权限规则
  ├─► getTools()                 — 构建工具注册表
  ├─► createStore(initialState)  — 创建 AppState store
  │
  └─► launchRepl() 或 ask()      — 进入交互或无头模式
```

### 阶段 2：用户输入处理

```
用户输入提示（REPL 或 SDK）
  │
  ▼
QueryEngine.submitMessage(prompt)
  │
  ├─► fetchSystemPromptParts()   — 构建系统提示 + 用户上下文
  ├─► processUserInput()         — 处理斜杠命令、附件
  │    │
  │    ├─► 解析斜杠命令（/compact、/model 等）
  │    ├─► 解析文件/图像附件
  │    ├─► 注入技能发现
  │    └─► createUserMessage()   — 创建类型化 UserMessage
  │
  ├─► mutableMessages.push(...newMessages)
  │
  └─► recordTranscript(messages) — 在 API 调用之前持久化到磁盘
```

### 阶段 3：查询循环

```
query({ messages, systemPrompt, tools, canUseTool, ... })
  │
  ├─► normalizeMessagesForAPI()  — 过滤仅 UI 消息
  ├─► queryModelWithStreaming()  — 调用 Anthropic API
  │    │
  │    ├─► stream_event: message_start
  │    ├─► stream_event: content_block_start (text/thinking/tool_use)
  │    ├─► stream_event: content_block_delta (流式文本)
  │    ├─► stream_event: content_block_stop
  │    └─► stream_event: message_delta (stop_reason, usage)
  │
  ├─► 对于每个 assistant 内容块：
  │    │
  │    ├─► 如果是 tool_use 块：
  │    │    │
  │    │    ├─► canUseTool(tool, input, context, toolUseID)
  │    │    │    │
  │    │    │    ├─► validateInput()       — 工具特定验证
  │    │    │    ├─► checkPermissions()    — 权限决策
  │    │    │    │    │
  │    │    │    │    ├─► 'allow'  → 执行工具
  │    │    │    │    ├─► 'deny'   → 向 API 返回错误
  │    │    │    │    └─► 'ask'    → 显示权限对话框
  │    │    │    │                    │
  │    │    │    │                    ├─► 用户允许 → 执行
  │    │    │    │                    └─► 用户拒绝 → 错误
  │    │    │    │
  │    │    │    └─► tool.call(args, context, canUseTool)
  │    │    │         │
  │    │    │         ├─► onProgress() 调用 → yield ProgressMessage
  │    │    │         └─► 返回 ToolResult
  │    │    │
  │    │    ├─► ToolResult → createUserMessage(tool_result)
  │    │    └─► 推送到 mutableMessages
  │    │
  │    └─► 如果是 text/thinking 块：
  │         └─► yield AssistantMessage（已经流式传输）
  │
  ├─► 检查终止条件：
  │    ├─► stop_reason === 'end_turn' → 完成
  │    ├─► maxTurns 超出 → yield 错误结果
  │    ├─► maxBudgetUsd 超出 → yield 错误结果
  │    └─► structured output 重试超出 → yield 错误结果
  │
  └─► 返回最终结果消息
```

### 阶段 4：消息规范化和 SDK 映射

```
QueryEngine 从 query() 接收消息
  │
  ├─► switch (message.type):
  │    │
  │    ├─► 'assistant':
  │    │    ├─► 推送到 mutableMessages
  │    │    └─► yield normalizeMessage() → SDKAssistantMessage
  │    │
  │    ├─► 'user':
  │    │    ├─► 推送到 mutableMessages
  │    │    └─► yield normalizeMessage() → SDKUserMessage
  │    │
  │    ├─► 'progress':
  │    │    ├─► 推送到 mutableMessages
  │    │    ├─► recordTranscript()（fire-and-forget）
  │    │    └─► yield normalizeMessage()
  │    │
  │    ├─► 'attachment':
  │    │    ├─► 推送到 mutableMessages
  │    │    ├─► 处理特殊类型：
  │    │    │    ├─► 'structured_output' → 提取数据
  │    │    │    ├─► 'max_turns_reached' → yield 错误结果
  │    │    │    └─► 'queued_command' → 作为用户消息重放
  │    │    └─► recordTranscript()（fire-and-forget）
  │    │
  │    ├─► 'system':
  │    │    ├─► 处理 snip 边界（如果 HISTORY_SNIP 启用）
  │    │    ├─► 推送到 mutableMessages
  │    │    ├─► 如果是 compact_boundary：
  │    │    │    ├─► 释放压缩前的消息供 GC
  │    │    │    └─► yield SDKCompactBoundaryMessage
  │    │    └─► 如果是 api_error: yield SDKApiRetryMessage
  │    │
  │    ├─► 'stream_event':
  │    │    ├─► 跟踪 usage（message_start、message_delta、message_stop）
  │    │    └─► 仅在 includePartialMessages 时 yield
  │    │
  │    └─► 'tombstone': 跳过（控制信号）
  │
  └─► 带 usage、cost、duration 的最终结果消息
```

### 阶段 5：Transcript 持久化

```
recordTranscript(messages)
  │
  ├─► 构建带 parentUuid 链接的消息链
  ├─► 处理去重（跳过已记录的 UUID）
  ├─► 写入 transcript 文件（JSONL 格式）
  │    └─► ~/.claude/projects/<cwd>/.claude/transcripts/<sessionId>.jsonl
  │
  └─► flushSessionStorage() — 当 EAGER_FLUSH 或 COWORK 模式时
```

---

## 关键数据结构

### QueryEngine 内部状态

```
QueryEngine {
  mutableMessages: Message[]      — 工作消息数组
  permissionDenials: SDKPermissionDenial[]  — 跟踪的拒绝
  totalUsage: NonNullableUsage    — 累积的令牌使用量
  discoveredSkillNames: Set<string>  — 每轮技能跟踪
  loadedNestedMemoryPaths: Set<string>  — 内存去重
  abortController: AbortController  — 中断支持
}
```

### QueryEngine.submitMessage() 中的消息流

```
1. 清除每轮跟踪（discoveredSkillNames）
2. 获取系统提示部分（工具、上下文、MCP）
3. 构建 ProcessUserInputContext
4. 处理孤立权限（如果有）
5. 处理用户输入（斜杠命令、附件）
6. 推送新消息到 mutableMessages
7. 记录 transcript（API 调用之前）
8. 构建系统 init 消息（首先 yield）
9. 进入 query() 循环
10. 对于每个 yield 的消息：
    - 按类型分类
    - 推送到 mutableMessages
    - 记录 transcript
    - yield 规范化 SDK 消息
11. 检查终止条件
12. yield 最终结果消息
```

---

## 集成点

| 组件 | 在消息流中的角色 |
|-----------|---------------------|
| `processUserInput` | 将原始输入转换为类型化消息 |
| `query.ts` | 驱动 API 流式循环 |
| `canUseTool` | 用权限检查控制工具执行 |
| `normalizeMessage` | 将内部消息映射到 SDK 格式 |
| `recordTranscript` | 将消息持久化到磁盘 |
| `compact.ts` | 用压缩摘要替换消息 |

## 相关文档

- [Permission Flow](./permission-flow.md)
- [Tool Execution Flow](./tool-execution-flow.md)
- [Session Lifecycle](./session-lifecycle.md)
- [Tool System](../01-core-modules/tool-system.md)
- [Query Engine](../01-core-modules/query-engine.md)
