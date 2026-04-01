# ListMcpResourcesTool

## 用途

ListMcpResourcesTool 是一个内置工具，用于从所有连接的 MCP 服务器发现和列出可用资源。它查询每个服务器的 `resources/list` 端点，聚合带有服务器归属的结果，并返回结构化资源元数据，包括 URI、名称、MIME 类型和描述。该工具自动添加到第一个声明资源能力的 MCP 服务器，使模型能够探索 MCP 生态系统中可用的数据源。

## 位置

- `restored-src/src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts` — 主要工具定义和资源获取（124 行）
- `restored-src/src/tools/ListMcpResourcesTool/UI.tsx` — 终端 UI 渲染（29 行）
- `restored-src/src/tools/ListMcpResourcesTool/prompt.ts` — 工具描述和提示（21 行）
- `restored-src/src/services/mcp/client.ts` — 资源获取（`fetchResourcesForClient`）、连接管理（约 2800 行）
- `restored-src/src/services/mcp/types.ts` — `ServerResource` 类型定义（259 行）

## 主要导出

### 来自 ListMcpResourcesTool.ts

| 导出 | 描述 |
|--------|-------------|
| `ListMcpResourcesTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | Zod 推断的输出类型：资源对象数组 |

### 来自 prompt.ts

| 导出 | 描述 |
|--------|-------------|
| `LIST_MCP_RESOURCES_TOOL_NAME` | 常量：`'ListMcpResourcesTool'` |
| `DESCRIPTION` | 模型上下文的工具描述 |
| `PROMPT` | 模型的详细使用说明 |

### 来自 UI.tsx

| 导出 | 描述 |
|--------|-------------|
| `renderToolUseMessage(input)` | 渲染工具调用消息 |
| `renderToolResultMessage(output, _, { verbose })` | 将资源列表渲染为格式化的 JSON |

### 来自 client.ts（资源获取）

| 导出 | 描述 |
|--------|-------------|
| `fetchResourcesForClient(client)` | 通过 `resources/list` 进行 LRU 缓存的资源发现 |
| `ServerResource` | 类型：`Resource & { server: string }` |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  server?: string,  // 可选：过滤到特定服务器的资源
}
```

| 参数 | 必需 | 描述 |
|-----------|----------|-------------|
| `server` | 否 | 要按其过滤的服务器名称。如果省略，返回所有服务器的资源。 |

### 输出 Schema

```typescript
[
  {
    uri: string,           // 资源 URI（唯一标识符）
    name: string,          // 人类可读的资源名称
    mimeType?: string,     // 资源内容的 MIME 类型
    description?: string,  // 资源描述
    server: string,        // 提供此资源的服务器
  },
  // ... 更多资源
]
```

## MCP 资源列表

### 资源发现管道

```
1. 工具调用
   - 模型使用可选的 { server } 调用 ListMcpResourcesTool
   - 根据 Zod schema 验证输入

2. 服务器过滤
   - 如果指定了服务器：
     - 过滤 mcpClients 到匹配的服务器名称
     - 如果服务器未找到则抛出（在错误中列出可用服务器）
   - 如果未指定服务器：
     - 处理所有连接的 mcpClients

3. 并行资源获取
   - Promise.all() 跨所有目标服务器：
     a. ensureConnectedClient(client) — 验证/重新连接
     b. fetchResourcesForClient(fresh) — 查询 resources/list
   - 单个服务器失败不会导致整个结果失败
     （捕获并记录，为该服务器返回空数组）

4. 结果聚合
   - results.flat() — 合并所有服务器数组
   - 每个资源已有来自 fetchResourcesForClient 的服务器字段
   - 返回 { data: flattenedArray }

5. 缓存行为
   - fetchResourcesForClient 按服务器名称进行 LRU 缓存
   - 最多 20 个条目（MCP_FETCH_CACHE_SIZE）
   - 在以下情况下缓存失效：
     a. client.onclose（连接断开）
     b. 服务器的 resources/list_changed 通知
     c. clearServerCache()（手动重新连接）
   - 从启动预取预热（prefetchAllMcpResources）
```

### fetchResourcesForClient 实现

```typescript
fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (client.type !== 'connected') return []

    if (!client.capabilities?.resources) return []

    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema,
    )

    if (!result.resources) return []

    // 为每个资源添加服务器名称
    return result.resources.map(resource => ({
      ...resource,
      server: client.name,
    }))
  },
  (client) => client.name,  // 缓存键
  MCP_FETCH_CACHE_SIZE,     // 最多 20 个条目
)
```

### MCP 协议：resources/list

```
请求：
{
  jsonrpc: "2.0",
  id: <number>,
  method: "resources/list",
  params: {}
}

响应：
{
  jsonrpc: "2.0",
  id: <number>,
  result: {
    resources: [
      {
        uri: string,           // 唯一资源标识符
        name: string,          // 人类可读名称
        description?: string,  // 可选描述
        mimeType?: string,     // 可选 MIME 类型
      }
    ],
    nextCursor?: string        // 分页游标（如果有更多资源）
  }
}
```

## 资源发现

### 何时发现资源

```
1. 初始连接
   - connectToServer() 建立连接
   - 检查服务器能力：capabilities?.resources
   - 如果存在资源能力：
     a. 第一个这样的服务器触发资源工具添加
     b. ListMcpResourcesTool 添加到工具数组
     c. ReadMcpResourceTool 添加到工具数组
     d. resourceToolsAdded 标志防止重复添加

2. 服务器重新连接
   - 认证或配置更改后 reconnectMcpServerImpl()：
     a. 通过 fetchResourcesForClient 获取资源
     b. 检查资源工具是否已存在
     c. 如果不存在则添加（hasResourceTools 检查）

3. 资源更改通知
   - 服务器发送 resources/list_changed 通知
   - 缓存失效：fetchResourcesForClient.cache.delete(name)
   - 下一次 ListMcpResourcesTool 调用获取新数据

4. 启动预取
   - 在启动时调用 prefetchAllMcpResources()
   - 为所有服务器预热资源缓存
   - 避免第一次工具调用的冷延迟
```

