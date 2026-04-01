# GlobTool

## 用途

`GlobTool`（对用户显示为 **Search** 工具）是一个快速的 파일 模式匹配实用工具，可以在任何代码库大小中通过名称模式或通配符查找文件。它支持标准 glob 模式如 `**/*.js` 或 `src/**/*.ts`，返回按修改时间排序的匹配文件路径，并与权限系统集成以实现安全的文件系统访问。结果在可配置的限制（默认 100）处截断，以防止上下文膨胀。

## 位置

`restored-src/src/tools/GlobTool/GlobTool.ts`

支持文件：
- `restored-src/src/tools/GlobTool/prompt.ts` — 工具名称常量和描述文本
- `restored-src/src/tools/GlobTool/UI.tsx` — 用于使用/结果/错误消息的终端 UI 渲染

## 主要导出

### 函数

- `userFacingName()`：返回 `'Search'` — 在 UI 中显示的人类可读工具名称
- `renderToolUseMessage(input, { verbose })`：渲染显示模式和可选路径的工具调用显示
- `renderToolUseErrorMessage(result, { verbose })`：为文件未找到和常规搜索错误渲染带有用户友好文本的错误消息
- `renderToolResultMessage(output, progressMessages, { verbose })`：渲染搜索结果（重用 `GrepTool` 的实现）
- `getToolUseSummary(input)`：返回用于活动显示的 glob 模式的截断摘要

### 类型

- `Input`：Zod 推断的输入 schema — `{ pattern: string, path?: string }`
- `Output`：`{ durationMs: number, numFiles: number, filenames: string[], truncated: boolean }`

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `GLOB_TOOL_NAME` | `'Glob'` | 内部工具标识符 |
| `DESCRIPTION` | 多行字符串 | 工具功能和使用指南 |
| `maxResultSizeChars` | `100_000` | 持久化的最大结果大小 |

## 依赖

### 内部依赖

- `glob`（`src/utils/glob.ts`）— 核心 glob 模式匹配引擎
- `expandPath`、`toRelativePath`（`src/utils/path.ts`）— 路径规范化和相对化
- `checkReadPermissionForTool`（`src/utils/permissions/filesystem.ts`）— 权限评估
- `matchWildcardPattern`（`src/utils/permissions/shellRuleMatching.ts`）— 用于权限规则通配符匹配
- `getCwd`（`src/utils/cwd.ts`）— 当前工作目录解析
- `getFsImplementation`（`src/utils/fsOperations.ts`）— 抽象的文件系统操作
- `suggestPathUnderCwd`、`FILE_NOT_FOUND_CWD_NOTE`（`src/utils/file.ts`）— 错误时的路径建议
- `isENOENT`（`src/utils/errors.ts`）— ENOENT 错误检测
- `lazySchema`（`src/utils/lazySchema.ts`）— 延迟 Zod schema 评估
- `GrepTool`（`src/tools/GrepTool/GrepTool.ts`）— 重用于结果消息渲染

### 外部依赖

- `zod/v4` — 输入/输出 schema 验证
- `react` — UI 组件渲染（基于 Ink 的终端 UI）

## 实现细节

### 核心逻辑

该工具使用 `buildTool()` 构建，遵循简化的执行管道：

1. **输入验证**（`validateInput`）— 基于 I/O 的检查，可选 `path` 参数是否存在以及是否为目录。为防止 NTLM 凭证泄漏，对 UNC 路径跳过文件系统操作。
2. **权限检查**（`checkPermissions`）— 通过 `checkReadPermissionForTool()` 评估读取权限。
3. **权限匹配器准备**（`preparePermissionMatcher`）— 使用通配符模式匹配进行规则评估。
4. **执行**（`call`）— 使用模式、解析路径和结果限制委托给 `glob()` 实用工具。
5. **结果相对化** — 将所有绝对路径转换为 `cwd` 下的相对路径以节省令牌。

`call()` 方法是核心执行：

