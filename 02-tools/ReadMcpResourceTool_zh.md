# ReadMcpResourceTool

## 用途

ReadMcpResourceTool 是一个内置工具，用于通过 URI 从 MCP 服务器读取特定资源。它向目标服务器发送 `resources/read` 请求，处理文本和二进制内容类型，将二进制 blob 持久化到具有 MIME 派生扩展名的磁盘，并返回带有 URI、MIME 类型以及文本内容或二进制数据文件路径的结构化内容。该工具使模型能够访问 MCP 服务器作为结构化资源公开的数据源。

## 位置

- `restored-src/src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` — 主要工具定义和资源读取（159 行）
- `restored-src/src/tools/ReadMcpResourceTool/UI.tsx` — 终端 UI 渲染（37 行）
- `restored-src/src/tools/ReadMcpResourceTool/prompt.ts` — 工具描述和提示（17 行）
- `restored-src/src/services/mcp/client.ts` — 客户端连接、结果转换（约 2800 行）
- `restored-src/src/utils/mcpOutputStorage.ts` — 二进制内容持久化（190 行）

## 主要导出

### 来自 ReadMcpResourceTool.ts

| 导出 | 描述 |
|--------|-------------|
| `ReadMcpResourceTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `inputSchema` | 延迟 Zod schema：`{ server: string, uri: string }` |
| `outputSchema` | 延迟 Zod schema：`{ contents: [{ uri, mimeType?, text?, blobSavedTo? }] }` |
| `Output` | Zod 推断的输出类型 |

### 来自 prompt.ts

| 导出 | 描述 |
|--------|-------------|
| `DESCRIPTION` | 模型上下文的工具描述 |
| `PROMPT` | 模型的详细使用说明 |

### 来自 UI.tsx

| 导出 | 描述 |
|--------|-------------|
| `renderToolUseMessage(input)` | 渲染工具调用消息 |
| `userFacingName()` | 返回 `'readMcpResource'` |
| `renderToolResultMessage(output, _, { verbose })` | 将资源内容渲染为格式化的 JSON |

### 来自 mcpOutputStorage.ts（二进制持久化）

| 导出 | 描述 |
|--------|-------------|
| `persistBinaryContent(bytes, mimeType, persistId)` | 将二进制写入磁盘，返回文件路径 |
| `getBinaryBlobSavedMessage(filepath, mimeType, size, sourceDescription)` | 格式化保存消息 |
| `extensionForMimeType(mimeType)` | 将 MIME 类型映射到文件扩展名 |
| `isBinaryContentType(contentType)` | 二进制与文本内容的启发式判断 |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  server: string,   // 必需：要从中读取的 MCP 服务器名称
  uri: string,      // 必需：要读取的特定资源的 URI
}
```

| 参数 | 必需 | 描述 |
|-----------|----------|-------------|
| `server` | 是 | 要从中读取的 MCP 服务器名称 |
| `uri` | 是 | 要读取的特定资源的 URI |

### 输出 Schema

```typescript
{
  contents: [
    {
      uri: string,              // 资源 URI
      mimeType?: string,        // 内容的 MIME 类型
      text?: string,            // 文本内容（用于文本资源）
      blobSavedTo?: string,     // 二进制保存到的文件路径（用于二进制资源）
    },
    // ... 更多内容块（资源可以返回多个内容）
  ],
}
```

## 资源解析

### 资源读取管道

```
1. 工具调用
   - 模型使用 { server, uri } 调用 ReadMcpResourceTool
   - 根据 Zod schema 验证输入

2. 服务器查找
   - 在 mcpClients 中按服务器名称查找客户端
   - 如果服务器未找到则抛出（列出可用服务器）
   - 如果服务器未连接则抛出（type !== 'connected'）
   - 如果服务器不支持资源则抛出（!capabilities?.resources）

3. 连接保证
   - ensureConnectedClient(client)
   - 如果备忘录缓存被清除（onclose）则重新连接
   - 如果重新连接失败则抛出

4. 资源请求
   - connectedClient.client.request({
       method: 'resources/read',
       params: { uri }
     }, ReadResourceResultSchema)
   - 返回带有 contents 数组的 ReadResourceResult

5. 内容拦截
   - 对于 result.contents 中的每个内容块：
     a. 文本内容（'text' in c）：
        - 直接通过：{ uri, mimeType, text }
     b. 二进制内容（'blob' in c）：
        - 解码 base64：Buffer.from(c.blob, 'base64')
        - 生成 persistId：mcp-resource-<timestamp>-<index>-<random>
        - 持久化到磁盘：persistBinaryContent(bytes, mimeType, persistId)
        - 替换为：{ uri, mimeType, blobSavedTo, text: savedMessage }
     c. 未知内容：
        - 直接通过：{ uri, mimeType }

6. 结果返回
   - { data: { contents: processedContentArray } }
```

