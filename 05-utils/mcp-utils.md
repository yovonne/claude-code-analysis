# MCP Utilities

## Purpose

The MCP utilities module provides schema validation and natural language date/time parsing for Model Context Protocol (MCP) elicitation inputs. It converts MCP schema definitions into Zod validators, validates user input against those schemas, and uses an LLM (Haiku) to parse natural language date/time strings into ISO 8601 format.

## Location

- `restored-src/src/utils/mcp/elicitationValidation.ts` — Schema validation for MCP elicitation inputs
- `restored-src/src/utils/mcp/dateTimeParser.ts` — Natural language date/time parsing via Haiku

## Key Exports

### Elicitation Validation (`elicitationValidation.ts`)

#### Types
- `ValidationResult`: Result of validating an elicitation input, containing `value`, `isValid`, and optional `error`

#### Functions
- `isEnumSchema(schema)`: Type guard for single-select enum schemas (legacy `enum` or new `oneOf` format)
- `isMultiSelectEnumSchema(schema)`: Type guard for multi-select enum schemas (`type: "array"` with `items.enum` or `items.anyOf`)
- `getMultiSelectValues(schema)`: Extracts values from a multi-select enum schema
- `getMultiSelectLabels(schema)`: Extracts display labels from a multi-select enum schema
- `getMultiSelectLabel(schema, value)`: Gets the label for a specific value in a multi-select enum
- `getEnumValues(schema)`: Extracts enum values from an EnumSchema (handles both `enum` and `oneOf` formats)
- `getEnumLabels(schema)`: Extracts display labels from an EnumSchema
- `getEnumLabel(schema, value)`: Gets the label for a specific enum value
- `validateElicitationInput(stringValue, schema)`: Synchronously validates user input against an MCP schema using Zod. Handles string formats (email, uri, date, date-time), numbers/integers with ranges, booleans, and enums
- `validateElicitationInputAsync(stringValue, schema, signal)`: Async validation that attempts natural language date/time parsing via Haiku when sync validation fails for date/date-time schemas
- `getFormatHint(schema)`: Returns a helpful placeholder/hint for a given schema format
- `isDateTimeSchema(schema)`: Type guard for date or date-time format schemas that support NL parsing

### Date/Time Parser (`dateTimeParser.ts`)

#### Types
- `DateTimeParseResult`: Union type `{ success: true; value: string } | { success: false; error: string }`

#### Functions
- `parseNaturalLanguageDateTime(input, format, signal)`: Parses natural language date/time input into ISO 8601 format using the Haiku LLM. Provides rich context (current datetime, timezone, day of week) in the prompt. Examples: "tomorrow at 3pm" → ISO 8601, "next Monday" → YYYY-MM-DD
- `looksLikeISO8601(input)`: Checks if a string looks like an ISO 8601 date/time (starts with YYYY-MM-DD). Used to decide whether to attempt NL parsing

## Dependencies

- `@modelcontextprotocol/sdk/types.js` — MCP schema type definitions
- `zod/v4` — Schema validation
- `../../services/api/claude.js` — Haiku LLM querying
- `../log.js`, `../messages.js`, `../systemPromptType.js`, `../slowOperations.js`, `../stringUtils.js` — Shared utilities

## Design Notes

- The validation system supports both legacy MCP enum format (`enum` array) and the newer `oneOf`/`anyOf` format, ensuring backward compatibility.
- Date/time parsing uses Haiku (a lightweight LLM) with carefully crafted prompts that include timezone context, making it robust for natural language inputs like "tomorrow at 3pm" or "next Monday".
- The async validation falls back to sync results when NL parsing fails, ensuring graceful degradation.
- String format hints (email, uri, date, date-time) provide user-friendly placeholders during elicitation.