### 服务器能力检测

```typescript
// 来自 connectToServer()：
const capabilities = client.getServerCapabilities()
// capabilities.resources 表示服务器支持资源

// 来自 getMcpToolsCommandsAndResources()：
const supportsResources = !!client.capabilities?.resources
if (supportsResources && !resourceToolsAdded) {
  resourceToolsAdded = true
  resourceTools.push(ListMcpResourcesTool, ReadMcpResourceTool)
}
```

### 资源工具添加逻辑

```
只有一个服务器获取资源工具（第一个具有资源能力的）：

1. resourceToolsAdded = false（模块级标志）
2. 对于每个按连接顺序的服务器：
   a. 如果服务器支持资源 AND !resourceToolsAdded：
      - 添加 ListMcpResourcesTool
      - 添加 ReadMcpResourceTool
      - 设置 resourceToolsAdded = true
   b. 具有资源的后续服务器：跳过添加

这确保了：
- 工具列表中没有重复的资源工具
- 只要任何服务器有资源，资源工具就可用
- 模型跨所有服务器看到统一的资源接口
```

## 元数据检索

### 资源对象结构

工具返回的每个资源包括：

| 字段 | 类型 | 来源 | 描述 |
|-------|------|--------|-------------|
| `uri` | string | MCP 服务器 | 唯一资源标识符（用于读取资源） |
| `name` | string | MCP 服务器 | 人类可读名称 |
| `mimeType` | string? | MCP 服务器 | MIME 类型（例如 `application/json`、`text/plain`） |
| `description` | string? | MCP 服务器 | 资源的可选描述 |
| `server` | string | 由客户端添加 | 提供此资源的服务器名称 |

### 空结果处理

```
当没有找到资源时：
- mapToolResultToToolResultBlockParam 返回：
  "No resources found. MCP servers may still provide tools even if they have no resources."

此消息澄清了：
- 资源和工具是独立的 MCP 能力
- 服务器可以没有资源而有工具
- 资源缺失并不表示有问题
```

## UI 渲染

### 工具使用消息

```
renderToolUseMessage(input)：
  - 带服务器：'List MCP resources from server "<server>"'
  - 无服务器：'List all MCP resources'
```

### 工具结果消息

```
renderToolResultMessage(output, _, { verbose })：
  - 空：在暗淡颜色中显示 '(No resources found)'
  - 非空：通过 OutputLine 显示 jsonStringify(output, null, 2)
    - 带 2 空格缩进的漂亮打印 JSON
    - OutputLine 组件处理截断
```

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `MCP_FETCH_CACHE_SIZE` | 资源获取的 LRU 缓存条目最大数量（默认 20） |

### 工具属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'ListMcpResourcesTool'` | 内部工具名称 |
| `userFacingName` | `'listMcpResources'` | UI 中的显示名称 |
| `isConcurrencySafe` | `true` | 只读，安全并发调用 |
| `isReadOnly` | `true` | 不修改服务器状态 |
| `shouldDefer` | `true` | 可以在 auto 模式中延迟 |
| `maxResultSizeChars` | `100,000` | 截断前的最大结果大小 |
| `searchHint` | `'list resources from connected MCP servers'` | 自动模式搜索提示 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `services/mcp/client.ts` | 资源获取、客户端连接管理 |
| `services/mcp/types.ts` | ServerResource 类型、连接类型 |
| `utils/errors.ts` | 错误消息提取 |
| `utils/log.ts` | MCP 错误日志 |
| `utils/slowOperations.ts` | JSON 字符串化 |
| `utils/terminal.ts` | 输出截断检测 |
| `utils/lazySchema.ts` | 延迟 Zod schema 评估 |
| `components/MessageResponse.js` | UI 消息组件 |
| `components/shell/OutputLine.js` | 终端输出渲染 |
| `ink.js` | Ink 终端 UI 原语 |

### 外部

| 包 | 用途 |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP 类型、ListResourcesResultSchema |
| `zod/v4` | 输入/输出 schema 验证 |
| `react` | UI 渲染 |

## 数据流

```
Model Request
    │
    ▼
ListMcpResourcesTool Input { server?: string }
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ RESOURCE DISCOVERY                                               │
│                                                                 │
│  1. Server filtering                                            │
│     - If server specified: filter to matching server            │
│     - If not: use all mcpClients                                │
│                                                                 │
│  2. Parallel fetch (Promise.all)                                │
│     For each server:                                            │
│     a. ensureConnectedClient() — verify connection              │
│     b. fetchResourcesForClient() — LRU-cached                  │
│        - If cached: return immediately                          │
│        - If not: client.request({ method: 'resources/list' })   │
│        - Add server field to each resource                      │
│     c. On error: log and return []                              │
│                                                                 │
│  3. Aggregate results                                           │
│     - results.flat() — merge all server arrays                  │
│     - Return { data: flattenedArray }                           │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Output [ { uri, name, mimeType?, description?, server }, ... ]
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ UI RENDERING                                                    │
│                                                                 │
│  renderToolResultMessage:                                       │
│  - Empty: '(No resources found)'                                │
│  - Non-empty: Pretty-printed JSON (2-space indent)            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Model Response (tool_result block with resource list)
```