```typescript
const { files, truncated } = await glob(
  input.pattern,
  GlobTool.getPath(input),  // 解析的路径或 cwd
  { limit, offset: 0 },     // 分页参数
  abortController.signal,    // 取消支持
  appState.toolPermissionContext,
)
```

### 输入 Schema

```typescript
z.strictObject({
  pattern: z.string(),       // 匹配文件的 glob 模式
  path: z.string().optional(), // 搜索目录（默认为 cwd）
})
```

### 输出 Schema

```typescript
z.object({
  durationMs: z.number(),          // 耗时（毫秒）
  numFiles: z.number(),            // 找到的文件总数
  filenames: z.array(z.string()),  // 匹配文件路径数组（相对）
  truncated: z.boolean(),          // 结果是否被截断（限制为 100）
})
```

### 关键算法

#### Glob 模式匹配

委托给 `src/utils/glob.ts` 中的 `glob()` 实用工具。该实用工具处理：
- 标准 glob 语法：`*`（除 `/` 外的任何字符）、`**`（包括 `/` 的任何字符）、`?`（单个字符）、`[abc]`（字符类）、`{a,b}`（交替）
- 重复使用的模式编译和缓存
- 达到限制时提前终止的递归目录遍历

#### 文件发现和路径解析

路径解析遵循明确的优先级：

1. 如果提供了 `path`，则通过 `expandPath()` 扩展（波浪号扩展、Windows 分隔符规范化）
2. 如果省略 `path`，则使用 `getCwd()` 作为搜索根
3. 所有返回的路径通过 `toRelativePath()` 转换为相对路径以减少令牌使用

#### 结果截断

结果限制为可配置的最大值（默认 100 个文件），由 `globLimits.maxResults` 控制。输出中的 `truncated` 布尔值指示是否返回了比存在更多的匹配，提示用户优化其模式或路径。

#### 输入验证

当提供 `path` 时，验证执行：
1. **UNC 路径检测**：以 `\\` 或 `//` 开头的路径跳过 stat 检查以防止 NTLM 哈希泄漏
2. **存在性检查**：`fs.stat()` 验证路径存在
3. **目录检查**：确认路径是目录，不是文件
4. **失败时的建议**：如果路径不存在，`suggestPathUnderCwd()` 提供当前工作目录下类似路径的建议

### 边缘情况

- **无结果**：返回 `"No files found"` 作为工具结果内容
- **截断结果**：附加 `"(Results are truncated. Consider using a more specific path or pattern.)"` 来引导用户
- **UNC 路径**：验证跳过 `\\server\share` 或 `//server/share` 路径的文件系统 stat 调用以防止 NTLM 哈希泄漏
- **无效路径类型**：如果路径存在但不是目录则返回错误代码 2
- **不存在的路径**：返回错误代码 1，并提供 cwd 下类似路径的建议

## 数据流

```
User Request
    │
    ▼
┌─────────────────────┐
│  validateInput()    │  ← 路径存在？是否目录？UNC 跳过？
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  checkPermissions() │  ← 读取权限评估
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  call()             │  ← 委托给 glob() 实用工具
│  glob(pattern,      │
│    path,            │
│    {limit, offset}, │
│    abortSignal,      │
│    permContext)     │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Relativize Paths   │  ← 对每个匹配应用 toRelativePath()
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  Result Mapping     │  ← mapToolResultToToolResultBlockParam
│  + Truncation Note  │
└─────────┴───────────┘
          │
          ▼
    Tool Result
```

## 集成点

### GrepTool UI 重用

`GlobTool` 重用 `GrepTool.renderToolResultMessage` 来渲染搜索结果。两个工具都生成文件名列表，因此共享组件显示 `"Found N files"` 摘要，可通过 `Ctrl+O` 展开内容。

### 权限系统

- `preparePermissionMatcher()` 创建一个匹配器函数，使用 `matchWildcardPattern()` 将权限规则模式与 glob 模式进行比较
- `checkPermissions()` 委托给 `checkReadPermissionForTool()`，评估工具与配置的读取权限规则
- 工具标记为 `isReadOnly: true`，强化其非变更性质

