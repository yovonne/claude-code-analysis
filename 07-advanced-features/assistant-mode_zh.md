# Assistant 模式与会话历史

## 目的

assistant 模块提供会话历史检索功能，支持从远程存储加载和分页历史会话事件。这实现了会话恢复、历史回顾和过去交互的分析。

## 位置

`restored-src/src/assistant/sessionHistory.ts`

## 主要导出

### 类型

- `HistoryPage`：表示分页的会话事件集，包含 `events`（SDKMessage[]）、`firstId`（最旧事件的 ID 游标）和 `hasMore`（是否存在更早的事件）。
- `HistoryAuthCtx`：认证上下文，包含 `baseUrl` 和 `headers`，预计算一次并在页面获取时重用。
- `SessionEventsResponse`：原始 API 响应结构，包含 `data`、`has_more`、`first_id` 和 `last_id` 字段。

### 常量

- `HISTORY_PAGE_SIZE`：默认每页 100 个事件。

### 函数

- `createHistoryAuthCtx(sessionId)`：为给定会话准备认证头和基础 URL 一次。返回可重用的 `HistoryAuthCtx`，包含 OAuth 头、组织 UUID 和 beta 头 `ccr-byoc-2025-07-29`。
- `fetchLatestEvents(ctx, limit)`：获取最新的 `limit` 个事件，锚定到最新点。按时间顺序返回事件；`hasMore=true` 表示存在更早的事件。
- `fetchOlderEvents(ctx, beforeId, limit)`：获取给定 `beforeId` 游标之前的最新事件，支持会话历史的向后分页。

## 依赖

### 内部依赖

- `../constants/oauth.js` — OAuth 配置（基础 API URL）
- `../entrypoints/agentSdkTypes.js` — `SDKMessage` 类型定义
- `../utils/debug.js` — `logForDebugging()` 用于错误日志
- `../utils/teleport/api.js` — `prepareApiRequest()` 用于认证令牌和 `getOAuthHeaders()` 用于头构建

### 外部依赖

- `axios` — HTTP 客户端用于 API 请求

## 实现细节

### 核心逻辑

该模块实现基于游标的会话事件分页系统：

1. **认证上下文准备** — `createHistoryAuthCtx()` 执行 OAuth 令牌获取和头构建一次，返回可重用的上下文对象。这避免了每次页面获取时的冗余认证查找。

2. **页面获取** — `fetchPage()` 是发出带可配置参数的 HTTP GET 请求的内部核心。它使用 `validateStatus: () => true` 来防止 axios 在非 2xx 响应时抛出异常，而是在上游返回 `null` 进行错误处理。

3. **最新优先分页** — `fetchLatestEvents()` 使用 `anchor_to_latest: true` 获取最近的事件。`has_more` 标志指示该页面之外是否存在更早的事件。

4. **向后分页** — `fetchOlderEvents()` 使用 `before_id` 游标获取下一页更早的事件，实现会话历史的向后无限滚动。

### 认证流程

```
createHistoryAuthCtx(sessionId)
    ↓
prepareApiRequest() → accessToken, orgUUID
    ↓
getOAuthHeaders(accessToken)
    ↓
构建 headers:
  - OAuth bearer token
  - anthropic-beta: ccr-byoc-2025-07-29
  - x-organization-uuid: orgUUID
    ↓
返回 { baseUrl, headers }
```

### 错误处理

- 网络错误通过 axios 调用上的 `.catch(() => null)` 捕获
- 非 200 响应通过 `logForDebugging()` 记录，包含 HTTP 状态码
- 无效响应数据（`resp.data.data` 不是数组）默认为空数组
- `validateStatus: () => true` 模式确保所有 HTTP 响应都被优雅处理，而不是抛出异常

## 数据流

```
会话恢复 / 历史请求
    ↓
createHistoryAuthCtx(sessionId)
    ↓ { baseUrl, headers }
    ↓
fetchLatestEvents(ctx, 100)
    ↓
  GET /v1/sessions/{sessionId}/events?limit=100&anchor_to_latest=true
    ↓
HistoryPage { events[], firstId, hasMore }
    ↓
(如果 hasMore) fetchOlderEvents(ctx, firstId, 100)
    ↓
  GET /v1/sessions/{sessionId}/events?limit=100&before_id={firstId}
    ↓
下一个 HistoryPage...
```

## 集成点

- **会话恢复** — 恢复上一会话时加载历史页面以恢复对话上下文
- **会话回顾** — 用户可以浏览过去的会话事件进行分析
- **Teleport API** — 使用 teleport 工具层进行认证和请求准备
- **OAuth 系统** — 依赖有效 OAuth 令牌进行 API 访问

## 配置

| 参数 | 默认值 | 用途 |
|---|---|---|
| `HISTORY_PAGE_SIZE` | 100 | 每页事件数 |
| `timeout` | 15000ms | HTTP 请求超时 |
| `anchor_to_latest` | true (fetchLatestEvents) | 锚定到最新事件 |
| `before_id` | 游标 (fetchOlderEvents) | 分页游标 |

## 测试

- `fetchPage()` 函数接受一个 `label` 参数用于清晰的调试日志，帮助测试诊断
- 错误路径（网络失败、非 200 状态）返回 `null` 而不是抛出异常，使它们易于测试
- 认证上下文可以独立于获取逻辑进行模拟

## 相关模块

- [Session Commands](../03-commands/session-commands.md) — 可能使用会话历史的命令
- [OAuth Service](../04-services/oauth-service.md) — 认证支持
- [Teleport Utils](../05-utils/teleport-utils.md) — API 请求准备

## 备注

- 模块名称 "assistant" 表明这可能是更广泛的 assistant 模式功能的一部分，尽管恢复的源代码中只有会话历史
- `ccr-byoc-2025-07-29` beta 头表明这使用特定 API 版本进行会话事件检索
- 分页设计（锚定到最新 + before-id 游标）针对常见情况优化：先查看最近事件，然后可选地向后遍历
