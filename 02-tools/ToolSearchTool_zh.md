# ToolSearchTool

## 目的

ToolSearchTool 是 Claude Code 中延迟加载工具的发现机制。当工具被延迟加载（不在启动时加载以节省上下文窗口空间）时，模型只能看到它们的名称。ToolSearchTool 允许模型查询并获取这些延迟工具的完整模式定义，以便可以调用它们。它支持直接选择（`select:ToolName`）和基于关键字的搜索，搜索分数跨越工具名称、搜索提示和描述。工具将匹配的工具作为 `tool_reference` 块返回，SDK 可以将其扩展为完整模式。

## 位置

- `restored-src/src/tools/ToolSearchTool/ToolSearchTool.ts` — 主工具定义和搜索逻辑（472 行）
- `restored-src/src/tools/ToolSearchTool/prompt.ts` — 工具提示和延迟工具分类（122 行）
- `restored-src/src/tools/ToolSearchTool/constants.ts` — 工具名称常量（2 行）

## 关键导出

### 来自 ToolSearchTool.ts

| 导出 | 描述 |
|--------|-------------|
| `ToolSearchTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `inputSchema` | 工具输入的 Zod 模式 |
| `outputSchema` | 工具输出的 Zod 模式 |
| `Output` | Zod 推断的输出类型：`{ matches, query, total_deferred_tools, pending_mcp_servers? }` |
| `clearToolSearchDescriptionCache()` | 清除记忆化的工具描述缓存 |

### 来自 prompt.ts

| 导出 | 描述 |
|--------|-------------|
| `TOOL_SEARCH_TOOL_NAME` | 常量：`'ToolSearch'` |
| `isDeferredTool(tool)` | 确定工具是否应被延迟（需要 ToolSearch 加载） |
| `formatDeferredToolLine(tool)` | 为 `<available-deferred-tools>` 消息格式化延迟工具名称 |
| `getPrompt()` | 返回完整的工具提示 |

## 输入/输出模式

### 输入模式

```typescript
{
  query: string,       // 必需：查找延迟工具的查询
  max_results?: number, // 可选：返回的最大结果数（默认：5）
}
```

查询格式：
- `select:Read,Edit,Grep` — 精确工具名称的直接选择（逗号分隔）
- `notebook jupyter` — 关键字搜索，返回最多 `max_results` 个最佳匹配
- `+slack send` — 要求名称中包含 "slack"，按剩余项排名

### 输出模式

```typescript
{
  matches: string[],                // 匹配的 tool 名称
  query: string,                    // 原始查询
  total_deferred_tools: number,     // 可用的延迟工具总数
  pending_mcp_servers?: string[],   // 仍在连接的 MCP 服务器（无匹配时）
}
```

## 延迟工具概念

### 什么是延迟工具？

延迟工具是初始系统提示中**不**包含其完整模式定义的工具。只在 `<available-deferred-tools>` 块中显示它们的名称。这在有许多工具（尤其是 MCP 工具）可用时节省上下文窗口空间。

### 延迟工具分类

如果满足以下任一条件，工具被延迟：

| 条件 | 结果 |
|-----------|--------|
| `tool.alwaysLoad === true` | **永不延迟**（通过 `_meta['anthropic/alwaysLoad']` 明确选择退出） |
| `tool.isMcp === true` | **始终延迟**（MCP 工具是特定于工作流的） |
| `tool.name === 'ToolSearch'` | **永不延迟**（需要它来加载其他工具） |
| `tool.name === 'Agent'` + Fork Subagent 启用 | **永不延迟**（必须在第 1 回合可用） |
| `tool.name === 'BriefTool'` + Kairos 启用 | **永不延迟**（主要通信渠道） |
| `tool.name === 'SendUserFile'` + Kairos + Bridge 活跃 | **永不延迟**（文件传送渠道） |
| `tool.shouldDefer === true` | **延迟** |

### 工具位置提示

提示告诉模型延迟工具名称出现的位置：

| 模式 | 消息 |
|------|---------|
| Delta 启用（A/B 测试或 ant） | "Deferred tools appear by name in `<system-reminder>` messages." |
| Delta 禁用（遗留） | "Deferred tools appear by name in `<available-deferred-tools>` messages." |

## 搜索执行管道

### 执行流程

```
1. 接收输入
   - Model 返回 ToolSearch tool_use，附带 { query, max_results? }

