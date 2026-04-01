# SendMessageTool

## 用途

SendMessageTool 是 Claude Code 的 swarm 协议的代理间通信系统。它使代理能够向队友发送消息、向整个团队广播、处理关闭请求/响应以及管理计划审批工作流。它支持团队内消息传递和通过 UDS 套接字及远程控制桥的跨会话通信。

## 位置

- `restored-src/src/tools/SendMessageTool/SendMessageTool.ts` — 主要工具定义（918 行）
- `restored-src/src/tools/SendMessageTool/prompt.ts` — 工具提示和描述（50 行）
- `restored-src/src/tools/SendMessageTool/constants.ts` — 工具名称常量
- `restored-src/src/tools/SendMessageTool/UI.tsx` — 工具使用/结果消息的 UI 渲染

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `SendMessageTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | Zod 推断的输入类型：`{ to, summary?, message }` |
| `SendMessageToolOutput` | 联合类型：`MessageOutput | BroadcastOutput | RequestOutput | ResponseOutput` |
| `MessageRouting` | 路由元数据：`{ sender, senderColor?, target, targetColor?, summary?, content? }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  to: string,              // 接收者：队友名称，"*" 表示广播，"uds:<path>"，或 "bridge:<session-id>"
  summary?: string,        // 5-10 个词的 UI 预览（当 message 是字符串时必需）
  message: string | {      // 纯文本或结构化消息
    type: 'shutdown_request', reason?: string
  } | {
    type: 'shutdown_response', request_id: string, approve: boolean, reason?: string
  } | {
    type: 'plan_approval_response', request_id: string, approve: boolean, feedback?: string
  }
}
```

### 输出类型

**MessageOutput**（直接消息）：
```typescript
{ success: boolean, message: string, routing?: MessageRouting }
```

**BroadcastOutput**（广播给所有人）：
```typescript
{ success: boolean, message: string, recipients: string[], routing?: MessageRouting }
```

**RequestOutput**（关闭请求）：
```typescript
{ success: boolean, message: string, request_id: string, target: string }
```

**ResponseOutput**（关闭/计划响应）：
```typescript
{ success: boolean, message: string, request_id?: string }
```

## 消息类型

### 纯文本消息

```json
{"to": "researcher", "summary": "assign task 1", "message": "start on task #1"}
```

- 按名称发送到特定队友
- 需要 `summary` 用于 UI 预览
- 自动传送到接收者的邮箱

### 广播消息

```json
{"to": "*", "summary": "team update", "message": "everyone check your tasks"}
```

- 发送给除发送者外的所有队友
- 代价昂贵（与团队规模成线性关系）——仅在每个人都真正需要时使用
- 无法通过广播发送结构化消息

### 关闭请求/响应

```json
// 请求（团队负责人到队友）
{"to": "researcher", "message": {"type": "shutdown_request", "reason": "work complete"}}

// 响应（队友到团队负责人）
{"to": "team-lead", "message": {"type": "shutdown_response", "request_id": "...", "approve": true}}
```

- 团队负责人发送 shutdown_request 以优雅地终止队友
- 队友响应 shutdown_response（批准或拒绝）
- 批准终止会终止队友进程
- 拒绝需要理由

### 计划审批响应

```json
{"to": "researcher", "message": {"type": "plan_approval_response", "request_id": "...", "approve": false, "feedback": "add error handling"}}
```

- 只有团队负责人可以批准/拒绝计划
- 批准继承负责人的权限模式
- 拒绝包括修订的反馈

## 消息路由

### 团队内路由

团队内的消息通过邮箱传递：

1. 发送者调用 SendMessageTool
2. 消息写入接收者的邮箱（`writeToMailbox`）
3. 接收者自动接收消息（无需轮询）
4. 如果接收者正在回合中，消息会排队，等回合结束时传递

### 跨会话路由（UDS_INBOX 功能）

| 目标方案 | 描述 |
|---------------|-------------|
| `"uds:/path/to.sock"` | 本地 Claude 会话的套接字（同一机器） |
| `"bridge:session_..."` | 远程控制对等会话（跨机器） |

跨会话消息：
- 作为 `<cross-session-message from="...">` 到达
- 要回复，将 `from` 属性复制为你的 `to`
- 仅限纯文本（无结构化消息）
- 需要明确用户同意（安全检查，不可绕过）

### 进程内子代理路由

当发送到注册的进程内子代理时：
1. 在 `agentNameRegistry` 中按名称查找代理
2. 如果任务正在运行：为下一个工具轮次排队消息
3. 如果任务已停止：自动恢复代理，消息作为提示
4. 如果没有活动任务：尝试从磁盘 transcripts 恢复

## 权限检查

桥（跨机器）消息需要明确用户同意：

```typescript
{
  behavior: 'ask',
  message: "Send a message to Remote Control session ...?",
  decisionReason: { type: 'safetyCheck', reason: 'Cross-machine bridge message requires explicit user consent' }
}
```

这是安全检查，不是模式检查——无法通过自动模式或 bypassPermissions 绕过。

## 验证规则

| 规则 | 错误 |
|------|-------|
| 空 `to` 字段 | "to must not be empty" |
| 空桥/uds 目标 | "address target must not be empty" |
| `to` 包含 `@` | "to must be a bare teammate name" |
| 结构化消息到桥 | "structured messages cannot be sent cross-session" |
| 桥未连接 | "Remote Control is not connected" |
| 字符串消息没有 summary | "summary is required when message is a string" |
| 广播结构化消息 | "structured messages cannot be broadcast" |
| shutdown_response 到非负责人 | "shutdown_response must be sent to team-lead" |
| 拒绝关闭没有理由 | "reason is required when rejecting a shutdown request" |

## 关闭流程

```
团队负责人                          队友
    |                                 |
    |-- shutdown_request ------------>|
    |                                 |
    |                                 |（决定批准/拒绝）
    |                                 |
|<-- shutdown_response (approve) -----|
    |                                 |
    |（终止队友）                      |（退出）
    |                                 |
```

如果队友批准：
- 进程内：中止控制器收到信号
- 外部：立即调用 gracefulShutdown()

## 计划审批流程

```
队友                           团队负责人
    |                                 |
    |-- plan_approval_request ------->|
    |                                 |
    |                                 |（审查计划）
    |                                 |
|<-- plan_approval_response ---------|
    |   （批准或拒绝 + 反馈）|
    |                                 |
```

只有团队负责人可以批准或拒绝计划。审批响应包括队友应该继承的权限模式。

## 依赖

| 模块 | 用途 |
|--------|---------|
| `utils/teammateMailbox.ts` | 邮箱写入操作 |
| `utils/teammate.ts` | 代理身份助手 |
| `utils/swarm/teamHelpers.ts` | 团队文件读取 |
| `utils/swarm/constants.ts` | TEAM_LEAD_NAME 常量 |
| `utils/agentId.ts` | 请求 ID 生成 |
| `utils/permissions/` | 权限检查 |
| `utils/peerAddress.ts` | 地址解析（uds/桥方案） |
| `utils/gracefulShutdown.ts` | 进程关闭 |
| `bridge/peerSessions.js` | 跨会话消息传递 |
| `utils/udsClient.js` | UDS 套接字通信 |
| `tasks/LocalAgentTask/` | 进程内代理任务管理 |
| `AgentTool/resumeAgent.js` | 代理恢复功能 |
| `services/analytics/` | 事件日志 |
