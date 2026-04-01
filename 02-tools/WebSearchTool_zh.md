# WebSearchTool

## 目的

WebSearchTool 使 Claude 能够执行实时网络搜索并利用结果来为响应提供信息。它与 Anthropic 的原生网络搜索 API（`web_search_20250305`）接口，为时事和最新数据提供最新信息。该工具支持域名过滤（允许/阻止列表），在搜索执行期间流式传输实时进度更新，并强制要求在响应中引用来源。它仅适用于第一方 API、Vertex AI（Claude 4.0+）和 Foundry 提供者。

## 位置

- `restored-src/src/tools/WebSearchTool/WebSearchTool.ts` — 主工具定义和搜索执行逻辑（436 行）
- `restored-src/src/tools/WebSearchTool/prompt.ts` — 带来源引用要求的工具提示（35 行）
- `restored-src/src/tools/WebSearchTool/UI.tsx` — 带进度显示的终端 UI 渲染（101 行）

## 关键导出

### 来自 WebSearchTool.ts

| 导出 | 描述 |
|--------|-------------|
| `WebSearchTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Input` | Zod 推断的输入类型：`{ query, allowed_domains?, blocked_domains? }` |
| `Output` | Zod 推断的输出类型：`{ query, results, durationSeconds }` |
| `SearchResult` | 类型：`{ tool_use_id, content: { title, url }[] }` |
| `WebSearchProgress` | 流式更新的进度数据类型（从 types/tools.js 重新导出） |

## 输入/输出模式

### 输入模式

```typescript
{
  query: string,            // 必需：要使用的搜索查询（最少 2 个字符）
  allowed_domains?: string[],  // 可选：仅包含这些域名的结果
  blocked_domains?: string[],  // 可选：永不包含这些域名的结果
}
```

### 输出模式

```typescript
{
  query: string,                    // 执行的搜索查询
  results: (SearchResult | string)[], // 搜索结果和/或文本评论
  durationSeconds: number,          // 完成搜索所用时间
}
```

## 搜索执行管道

### 执行流程

```
1. 接收输入
   - Model 返回 WebSearch tool_use，附带 { query, allowed_domains?, blocked_domains? }

2. 验证
   - validateInput()：
     a. 查询不能为空
     b. 不能同时指定 allowed_domains 和 blocked_domains

3. 权限检查
   - checkPermissions()：始终返回 'passthrough'（委托给 API 层）
   - 建议在本地设置中添加允许规则

4. 搜索执行（call）
   a. 使用搜索查询创建用户消息
   b. 构建工具模式（BetaWebSearchTool20250305）
   c. 选择模型（基于功能标志的 Haiku 或主模型）
   d. 使用 web_search 工具打开流式查询
   e. 处理流式事件：
      - 跟踪工具使用 ID
      - 累积部分 JSON 以获取进度更新
      - 发出 query_update 进度事件
      - 发出 search_results_received 进度事件
   f. 收集所有内容块

5. 结果处理
   a. 将内容块解析为结构化输出
   b. 提取文本摘要和搜索命中
   c. 计算持续时间
   d. 返回格式化输出
```

### 工具模式构建

发送给 API 的工具模式：

```typescript
{
  type: 'web_search_20250305',
  name: 'web_search',
  allowed_domains: input.allowed_domains,  // 可选域名允许列表
  blocked_domains: input.blocked_domains,  // 可选域名阻止列表
  max_uses: 8,                              // 每会话最大搜索次数（硬编码）
}
```

### 模型选择

| 功能标志 | 使用的模型 | 说明 |
|--------------|-----------|----------|
| `tengu_plum_vx3` 启用 | `getSmallFastModel()` (Haiku) | 禁用思考 |
| `tengu_plum_vx3` 禁用 | `context.options.mainLoopModel` | 按上下文配置 |

### 流式事件处理

工具处理四种类型的流式事件：

| 事件类型 | 处理 |
|------------|----------|
| `assistant` 消息 | 将内容块累积到最终结果 |
| `content_block_start` (server_tool_use) | 跟踪工具使用 ID 以提取查询 |
| `content_block_delta` (input_json_delta) | 累积部分 JSON，提取查询以获取进度更新 |
| `content_block_start` (web_search_tool_result) | 发出带结果计数的 `search_results_received` 进度 |

### 进度更新

通过 `onProgress` 回调发出两种类型的进度事件：