### MCP 协议：resources/read

```
请求：
{
  jsonrpc: "2.0",
  id: <number>,
  method: "resources/read",
  params: {
    uri: string    // 要读取的资源 URI
  }
}

响应：
{
  jsonrpc: "2.0",
  id: <number>,
  result: {
    contents: [
      {
        uri: string,
        mimeType?: string,
        // 要么是 text 要么是 blob：
        text?: string,     // 文本内容
        // 或者
        blob?: string,     // Base64 编码的二进制内容
      }
    ]
  }
}
```

## 内容提取

### 文本内容处理

当资源返回文本内容时，处理很简单：

1. 检查 `'text' in c` 返回 true
2. 原样返回：{ uri, mimeType, text }
3. 文本直接包含在工具结果中
4. 不需要文件持久化

示例：
```
{
  uri: "resource://example/data",
  mimeType: "application/json",
  text: '{"key": "value"}'
}
```

### 二进制内容处理

当资源返回二进制内容（blob）时，内容被拦截并持久化到磁盘：

1. 检查 `'blob' in c` 返回 true
2. 解码 base64：`Buffer.from(c.blob, 'base64')`
3. 生成唯一 persistId：`mcp-resource-<timestamp>-<index>-<random>`
4. 通过 `persistBinaryContent(bytes, mimeType, persistId)` 持久化到磁盘：
   - 从 MIME 类型确定扩展名（application/pdf -> .pdf 等）
   - 写入 tool-results 目录：`<tool-results-dir>/<persistId>.<ext>`
   - 返回 { filepath, size, ext }
5. 使用 `blobSavedTo` 路径和人类可读消息构建结果对象

示例结果：
```
{
  uri: c.uri,
  mimeType: c.mimeType,
  blobSavedTo: persisted.filepath,
  text: "Binary content (application/pdf, 1.2 MB) saved to /path/to/file.pdf"
}
```

为什么要持久化到磁盘：
- 上下文中的 Base64 blob 浪费令牌
- 本机工具可以按扩展名打开文件（PDF 阅读器等）
- Read 工具按扩展名分派以进行适当处理

### MIME 类型到扩展名映射

`extensionForMimeType` 函数将 MIME 类型映射到文件扩展名：

