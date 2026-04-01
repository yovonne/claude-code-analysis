# GrepTool

## 用途

`GrepTool`（对用户显示为 **Search** 工具）是一个基于 ripgrep 的强大内容搜索实用工具。它使用正则表达式在任意代码库大小中搜索文件内容，支持基于 glob 的文件过滤、上下文行、多种输出模式和分页。它自动排除版本控制目录，尊重 `.gitignore` 模式，并应用智能结果限制以防止上下文窗口膨胀。该工具是主要搜索机制——用户被指示始终使用它而不是通过 Bash 调用 `grep` 或 `rg`。

## 位置

`restored-src/src/tools/GrepTool/GrepTool.ts`

支持文件：
- `restored-src/src/tools/GrepTool/prompt.ts` — 工具名称常量和动态描述生成器
- `restored-src/src/tools/GrepTool/UI.tsx` — 带 `SearchResultSummary` 组件的终端 UI 渲染

## 主要导出

### 函数

- `getDescription()`：返回带有使用说明、正则表达式语法注释和 ripgrep 特定指导的完整工具描述
- `renderToolUseMessage(input, { verbose })`：渲染显示模式、路径和可选过滤器的工具调用显示
- `renderToolUseErrorMessage(result, { verbose })`：为文件未找到和常规搜索错误渲染带有用户友好文本的错误消息
- `renderToolResultMessage(output, progressMessages, { verbose })`：使用模式特定的摘要渲染搜索结果（内容行、匹配计数或文件列表）
- `getToolUseSummary(input)`：返回用于活动显示的搜索模式的截断摘要

### 类型

- `Input`：Zod 推断的输入 schema — `{ pattern: string, path?: string, glob?: string, output_mode?: 'content' | 'files_with_matches' | 'count', -B?: number, -A?: number, -C?: number, context?: number, -n?: boolean, -i?: boolean, type?: string, head_limit?: number, offset?: number, multiline?: boolean }`
- `Output`：`{ mode?: 'content' | 'files_with_matches' | 'count', numFiles: number, filenames: string[], content?: string, numLines?: number, numMatches?: number, appliedLimit?: number, appliedOffset?: number }`

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `GREP_TOOL_NAME` | `'Grep'` | 内部工具标识符 |
| `VCS_DIRECTORIES_TO_EXCLUDE` | `['.git', '.svn', '.hg', '.bzr', '.jj', '.sl']` | 自动排除的版本控制目录 |
| `DEFAULT_HEAD_LIMIT` | `250` | 未指定 `head_limit` 时的默认结果上限 |
| `maxResultSizeChars` | `20_000` | 持久化的最大结果大小（20K — 工具结果持久化阈值） |

## 依赖

### 内部依赖

- `ripGrep`（`src/utils/ripgrep.ts`）— 带超时处理的 Ripgrep 子进程包装器
- `expandPath`、`toRelativePath`（`src/utils/path.ts`）— 路径规范化和相对化
- `checkReadPermissionForTool`、`getFileReadIgnorePatterns`、`normalizePatternsToPath`（`src/utils/permissions/filesystem.ts`）— 权限评估和忽略模式提取
- `matchWildcardPattern`（`src/utils/permissions/shellRuleMatching.ts`）— 用于权限规则的通配符匹配
- `getGlobExclusionsForPluginCache`（`src/utils/plugins/orphanedPluginFilter.ts`）— 插件目录排除模式
- `semanticBoolean`、`semanticNumber`（`src/utils/semanticBoolean.ts`、`src/utils/semanticNumber.ts`）— LLM 友好数字/布尔处理的 Zod schema 包装器
- `plural`（`src/utils/stringUtils.ts`）— 结果摘要的复数化实用工具
- `getCwd`（`src/utils/cwd.ts`）— 当前工作目录解析
- `getFsImplementation`（`src/utils/fsOperations.ts`）— 抽象的文件系统操作（用于按 mtime 排序）
- `suggestPathUnderCwd`、`FILE_NOT_FOUND_CWD_NOTE`、`getDisplayPath`（`src/utils/file.ts`）— 路径建议和显示格式化
- `isENOENT`（`src/utils/errors.ts`）— ENOENT 错误检测
- `truncate`（`src/utils/format.ts`）— 摘要文本截断
- `extractTag`（`src/utils/messages.ts`）— 错误消息中的 XML 标签提取

