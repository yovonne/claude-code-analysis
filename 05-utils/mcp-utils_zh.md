# MCP 工具

## 目的

MCP 工具模块为 Model Context Protocol（MCP）引出输入提供模式验证和自然语言日期/时间解析。它将 MCP 模式定义转换为 Zod 验证器，根据这些模式验证用户输入，并使用 LLM（Haiku）将自然语言日期/时间字符串解析为 ISO 8601 格式。

## 位置

- `restored-src/src/utils/mcp/elicitationValidation.ts` — MCP 引出输入的模式验证
- `restored-src/src/utils/mcp/dateTimeParser.ts` — 通过 Haiku 进行自然语言日期/时间解析

## 主要导出

### 引出验证（`elicitationValidation.ts`）

#### 类型
- `ValidationResult`: 验证引出输入的结果，包含 `value`、`isValid` 和可选的 `error`

#### 函数
- `isEnumSchema(schema)`: 单选枚举模式的类型守卫（遗留 `enum` 或新 `oneOf` 格式）
- `isMultiSelectEnumSchema(schema)`: 多选枚举模式的类型守卫（`type: "array"` 带 `items.enum` 或 `items.anyOf`）
- `getMultiSelectValues(schema)`: 从多选枚举模式提取值
- `getMultiSelectLabels(schema)`: 从多选枚举模式提取显示标签
- `getMultiSelectLabel(schema, value)`: 获取多选枚举中特定值的标签
- `getEnumValues(schema)`: 从 EnumSchema 提取枚举值（处理 `enum` 和 `oneOf` 格式）
- `getEnumLabels(schema)`: 从 EnumSchema 提取显示标签
- `getEnumLabel(schema, value)`: 获取特定枚举值的标签
- `validateElicitationInput(stringValue, schema)`: 使用 Zod 同步验证用户输入是否符合 MCP 模式。处理字符串格式（email、uri、date、date-time）、带范围的数字/整数、布尔值和枚举
- `validateElicitationInputAsync(stringValue, schema, signal)`: 异步验证，在日期/日期时间模式的同步验证失败时尝试通过 Haiku 进行自然语言日期/时间解析
- `getFormatHint(schema)`: 返回给定模式格式的有用占位符/提示
- `isDateTimeSchema(schema)`: 支持 NL 解析的日期或日期时间格式模式的类型守卫

### 日期/时间解析器（`dateTimeParser.ts`）

#### 类型
- `DateTimeParseResult`: 联合类型 `{ success: true; value: string } | { success: false; error: string }`

#### 函数
- `parseNaturalLanguageDateTime(input, format, signal)`: 使用 Haiku LLM 将自然语言日期/时间输入解析为 ISO 8601 格式。在提示中提供丰富的上下文（当前日期时间、时区、星期几）。示例："tomorrow at 3pm" → ISO 8601，"next Monday" → YYYY-MM-DD
- `looksLikeISO8601(input)`: 检查字符串是否看起来像 ISO 8601 日期/时间（以 YYYY-MM-DD 开头）。用于决定是否尝试 NL 解析

## 依赖

- `@modelcontextprotocol/sdk/types.js` — MCP 模式类型定义
- `zod/v4` — 模式验证
- `../../services/api/claude.js` — Haiku LLM 查询
- `../log.js`、`../messages.js`、`../systemPromptType.js`、`../slowOperations.js`、`../stringUtils.js` — 共享工具

## 设计注意事项

- 验证系统支持遗留 MCP 枚举格式（`enum` 数组）和更新的 `oneOf`/`anyOf` 格式，确保向后兼容。
- 日期/时间解析使用 Haiku（轻量级 LLM），配合精心设计的提示包含时区上下文，使其对"tomorrow at 3pm"或"next Monday"等自然语言输入具有鲁棒性。
- 异步验证在 NL 解析失败时回退到同步结果，确保优雅降级。
- 字符串格式提示（email、uri、date、date-time）在引出期间提供用户友好的占位符。
