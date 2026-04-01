# SyntheticOutputTool

## 用途

SyntheticOutputTool（也称为 `StructuredOutput`）使 Claude 能够以符合用户提供的 JSON schema 的结构化 JSON 形式返回最终响应。它专门用于非交互式会话（SDK/CLI 工作流），其中调用代码需要机器可解析的输出而不是自由格式文本。该工具在每次调用时动态创建，具有自定义 JSON schema，使用 Ajv 验证，并按 schema 身份缓存以避免重复编译开销。

## 位置

- `restored-src/src/tools/SyntheticOutputTool/SyntheticOutputTool.ts` — 工具定义、schema 验证和工厂（164 行）

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `SyntheticOutputTool` | 基础工具定义（无自定义 schema） |
| `SYNTHETIC_OUTPUT_TOOL_NAME` | 常量：`'StructuredOutput'` |
| `createSyntheticOutputTool(jsonSchema)` | 工厂函数：创建配置有给定 JSON schema 的工具实例 |
| `isSyntheticOutputToolEnabled(opts)` | 检查工具是否应启用（需要 `isNonInteractiveSession`） |
| `Output` | 类型：`string`（成功消息） |

## 输入/输出 Schema

### 输入 Schema

基础工具接受任何对象（`z.object({}).passthrough()`）。当通过 `createSyntheticOutputTool()` 创建时，实际输入 schema 是用户提供的 JSON schema，在创建时由 Ajv 验证并在运行时强制执行。

### 输出 Schema

```typescript
string  // "Structured output provided successfully"
```

实际结构化数据在调用结果的 `structured_output` 字段中返回（不作为 Zod 输出 schema 的一部分）。

## 工具创建

### 工厂函数

```typescript
createSyntheticOutputTool(jsonSchema: Record<string, unknown>): CreateResult
```

返回：
- `{ tool: Tool<InputSchema> }` — 成功创建的工具实例
- `{ error: string }` — 无效 schema 的 Ajv 验证错误消息

### 创建管道

```
1. 缓存检查
   - 在 WeakMap 缓存中查找 jsonSchema（按对象身份）
   - 如果已缓存 → 返回缓存结果

2. SCHEMA 验证（Ajv）
   - 使用 allErrors: true 创建 Ajv 实例
   - 使用 ajv.validateSchema() 验证 schema
   - 如果无效 → 返回 { error: ajv.errorsText() }

3. SCHEMA 编译
   - 使用 ajv.compile() 编译 schema
   - 返回验证函数

4. 工具构建
   - 克隆基础 SyntheticOutputTool
   - 用用户 schema 覆盖 inputJSONSchema
   - 覆盖 call() 以根据编译的 schema 验证输入
   - schema 不匹配时抛出 TelemetrySafeError

5. 缓存存储
   - 将结果存储在 WeakMap 中（按 schema 身份键控）
   - 返回 { tool }
```

### Schema 缓存

`WeakMap<object, CreateResult>` 按 schema 对象身份缓存工具实例：

```typescript
const toolCache = new WeakMap<object, CreateResult>()
```

这对于性能至关重要，因为工作流脚本在每次运行中多次调用 `agent({schema: SCHEMA})`（30-80 次），使用相同的 schema 对象引用。如果没有缓存，每次调用都会执行 `new Ajv() + validateSchema() + compile()`（约 1.4ms 的 JIT 代码生成）。身份缓存将 80 次调用工作流的 Ajv 开销从约 110ms 减少到约 4ms。

## 运行时验证

当模型调用工具时，输入根据编译的 schema 进行验证：

```typescript
async call(input) {
  const isValid = validateSchema(input)
  if (!isValid) {
    const errors = validateSchema.errors
      ?.map(e => `${e.instancePath || 'root'}: ${e.message}`)
      .join(', ')
    throw new TelemetrySafeError(
      `Output does not match required schema: ${errors}`,
      `StructuredOutput schema mismatch: ${(errors ?? '').slice(0, 150)}`,
    )
  }
  return {
    data: 'Structured output provided successfully',
    structured_output: input,
  }
}
```

错误消息包括：
- 实例路径（例如 `/items/0/name`）以精确定位
- Ajv 错误消息（例如 "must be string"）
- 遥测安全版本（错误截断至 150 字符，无代码/文件路径）

## 工具启用

仅当 `isSyntheticOutputToolEnabled()` 返回 true 时才创建工具：

```typescript
isSyntheticOutputToolEnabled(opts: { isNonInteractiveSession: boolean }): boolean
  → opts.isNonInteractiveSession
```

一旦创建，工具始终启用（`isEnabled() → true`）。它在创建时门控，而不是每次调用。

## 权限处理

| 操作 | 权限决策 |
|-----------|-------------------|
| 所有操作 | 自动允许（始终允许） |

该工具被认为是安全的，因为它仅验证和返回数据——没有副作用。

## UI 渲染

由于该工具用于非交互式使用，实现最小化 UI：

| 方法 | 输出 |
|--------|--------|
| `renderToolUseMessage(input)` | 显示最多 3 个字段键值对，或 `"N fields: key1, key2, key3…"` |
| `renderToolUseRejectedMessage()` | `"Structured output rejected"` |
| `renderToolUseErrorMessage()` | `"Structured output error"` |
| `renderToolUseProgressMessage()` | `null` |
| `renderToolResultMessage(output)` | 直接返回输出字符串 |

## 工具结果格式化

`mapToolResultToToolResultBlockParam()` 返回标准 tool_result 块：

```json
{
  "tool_use_id": "<id>",
  "type": "tool_result",
  "content": "Structured output provided successfully"
}
```

实际结构化数据在调用结果的 `structured_output` 字段中可用，SDK 单独提取。

## 提示

工具提示指示模型：

```
Use this tool to return your final response in the requested structured format.
You MUST call this tool exactly once at the end of your response to provide the structured output.
```

## 配置

### 限制

| 限制 | 值 | 描述 |
|-------|-------|-------------|
| `maxResultSizeChars` | 100,000 | 存储在上下文中的最大结果大小 |

### 工具属性

| 属性 | 值 | 描述 |
|----------|-------|-------------|
| `isMcp` | false | 不是 MCP 工具 |
| `isReadOnly` | true | 不修改状态 |
| `isConcurrencySafe` | true | 可安全并发调用 |
| `isOpenWorld` | false | 封闭世界工具 |
| `shouldDefer` | 未设置 | 在启动时加载 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `Tool.js` | 工具构建和类型 |
| `utils/errors.js` | 遥测安全错误类 |
| `utils/slowOperations.js` | JSON 字符串化 |
| `utils/lazySchema.js` | 延迟 schema 加载 |

### 外部

| 包 | 用途 |
|---------|---------|
| `ajv` | JSON Schema 验证 |
| `zod/v4` | 基础输入/输出 schema |

## 数据流

```
Model Request
    |
    v
SyntheticOutputTool Input { ...user schema fields... }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  - Auto-allow (always allowed)                                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ SCHEMA VALIDATION                                                │
│                                                                 │
│  1. 根据编译的 Ajv schema 验证输入                                │
│  2. 如果无效：                                                   │
│     - 收集带有实例路径的错误消息                                  │
│     - 抛出 TelemetrySafeError                                    │
│  3. 如果有效：                                                   │
│     - 返回 { data: 成功消息, structured_output: input }          │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { data: string, structured_output: object }
    |
    v
SDK extracts structured_output for programmatic use
```