### 外部依赖

- `zod/v4` — 输入/输出 schema 验证
- `react` — UI 组件渲染（基于 Ink 的终端 UI，带 React Compiler）
- `@anthropic-ai/sdk` — 工具结果块类型

## 实现细节

### 核心逻辑

该工具使用 `buildTool()` 构建，遵循多阶段执行管道：

1. **输入验证**（`validateInput`）— 基于 I/O 的检查，可选 `path` 参数是否存在。为防止 NTLM 凭证泄漏，对 UNC 路径跳过文件系统操作。
2. **权限检查**（`checkPermissions`）— 通过 `checkReadPermissionForTool()` 评估读取权限。
3. **权限匹配器准备**（`preparePermissionMatcher`）— 使用通配符模式匹配将权限规则与正则表达式模式进行比较。
4. **参数构建**（`call`）— 从输入参数构建 ripgrep 命令行参数。
5. **Ripgrep 执行** — 使用构造的参数和搜索路径调用 `ripGrep()`。
6. **结果处理** — 模式特定处理：相对化路径、应用头部限制、解析计数或按 mtime 排序。

### 输入 Schema

```typescript
z.strictObject({
  pattern: z.string(),              // 搜索的正则表达式模式
  path: z.string().optional(),      // 搜索的文件或目录（默认为 cwd）
  glob: z.string().optional(),      // 过滤文件的 glob 模式（例如 "*.js"、"*.{ts,tsx}"）
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  '-B': z.number().optional(),      // 匹配前的行数（rg -B）
  '-A': z.number().optional(),      // 匹配后的行数（rg -A）
  '-C': z.number().optional(),      // 上下文行（rg -C，context 的别名）
  context: z.number().optional(),   // 前后上下文行（rg -C）
  '-n': z.boolean().optional(),     // 显示行号（默认为 true）
  '-i': z.boolean().optional(),     // 不区分大小写搜索（rg -i）
  type: z.string().optional(),      // 文件类型过滤器（rg --type: js、py、rust、go 等）
  head_limit: z.number().optional(),// 限制输出到前 N 个结果（默认 250）
  offset: z.number().optional(),    // 在应用 head_limit 之前跳过前 N 个结果
  multiline: z.boolean().optional(),// 启用多行模式（rg -U --multiline-dotall）
})
```

### 输出 Schema（模式相关）

```typescript
z.object({
  mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  numFiles: z.number(),
  filenames: z.array(z.string()),
  content: z.string().optional(),       // 对于 content 和 count 模式
  numLines: z.number().optional(),      // 对于 content 模式
  numMatches: z.number().optional(),    // 对于 count 模式
  appliedLimit: z.number().optional(),  // 实际应用的限制
  appliedOffset: z.number().optional(), // 应用的偏移量
})
```

### 关键算法

#### 通过 Ripgrep 进行正则表达式搜索

该工具从输入参数构建 ripgrep 命令行参数：

```typescript
const args = ['--hidden']  // 默认包含隐藏文件

// 排除 VCS 目录
for (const dir of VCS_DIRECTORIES_TO_EXCLUDE) {
  args.push('--glob', `!${dir}`)
}

// 限制行长度以防止精简/ base64 混乱
args.push('--max-columns', '500')

// 多行模式（仅在明确请求时）
if (multiline) {
  args.push('-U', '--multiline-dotall')
}

// 不区分大小写
if (case_insensitive) {
  args.push('-i')
}

// 输出模式
if (output_mode === 'files_with_matches') {
  args.push('-l')  // 列出有匹配的文件
} else if (output_mode === 'count') {
  args.push('-c')  // 显示匹配计数
}

// 行号（仅用于 content 模式）
if (show_line_numbers && output_mode === 'content') {
  args.push('-n')
}

// 上下文行（-C/context 优先于 -B/-A）
if (output_mode === 'content') {
  if (context !== undefined) {
    args.push('-C', context.toString())
  } else if (context_c !== undefined) {
    args.push('-C', context_c.toString())
  } else {
    if (context_before !== undefined) args.push('-B', context_before.toString())
    if (context_after !== undefined) args.push('-A', context_after.toString())
  }
}

// 模式处理（如果模式以破折号开头，使用 -e 标志）
if (pattern.startsWith('-')) {
  args.push('-e', pattern)
} else {
  args.push(pattern)
}
```

