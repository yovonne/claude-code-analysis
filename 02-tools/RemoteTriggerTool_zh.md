# RemoteTriggerTool

## 用途

RemoteTriggerTool 通过 claude.ai CCR API 管理预定的远程 Claude Code 代理（触发器）。它提供类型化接口来列出、创建、更新、检索和运行远程触发器，同时不向 shell 暴露 OAuth 令牌。所有认证都通过工具内部 OAuth 流程在进程内处理。

## 位置

- `restored-src/src/tools/RemoteTriggerTool/RemoteTriggerTool.ts` — 主要工具定义（162 行）
- `restored-src/src/tools/RemoteTriggerTool/prompt.ts` — 工具提示、描述和常量
- `restored-src/src/tools/RemoteTriggerTool/UI.tsx` — 工具使用/结果消息的 UI 渲染

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `RemoteTriggerTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | Zod 推断的输入类型：`{ action, trigger_id?, body? }` |
| `Output` | Zod 推断的输出类型：`{ status, json }` |
| `REMOTE_TRIGGER_TOOL_NAME` | 常量：`'RemoteTrigger'` |
| `DESCRIPTION` | 工具描述字符串 |
| `PROMPT` | 工具提示字符串 |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  action: 'list' | 'get' | 'create' | 'update' | 'run',
  trigger_id?: string,    // 必需用于：get, update, run。正则：/^[\w-]+$/
  body?: Record<string, unknown>,  // 必需用于：create, update
}
```

### 输出 Schema

```typescript
{
  status: number,    // 来自 API 的 HTTP 状态码
  json: string,      // JSON 响应体（字符串化）
}
```

## 操作

### list

列出组织的所有远程触发器。

```json
{ "action": "list" }
```

- **方法**: GET
- **端点**: `/v1/code/triggers`
- **需要**: `trigger_id` — 否，`body` — 否

### get

按 ID 检索特定触发器。

```json
{ "action": "get", "trigger_id": "my-trigger-id" }
```

- **方法**: GET
- **端点**: `/v1/code/triggers/{trigger_id}`
- **需要**: `trigger_id` — 是

### create

创建新的远程触发器。

```json
{ "action": "create", "body": { "name": "daily-check", "schedule": "0 9 * * *" } }
```

- **方法**: POST
- **端点**: `/v1/code/triggers`
- **需要**: `body` — 是

### update

更新现有触发器（部分更新）。

```json
{ "action": "update", "trigger_id": "my-trigger-id", "body": { "schedule": "0 10 * * *" } }
```

- **方法**: POST
- **端点**: `/v1/code/triggers/{trigger_id}`
- **需要**: `trigger_id` — 是，`body` — 是

### run

手动触发预定触发器。

```json
{ "action": "run", "trigger_id": "my-trigger-id" }
```

- **方法**: POST
- **端点**: `/v1/code/triggers/{trigger_id}/run`
- **需要**: `trigger_id` — 是

## API 通信

### 请求构建

```typescript
const base = `${getOauthConfig().BASE_API_URL}/v1/code/triggers`
const headers = {
  Authorization: `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': 'ccr-triggers-2026-01-30',
  'x-organization-uuid': orgUUID,
}
```

### 认证流程

1. 检查并在需要时刷新 OAuth 令牌（`checkAndRefreshOAuthTokenIfNeeded`）
2. 获取访问令牌（`getClaudeAIOAuthTokens`）
3. 解析组织 UUID（`getOrganizationUUID`）
4. 如果未认证：抛出错误，引导用户运行 `/login`

### HTTP 客户端

使用 `axios`：
- 20 秒超时
- Abort controller 信号支持
- `validateStatus: () => true`（接受所有状态码）
- 响应返回为 `{ status, json }`

## 功能门控

仅当两个条件都满足时工具才启用：

```typescript
isEnabled() {
  return (
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali', false) &&
    isPolicyAllowed('allow_remote_sessions')
  )
}
```

| 门控 | 用途 |
|------|---------|
| `tengu_surreal_dali` | 远程触发器的 GrowthBook 功能标志 |
| `allow_remote_sessions` | 组织策略限制 |

## 并发和只读分类

- **并发安全**：是 — 所有操作都是幂等 API 调用
- **只读**：`list` 和 `get` 操作被分类为只读
- **非只读**：`create`、`update`、`run` 操作会修改状态

## 结果格式化

使用 `mapToolResultToToolResultBlockParam` 格式化结果：

```
HTTP {status}
{json response}
```

UI 渲染结果显示 HTTP 状态和行数：

```
HTTP 200 (15 lines)
```

## 错误处理

| 错误条件 | 响应 |
|----------------|----------|
| 未认证 | "Not authenticated with a claude.ai account. Run /login and try again." |
| 无法解析组织 | "Unable to resolve organization UUID." |
| 缺少 trigger_id | 抛出错误，说明特定操作要求 |
| 缺少 body | 抛出错误，说明特定操作要求 |
| API 错误 | 返回 HTTP 状态 + 错误 JSON（不抛出，validateStatus 接受所有） |

## 安全考虑

- **OAuth 令牌从不暴露到 shell**：令牌通过头在进程内添加
- **无 shell 执行**：使用 axios HTTP 客户端，而非 shell 命令
- **组织范围**：所有操作都包含 `x-organization-uuid` 头
- **Beta API**：使用 `anthropic-beta: ccr-triggers-2026-01-30` 头

## 依赖

| 模块 | 用途 |
|--------|---------|
| `utils/auth.ts` | OAuth 令牌管理（检查、刷新、获取） |
| `constants/oauth.ts` | OAuth 配置（BASE_API_URL） |
| `services/oauth/client.ts` | 组织 UUID 解析 |
| `services/policyLimits/` | 组织策略限制检查 |
| `services/analytics/growthbook.ts` | 功能标志评估 |
| `utils/slowOperations.ts` | jsonStringify 用于响应格式化 |
| `axios` | API 请求的 HTTP 客户端 |
| `zod/v4` | 输入/输出 schema 验证 |
