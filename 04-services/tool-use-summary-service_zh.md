# 工具使用摘要服务

## 目的

使用 Haiku（Claude 的快速模型）生成已完成工具批次的类人摘要。由 SDK 用于向客户端提供高级进度更新，特别是移动应用显示，其中工具执行细节需要压缩成短标签。

## 位置

`restored-src/src/services/toolUseSummary/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `toolUseSummaryGenerator.ts` | 使用 Haiku API 生成摘要 |

## 关键导出

### 函数

- `generateToolUseSummary(params)`：从工具执行数据生成简短的摘要字符串

### 类型

- `GenerateToolUseSummaryParams`：输入参数
  - `tools`：每个执行工具的 `{ name, input, output }` 数组
  - `signal`：用于取消的 AbortSignal
  - `isNonInteractiveSession`：这是否是非交互式（SDK/程序化）会话
  - `lastAssistantText`：可选的最后一个助手消息文本用于上下文
- `ToolInfo`：`{ name: string, input: unknown, output: unknown }`

## 依赖

### 内部依赖
- `api/claude` — 用于摘要生成的 queryHaiku
- `errors` — 用于错误转换的 toError
- `log` — 用于错误日志的 logError
- `slowOperations` — 用于序列化的 jsonStringify
- `systemPromptType` — 用于提示格式化的 asSystemPrompt
- `constants/errorIds` — 用于跟踪的错误 ID

### 外部依赖
- 无（使用内部 API 客户端）

## 实现细节

### 摘要生成

1. 验证输入 — 如果没有工具则返回 null
2. 构建简洁的工具摘要：
   - 将输入和输出序列化为 JSON（每个截断到 300 字符）
   - 格式化为 `Tool: {name}\nInput: {input}\nOutput: {output}`
3. 可选地前缀来自最后一个助手消息的用户意图（截断到 200 字符）
4. 使用以下参数调用 Haiku API：
   - 系统提示：简短过去时摘要标签的指令
   - 用户提示：带可选上下文的工具摘要
   - 启用提示缓存
   - 无 MCP 工具，无代理，无追加系统提示
5. 从响应中提取文本内容，修剪空白
6. 返回摘要字符串或如果为空/失败则返回 null

### 系统提示设计

系统提示指示模型：
- 写一个简短的摘要标签（git-commit-subject 风格）
- 截断到约 30 个字符（用于移动显示）
- 使用过去时动词 + 最具特色的名词
- 删除冠词、连接词和长位置上下文
- 示例："Searched in auth/"、"Fixed NPE in UserService"、"Created signup endpoint"

### 输入截断

- 工具输入/输出：每个 300 字符
- 最后一个助手文本：200 字符
- 序列化失败：返回 `[unable to serialize]`

## 数据流

```
工具批次完成
    ↓
generateToolUseSummary({ tools, signal, isNonInteractiveSession, lastAssistantText })
    ↓
截断工具输入/输出（每个 300 字符）
    ↓
构建工具摘要字符串
    ↓
可选地预置用户意图（200 字符）
    ↓
queryHaiku() — 快速模型，缓存提示
    ↓
提取文本内容，修剪
    ↓
返回摘要字符串或 null
```

## 集成点

- **SDK**：由 SDK 调用以生成客户端应用的进度更新
- **移动应用**：摘要显示为单行（约 30 字符截断）
- **非交互式会话**：`isNonInteractiveSession` 标志传递到 API

## 配置

- 模型：Haiku（快速、经济，用于摘要生成）
- 提示缓存：启用（系统提示是静态的）
- 无工具访问：摘要生成不需要 MCP 工具或代理

## 错误处理

- **非关键**：摘要是尽力而为 — 失败记录但不阻止执行
- **错误跟踪**：使用 `E_TOOL_USE_SUMMARY_GENERATION_FAILED` 错误 ID
- **空输入**：对空工具数组返回 null
- **序列化失败**：优雅处理不可序列化的值
- **API 失败**：捕获错误，记录，返回 null

## 测试

- 无测试专用导出 — 纯函数，使用模拟 queryHaiku 易于测试

## 相关模块

- [分析](./analytics-service.md) — API 查询遥测
- [查询引擎](../01-core-modules/query-engine.md) — LLM 查询基础设施

## 备注

- 专为移动应用显示设计 — 摘要必须简短且信息丰富
- 专门使用 Haiku 以提高速度和成本（摘要是非关键的）
- 提示缓存降低成本，因为系统提示是常量
- 摘要格式镜像 git 提交主题：过去时动词 + 特色名词
- 最后一个助手文本提供用户意图上下文但是可选的