通过 `ripGrep(args, absolutePath, abortController.signal)` 调用 ripgrep 子进程，处理执行、输出解析和超时管理。

#### 使用 Glob 模式进行文件过滤

`glob` 参数支持逗号分隔的模式和大括号扩展：

```typescript
// 在逗号和空格上拆分，但保留带大括号的模式
const globPatterns: string[] = []
const rawPatterns = glob.split(/\s+/)

for (const rawPattern of rawPatterns) {
  if (rawPattern.includes('{') && rawPattern.includes('}')) {
    globPatterns.push(rawPattern)  // 不拆分大括号模式
  } else {
    globPatterns.push(...rawPattern.split(',').filter(Boolean))
  }
}

for (const globPattern of globPatterns.filter(Boolean)) {
  args.push('--glob', globPattern)
}
```

这正确处理像 `*.{ts,tsx}`（保持原样）和 `*.js,*.jsx`（拆分为单独的 `--glob` 标志）这样的模式。

#### 忽略模式集成

该工具自动应用来自权限上下文的忽略模式：

```typescript
const ignorePatterns = normalizePatternsToPath(
  getFileReadIgnorePatterns(appState.toolPermissionContext),
  getCwd(),
)
for (const ignorePattern of ignorePatterns) {
  // 绝对路径：!/path/to/ignore
  // 相对路径：!**/path/to/ignore（用于 ripgrep 的相对匹配）
  const rgIgnorePattern = ignorePattern.startsWith('/')
    ? `!${ignorePattern}`
    : `!**/${ignorePattern}`
  args.push('--glob', rgIgnorePattern)
}
```

此外，孤立的插件版本目录通过 `getGlobExclusionsForPluginCache()` 排除。

#### 上下文行

上下文行处理支持三种具有明确优先级的规范方法：

1. `context` 或 `-C` — 对称上下文（前后 N 行）
2. `-B` 和 `-A` — 非对称上下文（单独的之前/之后计数）

当指定 `context` 或 `-C` 时，它优先于 `-B`/`-A`。`-C` 和 `context` 都映射到 ripgrep 的 `-C` 标志。

#### 输出模式

该工具支持三种不同的输出模式：

| 模式 | Ripgrep 标志 | 输出 | 分页 |
|---|---|---|---|
| `files_with_matches`（默认） | `-l` | 文件路径列表 | `head_limit` 限制文件计数 |
| `content` | 无（默认） | 带上下文的匹配行 | `head_limit` 限制输出行 |
| `count` | `-c` | 每个文件的匹配计数 | `head_limit` 限制计数条目 |

**Content 模式** 处理格式为 `/absolute/path:line_content` 或 `/absolute/path:num:content` 的 ripgrep 输出行，将绝对路径转换为相对路径：

```typescript
const finalLines = limitedResults.map(line => {
  const colonIndex = line.indexOf(':')
  if (colonIndex > 0) {
    const filePath = line.substring(0, colonIndex)
    const rest = line.substring(colonIndex)
    return toRelativePath(filePath) + rest
  }
  return line
})
```

**Count 模式** 解析 ripgrep 的 `filename:count` 格式并聚合总数：

```typescript
let totalMatches = 0
let fileCount = 0
for (const line of finalCountLines) {
  const colonIndex = line.lastIndexOf(':')
  if (colonIndex > 0) {
    const countStr = line.substring(colonIndex + 1)
    const count = parseInt(countStr, 10)
    if (!isNaN(count)) {
      totalMatches += count
      fileCount += 1
    }
  }
}
```