| MIME 类型模式 | 扩展名 |
|-------------------|--------|
| `application/pdf` | pdf |
| `application/json` | json |
| `text/csv` | csv |
| `text/plain` | txt |
| `text/html` | html |
| `text/markdown` | md |
| `application/zip` | zip |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | docx |
| `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | xlsx |
| `application/vnd.openxmlformats-officedocument.presentationml.presentation` | pptx |
| `application/msword` | doc |
| `application/vnd.ms-excel` | xls |
| `audio/mpeg`、`audio/wav`、`audio/ogg` | mp3、wav、ogg |
| `video/mp4`、`video/webm` | mp4、webm |
| `image/png`、`image/jpeg`、`image/gif`、`image/webp` | png、jpg、gif、webp |
| `image/svg+xml` | svg |
| 其他任何 | bin |

扩展名很重要，因为：
- Read 工具按文件扩展名分派
- PDF、图像等需要正确的扩展名才能正确处理
- 未知类型获得 'bin' 作为安全回退

### 二进制内容检测

`isBinaryContentType` 函数对内容类型进行分类：

视为 TEXT（不是二进制）：
- `text/*`（所有文本类型）
- `application/json` 和 `*/*+json` 变体
- `application/xml` 和 `*/*+xml` 变体
- `application/javascript`
- `application/x-www-form-urlencoded`

视为 BINARY（持久化到磁盘）：
- `application/pdf`、`application/zip`、`application/octet-stream`
- `image/*`、`audio/*`、`video/*`
- `application/vnd.*`（Office 格式）
- 与上述文本模式不匹配的任何内容

## 错误处理

### 请求前验证

| 错误条件 | 错误消息 |
|----------------|---------------|
| 服务器未找到 | `Server "<name>" not found. Available servers: <list>` |
| 服务器未连接 | `Server "<name>" is not connected` |
| 服务器缺乏资源能力 | `Server "<name>" does not support resources` |

### 二进制持久化错误

如果 `persistBinaryContent` 失败，错误会被优雅处理：

```
{
  uri: c.uri,
  mimeType: c.mimeType,
  text: "Binary content could not be saved to disk: <error message>"
}
```

错误被记录但不会导致整个工具调用失败。同一响应中的其他内容块仍会正常处理。

### 结果截断

`isResultTruncated` 函数：
- 通过 `jsonStringify` 序列化输出
- 通过 `isOutputLineTruncated` 检查输出行是否被截断
- 如果结果被截断则返回 true
- `maxResultSizeChars`：100,000（超过此限制的结果可能被截断）

## UI 渲染

### 工具使用消息

```
renderToolUseMessage(input)：
  - 缺少 uri 或 server -> null
  - 完整：'Read resource "<uri>" from server "<server>"'
```

### 用户 facing 名称

```
userFacingName()：'readMcpResource'
```

### 工具结果消息

```
renderToolResultMessage(output, _, { verbose })：
  - 空或无内容：在暗淡颜色中显示 '(No content)'
  - 非空：通过 OutputLine 显示 jsonStringify(output, null, 2)
    - 带 2 空格缩进的漂亮打印 JSON
    - OutputLine 组件处理截断
```

## 配置

### 工具属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `name` | `'ReadMcpResourceTool'` | 内部工具名称 |
| `userFacingName` | `'readMcpResource'` | UI 中的显示名称 |
| `isConcurrencySafe` | `true` | 只读，安全并发调用 |
| `isReadOnly` | `true` | 不修改服务器状态 |
| `shouldDefer` | `true` | 可以在 auto 模式中延迟 |
| `maxResultSizeChars` | `100,000` | 截断前的最大结果大小 |
| `searchHint` | `'read a specific MCP resource by URI'` | 自动模式搜索提示 |

### 持久化 ID 格式

```
mcp-resource-<timestamp>-<index>-<random>

示例：mcp-resource-1712345678901-0-a3f7b2

组件：
- timestamp：Date.now() 用于唯一性和时间排序
- index：响应数组中的内容块索引
- random：6 个字符的随机字符串（Math.random().toString(36).slice(2, 8)）
```

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `services/mcp/client.ts` | 客户端连接管理、ensureConnectedClient |
| `utils/mcpOutputStorage.ts` | 二进制内容持久化、MIME 映射 |
| `utils/slowOperations.ts` | JSON 字符串化 |
| `utils/terminal.ts` | 输出截断检测 |
| `utils/lazySchema.ts` | 延迟 Zod schema 评估 |
| `utils/errors.ts` | 错误实用工具 |
| `components/MessageResponse.js` | UI 消息组件 |
| `components/shell/OutputLine.js` | 终端输出渲染 |
| `ink.js` | Ink 终端 UI 原语 |

### 外部

| 包 | 用途 |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP 类型、ReadResourceResultSchema |
| `zod/v4` | 输入/输出 schema 验证 |
| `react` | UI 渲染 |
| `fs/promises` | 文件系统操作（writeFile） |
| `path` | 用于文件路径的路径连接 |

## 数据流

```
Model Request
    │
    ▼
ReadMcpResourceTool Input { server: string, uri: string }
    │
    ▼
+-----------------------------------------------------------------+
| RESOURCE READING                                                 |
|                                                                 |
|  1. Server validation                                           |
|     - Find client by server name                                |
|     - Check connected status                                    |
|     - Check resources capability                                |
|                                                                 |
|  2. Connection assurance                                        |
|     - ensureConnectedClient()                                   |
|     - Reconnect if needed                                       |
|                                                                 |
|  3. Resource request                                            |
|     - client.request({ method: 'resources/read', params })      |
|     - Schema: ReadResourceResultSchema                          |
|                                                                 |
|  4. Content processing (Promise.all)                            |
|     For each content block:                                     |
|     a. Text content -> pass through                             |
|     b. Binary content ->                                        |
|        - Decode base64                                          |
|        - Persist to disk with MIME-derived extension            |
|        - Replace with filepath + saved message                  |
|     c. Unknown content -> pass through with uri + mimeType      |
|                                                                 |
|  5. Return { data: { contents: processedArray } }               |
+-----------------------------------------------------------------+
    │
    ▼
Output { contents: [{ uri, mimeType?, text?, blobSavedTo? }, ...] }
    │
    ▼
+-----------------------------------------------------------------+
| UI RENDERING                                                    |
|                                                                 |
|  renderToolResultMessage:                                       |
|  - Empty: '(No content)'                                        |
|  - Non-empty: Pretty-printed JSON (2-space indent)              |
|                                                                 |
|  Binary content shows:                                          |
|  "Binary content (application/pdf, 1.2 MB) saved to /path/..."  |
+-----------------------------------------------------------------+
    │
    ▼
Model Response (tool_result block with resource content)
```