2. 延迟工具集合
   - 按 isDeferredTool() 过滤所有工具
   - 延迟工具更改时使缓存失效

3. 待处理 MCP 服务器检查
   - 检查 AppState.mcp.clients 中的 'pending' 类型
   - 无匹配时在结果中包含待处理服务器名称

4. 查询解析
   a. 检查 select: 前缀
   b. 如果是 select: 解析逗号分隔的工具名称
   c. 否则：关键字搜索

5. 选择模式（select:ToolName）
   a. 按逗号拆分，trim 每个名称
   b. 在延迟工具中查找每个名称，然后在所有工具中查找
   c. 收集找到的名称和缺失的名称
   d. 如果未找到任何内容 → 返回空匹配
   e. 返回找到的工具名称作为匹配

6. 关键字搜索模式
   a. 检查精确名称匹配（先延迟，然后所有工具）
   b. 检查 MCP 前缀匹配（mcp__server）
   c. 将查询解析为术语（必需 + 可选）
   d. 按必需术语预过滤
   e. 对剩余工具评分
   f. 按分数排序，取前 max_results

7. 结果格式化
   - 将匹配作为 tool_reference 块返回
   - 无匹配时包含待处理 MCP 服务器
   - 记录分析事件
```

### 选择模式

通过 `select:` 前缀直接选择工具：

```
select:Read          → 单个工具
select:Read,Edit     → 多个工具
select:mcp__github__list_issues  → MCP 工具
```

- 支持逗号分隔名称进行多选
- 先在延迟工具中查找名称，然后在所有工具中查找
- 如果名称不是延迟的但是**在**完整工具集中，仍会返回（无害的空操作）
- 缺失的名称被记录但不会阻止返回找到的工具

### 关键字搜索

关键字搜索使用加权评分系统：

| 匹配类型 | MCP 分数 | 普通分数 |
|------------|-----------|---------------|
| 精确部分匹配 | 12 | 10 |
| 部分包含术语 | 6 | 5 |
| 全名包含术语（回退） | 3 | 3 |
| searchHint 匹配 | 4 | 4 |
| 描述匹配（单词边界） | 2 | 2 |

#### 术语解析

查询术语解析为必需和可选：

| 语法 | 类型 |
|--------|------|
| `+term` | 必需（必须匹配） |
| `term` | 可选（匹配则排名更高） |

#### 工具名称解析

工具名称拆分为可搜索的部分：

| 名称格式 | 解析 |
|-------------|---------|
| `mcp__server__action` | 按 `__` 和 `_` 拆分：`["server", "action"]` |
| `CamelCaseTool` | 按驼峰命名拆分：`["camel", "case", "tool"]` |
| `snake_case_tool` | 按 `_` 拆分：`["snake", "case", "tool"]` |

#### 必需术语预过滤

当存在必需术语（`+term`）时，工具被预过滤为仅匹配所有必需术语：
- 解析的名称部分
- 部分子字符串匹配
- 描述（单词边界正则表达式）
- searchHint（单词边界正则表达式）

这避免了对不可能匹配的工具进行评分。

### 缓存

工具描述通过工具名称记忆化以避免重复的昂贵 `tool.prompt()` 调用：

```typescript
const getToolDescriptionMemoized = memoize(
  async (toolName, tools) => tool.prompt(...),
  (toolName) => toolName  // 缓存键
)
```

缓存失效：
- `maybeInvalidateCache()` 检查延迟工具集是否已更改
- 如果更改，清除记忆化缓存并更新缓存键
- `clearToolSearchDescriptionCache()` 提供手动缓存清除

## 结果格式化

### 工具引用块

找到匹配时，结果作为 `tool_reference` 块返回：

```json
{
  "type": "tool_result",
  "tool_use_id": "<id>",
  "content": [
    { "type": "tool_reference", "tool_name": "Read" },
    { "type": "tool_reference", "tool_name": "Edit" }
  ]
}
```

SDK 将这些引用扩展为完整工具模式。

### 无匹配

当无匹配时：

```
No matching deferred tools found. Some MCP servers are still connecting:
server1, server2. Their tools will become available shortly — try searching again.
```

## 分析

每次搜索记录 `tengu_tool_search_outcome` 事件：

| 字段 | 描述 |
|-------|---------|
| `query` | 搜索查询 |
| `queryType` | `'select'` 或 `'keyword'` |
| `matchCount` | 返回的匹配数量 |
| `totalDeferredTools` | 可用的延迟工具总数 |
| `maxResults` | 请求的最大结果数 |
| `hasMatches` | 是否找到任何匹配 |

## 权限处理

ToolSearchTool 没有定义显式的 `checkPermissions` 方法。它从工具框架继承默认权限行为。由于它是一个只读发现工具，它有效地在无权限障碍的情况下运作。

## 工具可用性

| 条件 | 效果 |
|-----------|---------|
| `isToolSearchEnabledOptimistic()` 返回 false | 工具被禁用 |

## 配置

### 限制

| 限制 | 值 | 描述 |
|-------|-------|-------------|
| `max_results` 默认值 | 5 | 默认最大结果数 |
| `maxResultSizeChars` | 100,000 | 存储在上下文中的最大结果大小 |

### 功能标志

| 标志 | 目的 |
|------|---------|
| `KAIROS` / `KAIROS_BRIEF` | 影响 Brief 工具延迟 |
| `KAIROS` | 影响 SendUserFile 工具延迟 |
| `FORK_SUBAGENT` | 影响 Agent 工具延迟 |
| `tengu_glacier_2xr` | 控制延迟工具位置提示（system-reminder 与 available-deferred-tools） |

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `Tool.js` | 工具构建，findToolByName |
| `utils/toolSearch.js` | 工具搜索启用检查 |
| `utils/debug.js` | 调试日志 |
| `utils/stringUtils.js` | 正则表达式转义 |
| `services/analytics/` | 事件记录 |
| `bootstrap/state.js` | ReplBridge 状态 |
| `services/analytics/growthbook.js` | 功能标志 |
| `tools/AgentTool/constants.js` | Agent 工具名称 |

### 外部

| 包 | 目的 |
|---------|---------|
| `@anthropic-ai/sdk` | ToolResultBlockParam 类型 |
| `lodash-es/memoize.js` | 工具描述记忆化 |
| `zod/v4` | 输入/输出模式验证 |

## 数据流

```
Model 请求
    |
    v
ToolSearchTool 输入 { query, max_results? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 延迟工具集合                                                      │
│  - 按 isDeferredTool() 过滤工具                                  │
│  - 延迟工具更改时使缓存失效                                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 查询解析                                                          │
│                                                                 │
│  有 select: 前缀？                                               │
│    → 按逗号拆分，查找每个名称                                     │
│    → 返回找到的名称                                              │
│                                                                 │
│  否则：关键字搜索                                                 │
│    → 解析术语（+必需，可选）                                     │
│    → 按必需术语预过滤                                           │
│    → 对工具评分（名称部分、提示、描述）                          │
│    → 按分数排序，取前 max_results                                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 结果格式化                                                        │
│                                                                 │
│  找到匹配：                                                       │
│    → 返回 tool_reference 块                                      │
│                                                                 │
│  无匹配：                                                         │
│    → 包含待处理 MCP 服务器信息                                    │
│    → 返回文本消息                                                │
│                                                                 │
│  记录分析事件                                                     │
└─────────────────────────────────────────────────────────────────┘
    |
    v
输出 { matches[], query, total_deferred_tools, pending_mcp_servers? }
    |
    v
Model 响应（带 tool_reference 块的 tool_result）
```