**Files with matches 模式**（默认）按修改时间排序结果：

```typescript
const stats = await Promise.allSettled(
  results.map(_ => getFsImplementation().stat(_)),
)
const sortedMatches = results
  .map((_, i) => {
    const r = stats[i]!
    return [_, r.status === 'fulfilled' ? (r.value.mtimeMs ?? 0) : 0] as const
  })
  .sort((a, b) => {
    if (process.env.NODE_ENV === 'test') {
      return a[0].localeCompare(b[0])  // 测试中确定性
    }
    const timeComparison = b[1] - a[1]
    if (timeComparison === 0) {
      return a[0].localeCompare(b[0])  // 文件名决胜
    }
    return timeComparison
  })
  .map(_ => _[0])
```

使用 `Promise.allSettled`，因此单个 `ENOENT`（文件在 ripgrep 扫描和此 stat 调用之间被删除）不会拒绝整个批次。失败的 stat 排序为 mtime 0。

#### 使用 Head Limit 和 Offset 进行分页

`applyHeadLimit()` 函数在所有输出模式中实现分页：

```typescript
function applyHeadLimit<T>(
  items: T[],
  limit: number | undefined,
  offset: number = 0,
): { items: T[]; appliedLimit: number | undefined } {
  // 显式 0 = 无限逃生出口
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT  // 250
  const sliced = items.slice(offset, offset + effectiveLimit)
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

关键设计决策：
- `limit === 0` 是无限出口（避免 falsy `0` 被视为"未指定限制"）
- `appliedLimit` 仅在实际发生截断时设置，因此模型知道可能有更多结果
- `offset` 启用分页：使用 `offset: 250, head_limit: 250` 获取结果 250-500

`formatLimitInfo()` 辅助函数生成分页元数据的显示文本：

```typescript
function formatLimitInfo(
  appliedLimit: number | undefined,
  appliedOffset: number | undefined,
): string {
  const parts: string[] = []
  if (appliedLimit !== undefined) parts.push(`limit: ${appliedLimit}`)
  if (appliedOffset) parts.push(`offset: ${appliedOffset}`)
  return parts.join(', ')
}
```

### 边缘情况

- **以破折号开头的模式**：使用 `-e` 标志防止 ripgrep 将其解释为命令行选项
- **无结果**：返回模式适当的空消息（"No files found"、"No matches found"）
- **扫描和 stat 之间删除的文件**：`Promise.allSettled` 优雅处理；失败的 stat 排序为 mtime 0
- **WSL 性能**：Ripgrep 超时在内部处理；`RipgrepTimeoutError` 向上传播，以便代理知道搜索未完成
- **精简/ base64 内容**：`--max-columns 500` 限制行长度以防止输出混乱
- **测试确定性**：在 `NODE_ENV === 'test'` 中，排序使用文件名而不是 mtime 以获得可重现的结果
- **多行模式**：仅在通过 `multiline: true` 明确请求时启用，以避免单行搜索的性能下降

## 数据流

```
User Request
    │
    ▼
┌─────────────────────┐
│  validateInput()    │  ← 路径存在？UNC 跳过？
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  checkPermissions() │  ← 读取权限评估
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  Build rg args      │  ← 来自输入的标志：模式、上下文、glob、类型等
│  --hidden            │
│  --glob !.git ...    │  ← VCS 排除
│  --max-columns 500   │  ← 行长度限制
│  -l/-c/content       │  ← 输出模式
│  -C/-B/-A            │  ← 上下文行
│  --glob (user)       │  ← 用户 glob 过滤器
│  --glob (ignore)     │  ← 权限忽略模式
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  ripGrep()          │  ← 带超时的子进程执行
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  Mode Processing    │
│  ┌───────────────┐  │
│  │ content:      │  │  应用 head_limit → 相对化路径 → 连接行
│  │ count:        │  │  应用 head_limit → 相对化 → 解析总数
│  │ files:        │  │  Stat → 按 mtime 排序 → 应用 head_limit → 相对化
│  └───────────────┘  │
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  Result Mapping     │  ← mapToolResultToToolResultBlockParam
│  + Pagination Info  │
└─────────┴───────────┘
          │
          ▼
    Tool Result