| 类型 | 数据 |
|------|------|
| `query_update` | `{ query }` — 检测到新搜索查询时 |
| `search_results_received` | `{ resultCount, query }` — 搜索结果到达时 |

## 结果解析

### 内容块结构

API 返回块序列：

```
- text（介绍文本，始终存在）
[
  - server_tool_use
  - web_search_tool_result
  - text 和 citation 块混合
]+
（每个搜索重复）
```

### 解析逻辑

`makeOutputFromSearchResponse()` 处理块：

1. 将文本块累积到 `textAcc`
2. 在 `server_tool_use` 时：将累积的文本刷新到结果，重置
3. 在 `web_search_tool_result` 时：提取命中（标题 + url）并添加到结果
4. 在 `text` 时：附加到累加器（在当前文本阶段）或开始新的文本阶段
5. 最终累积的文本被刷新到结果

错误处理：如果 `web_search_tool_result.content` 不是数组（错误情况），则将错误消息字符串推送到结果而不是。

## 提供者可用性

| 提供者 | 启用 | 说明 |
|----------|---------|-------|
| `firstParty` | 始终 | Anthropic 的直接 API |
| `vertex` | 仅 Claude 4.0+ | 需要 `claude-opus-4`、`claude-sonnet-4` 或 `claude-haiku-4` |
| `foundry` | 始终 | 模型已支持网络搜索 |
| 其他 | 禁用 | — |

## 提示要求

工具提示强制要求引用来源：

```
CRITICAL REQUIREMENT - You MUST follow this:
  - After answering the user's question, you MUST include a "Sources:" section
  - List all relevant URLs as markdown hyperlinks: [Title](URL)
  - This is MANDATORY - never skip including sources
```

提示还包括当前月份/年份以进行准确的搜索查询，并指出网络搜索仅在美国可用。

## 工具结果格式化

`mapToolResultToToolResultBlockParam()` 格式化输出给模型：

```
Web search results for query: "<query>"

<text summaries>
Links: [{"title": "...", "url": "..."}]

REMINDER: You MUST include the sources above in your response to the user using markdown hyperlinks.
```

结果数组中的 null/undefined 条目被静默跳过（防止 JSON 往返问题导致的压缩或 transcript 反序列化）。

## 配置

### 功能标志

| 标志 | 目的 |
|------|---------|
| `tengu_plum_vx3` | 启用时，对搜索使用 Haiku 模型并禁用思考 |

### 限制

| 限制 | 值 | 描述 |
|-------|-------|-------------|
| `max_uses` | 8 | 每会话最大网络搜索次数（硬编码） |
| `maxResultSizeChars` | 100,000 | 存储在上下文中的最大结果大小 |
| 查询最小长度 | 2 个字符 | 最小查询长度 |

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `services/api/claude.js` | 模型查询流式处理 |
| `services/analytics/growthbook.js` | 功能标志访问 |
| `utils/model/model.js` | 模型选择工具 |
| `utils/slowOperations.js` | JSON 解析/字符串化 |
| `utils/messages.js` | 消息创建 |
| `utils/systemPromptType.js` | 系统提示包装器 |
| `constants/common.js` | 月份/年份格式化 |

### 外部

| 包 | 目的 |
|---------|---------|
| `@anthropic-ai/sdk` | Web 搜索工具类型 |
| `zod/v4` | 输入/输出模式验证 |
| `react` | UI 渲染 |

## 数据流

```
Model 请求
    |
    v
WebSearchTool 输入 { query, allowed_domains?, blocked_domains? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 验证                                                               │
│ - 查询不为空                                                       │
│ - 不能同时指定 allowed_domains 和 blocked_domains                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 权限检查                                                           │
│  - 始终直通（委托给 API 层）                                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 搜索执行（流式）                                                   │
│                                                                 │
│  1. 构建 web_search 工具模式                                     │
│  2. 选择模型（Haiku 或主模型）                                    │
│  3. 打开流式查询                                                  │
│  4. 处理事件：                                                    │
│     - 跟踪工具使用 ID                                             │
│     - 从部分 JSON 提取查询                                        │
│     - 发出进度更新                                                │
│     - 收集内容块                                                  │
│  5. 将结果解析为 SearchResult | string[]                         │
└─────────────────────────────────────────────────────────────────┘
    |
    v
输出 { query, results[], durationSeconds }
    |
    v
Model 响应（带格式化链接和来源提醒的 tool_result）
```