### 自动分类器

`toAutoClassifierInput()` 返回 `input.pattern`，使自动分类器能够按搜索的 glob 模式对工具调用进行分类。

### 搜索/读取分类

`isSearchOrReadCommand()` 返回 `{ isSearch: true, isRead: false }`，将其分类为搜索操作（不是读取操作）以进行活动跟踪和分析。

### 并发安全

`isConcurrencySafe()` 返回 `true`，允许与其他工具并行执行。Glob 操作是只读的且无状态的。

## 配置

### 结果限制

| 参数 | 默认 | 描述 |
|---|---|---|
| `globLimits.maxResults` | `100` | 每个 glob 调用返回的最大文件数 |
| `maxResultSizeChars` | `100,000` | 持久化工具结果的字符限制 |

### 环境变量

无特定工具的环境变量。限制通过运行时传递的 `globLimits` 上下文参数控制。

## 错误处理

### 验证错误（I/O 前）

| 错误代码 | 条件 | 消息 |
|---|---|---|
| 1 | 目录不存在 | "Directory does not exist: {path}. … Did you mean {suggestion}?" |
| 2 | 路径不是目录 | "Path is not a directory: {path}" |

### 运行时错误

| 错误 | 条件 | 处理 |
|---|---|---|
| `ENOENT` | 验证期间路径未找到 | 建议 cwd 下的类似路径 |
| Abort | `abortController.signal` 触发 | 将取消传播到 `glob()` 实用工具 |

### 错误 UI 渲染

`UI.tsx` 中的 `renderToolUseErrorMessage()` 提供用户友好的错误显示：
- 检测"file not found"模式（包含 `FILE_NOT_FOUND_CWD_NOTE`）并显示"File not found"
- 检测 `<tool_use_error>` 标签并显示"Error searching files"
- 回退到 `FallbackToolUseErrorMessage` 以处理其他错误

## 性能优化

### 源处结果限制

`glob()` 实用工具接受 `limit` 参数，一旦达到限制就停止遍历。这避免了在只需要前 N 个匹配时扫描整个文件系统。

### 路径相对化

所有返回的路径通过 `toRelativePath()` 从绝对路径转换为相对路径（相对于 `cwd`）。这减少了工具结果中的令牌使用，特别是对于深度嵌套的项目。

### 延迟 Schema 评估

输入和输出 schema 使用 `lazySchema()` 延迟 Zod schema 构建直到首次访问，减少冷启动开销。

### AbortController 支持

`abortController.signal` 传递到 `glob()` 实用工具，在用户导航离开或请求被替代时启用长时间搜索的取消。

### 并发安全

标记为并发安全（`isConcurrencySafe: true`），允许多个 glob 搜索并行运行而不会出现竞争条件。

## 测试

`GlobTool/` 目录中没有特定于工具的测试文件。测试可能由更广泛的工具测试套件中的集成测试覆盖。

## 相关模块

- [GrepTool](./GrepTool.md) — 互补的内容搜索工具；共享 UI 渲染
- [BashTool](./BashTool.md) — 用于目录列表和文件系统探索
- [AgentTool](./AgentTool.md) — 用于需要多轮的开放式搜索
- [glob utility](../05-utils/glob.md) — 核心 glob 模式匹配引擎

## 备注

- 对用户显示的工具名称是 `'Search'`（通过 `userFacingName()`），而内部标识符是 `'Glob'`
- 结果由底层 `glob()` 实用工具按修改时间排序
- 100 个文件的截断限制故意设置得很低，以防止上下文窗口膨胀——鼓励用户缩小模式或路径以获得更有针对性的结果
- 对于可能需要多轮 glob 和 grep 的开放性搜索，工具描述建议使用 `Agent` 工具
- 该工具重用 GrepTool 的 `renderToolResultMessage`，因为两者都生成具有相似显示要求的文件名列表