```

## 集成点

### GlobTool UI 共享

`GlobTool` 重用 `GrepTool.renderToolResultMessage` 来渲染文件列表结果。`UI.tsx` 中的共享 `SearchResultSummary` 组件处理两个工具的显示。

### 权限系统

- `preparePermissionMatcher()` 创建一个匹配器函数，使用 `matchWildcardPattern()` 将权限规则模式与正则表达式模式进行比较
- `checkPermissions()` 委托给 `checkReadPermissionForTool()`，评估工具与配置的读取权限规则
- 来自 `getFileReadIgnorePatterns()` 的忽略模式自动转换为 ripgrep `--glob` 排除
- 工具标记为 `isReadOnly: true`，强化其非变更性质

### 自动分类器

`toAutoClassifierInput()` 返回 `input.pattern` 或 `input.pattern in input.path`，使自动分类器能够按搜索模式和范围对工具调用进行分类。

### 搜索/读取分类

`isSearchOrReadCommand()` 返回 `{ isSearch: true, isRead: false }`，将其分类为搜索操作（不是读取操作）以进行活动跟踪和分析。

### 并发安全

`isConcurrencySafe()` 返回 `true`，允许与其他工具并行执行。Grep 操作是只读的且无状态的。

### 插件系统集成

`getGlobExclusionsForPluginCache()` 为孤立的插件版本目录提供排除模式，防止过时的插件工件出现在搜索结果中。

## 配置

### 结果限制

| 参数 | 默认 | 描述 |
|---|---|---|
| `DEFAULT_HEAD_LIMIT` | `250` | 未指定 `head_limit` 时的默认结果上限 |
| `maxResultSizeChars` | `20,000` | 持久化工具结果的字符限制 |
| `--max-columns` | `500` | ripgrep 输出中的最大行长度 |

### 环境变量

无特定工具的环境变量。限制通过输入参数（`head_limit`、`offset`）控制。

### 特性标志

没有特性标志直接控制 GrepTool 行为。

## 错误处理

### 验证错误（I/O 前）

| 错误代码 | 条件 | 消息 |
|---|---|---|
| 1 | 路径不存在 | "Path does not exist: {path}. … Did you mean {suggestion}?" |

### 运行时错误

| 错误 | 条件 | 处理 |
|---|---|---|
| `ENOENT` | 验证期间路径未找到 | 建议 cwd 下的类似路径 |
| `RipgrepTimeoutError` | Ripgrep 执行超时（WSL、大型代码库） | 向上传播到代理循环——发出不完整搜索信号 |
| Abort | `abortController.signal` 触发 | 将取消传播到 ripgrep 子进程 |

### 错误 UI 渲染

`UI.tsx` 中的 `renderToolUseErrorMessage()` 提供用户友好的错误显示：
- 检测"file not found"模式（包含 `FILE_NOT_FOUND_CWD_NOTE`）并显示"File not found"
- 检测 `<tool_use_error>` 标签并显示"Error searching files"
- 回退到 `FallbackToolUseErrorMessage` 以处理其他错误

### 按模式的 结果映射

`mapToolResultToToolResultBlockParam()` 按模式不同地格式化结果：

**Content 模式**：显示带可选分页页脚的匹配行：
```
[matching content lines]

[Showing results with pagination = limit: 250, offset: 0]
```

**Count 模式**：显示计数行和摘要页脚：
```
[filename:count lines]

Found 42 total occurrences across 8 files. with pagination = limit: 250
```

**Files with matches 模式**：显示文件计数标题和文件列表：
```
Found 15 files
src/file1.ts
src/file2.ts
...
```

## 性能优化

### Ripgrep 作为搜索引擎

选择 Ripgrep 是因为其性能优势：
- 基于 Rust 的实现，带 SIMD 优化
- 自动尊重 `.gitignore`
- 并行目录遍历
- 内存映射文件读取
- 智能正则表达式引擎，带字面量优化

### 自动排除

多种排除策略减少搜索范围：
- **VCS 目录**：`.git`、`.svn`、`.hg`、`.bzr`、`.jj`、`.sl` 始终排除
- **隐藏文件**：默认包含（`--hidden`），但受 `.gitignore` 规则约束
- **权限忽略模式**：从权限上下文提取并应用为 `--glob` 排除
- **孤立的插件目录**：通过 `getGlobExclusionsForPluginCache()` 动态排除

### 行长度限制

`--max-columns 500` 防止极长行（精简 JavaScript、base64 字符串、生成代码）弄乱输出和消耗上下文令牌。

### Head Limit 应用顺序

在 content 和 count 模式中，`head_limit` 在路径相对化**之前**应用：

```typescript
// 首先应用 head_limit — relativize 是每行工作，所以
// 避免处理将被丢弃的行
const { items: limitedResults, appliedLimit } = applyHeadLimit(
  results, head_limit, offset,
)
```

这避免了对将被丢弃的行进行不必要的字符串操作，这对于返回 10k+ 行的宽泛模式很重要。

### 路径相对化

所有返回的路径通过 `toRelativePath()` 从绝对路径转换为相对路径（相对于 `cwd`）。这减少了工具结果中的令牌使用，特别是对于深度嵌套的项目。

### 延迟 Schema 评估

输入和输出 schema 使用 `lazySchema()` 延迟 Zod schema 构建直到首次访问，减少冷启动开销。

### AbortController 支持

`abortController.signal` 传递到 `ripGrep()` 函数，在用户导航离开或请求被替代时启用长时间搜索的取消。

### WSL 超时处理

WSL 对 WSL2 上的文件读取有 3-5 倍的性能惩罚。Ripgrep 通过 `execFile` 超时在内部处理超时，`RipgrepTimeoutError` 向上传播到代理循环。这是刻意的——它发出搜索未完成的信号，而不是暗示没有找到匹配。

### 排序优化

对于 `files_with_matches` 模式，文件 stat 通过 `Promise.allSettled()` 并行获取：
- 单个 stat 失败不会阻塞整个排序
- 失败的 stat 默认为 mtime 0（排序到最后）
- 在测试模式下，基于文件名的排序确保确定性结果

### 默认 Head Limit 原理

250 的默认限制根据 20K 字符工具结果持久化阈值校准（grep 重会话约 6-24K 令牌）。它对于探索性搜索来说足够慷慨，同时防止上下文膨胀。用户可以显式传递 `head_limit: 0` 以获得无限结果。

## 测试

`GrepTool/` 目录中没有特定于工具的测试文件。测试可能由更广泛的工具测试套件中的集成测试覆盖。

## 相关模块

- [GlobTool](./GlobTool.md) — 互补的文件模式匹配工具；共享 UI 渲染
- [BashTool](./BashTool.md) — 用于目录列表；GrepTool 优于通过 Bash 的 `rg`/`grep`
- [AgentTool](./AgentTool.md) — 用于需要多轮的开放式搜索
- [ripgrep utility](../05-utils/ripgrep.md) — Ripgrep 子进程包装器

## 备注

- 对用户显示的工具名称是 `'Search'`（通过 `userFacingName()`），而内部标识符是 `'Grep'`
- 用户被明确指示**永远不要**将 `grep` 或 `rg` 作为 Bash 命令调用——GrepTool 有优化的权限和访问
- Ripgrep 模式语法与 grep 不同：字面量大括号需要转义（使用 `interface\{\}` 在 Go 代码中找到 `interface{}`）
- 多行模式（`multiline: true`）启用跨行模式，其中 `.` 匹配换行符，但由于性能影响应谨慎使用
- 该工具标记为 `isConcurrencySafe: true` 和 `isReadOnly: true`，表明它可以与其他工具并行调用并且不修改状态
- 工具定义上的 `strict: true` 标志强制执行严格的输入 schema 验证
- `files_with_matches` 模式的结果按修改时间排序（最新的优先），文件名作为决胜
