# FileReadTool

## 目的

`FileReadTool`（对用户暴露为 **Read** 工具）是 Claude Code 中的主要文件读取机制。它从本地文件系统读取文本文件、图像、PDF 和 Jupyter notebook，返回带行号、元数据的内容，并根据每种文件类型进行适当格式化。它包括去重、令牌预算、权限检查和多格式渲染。

## 位置

`restored-src/src/tools/FileReadTool/FileReadTool.ts`

支持文件：
- `restored-src/src/tools/FileReadTool/limits.ts` — 令牌和大小限制配置
- `restored-src/src/tools/FileReadTool/prompt.ts` — 工具提示模板和常量
- `restored-src/src/tools/FileReadTool/UI.tsx` — 用于使用/结果/错误消息的终端 UI 渲染
- `restored-src/src/tools/FileReadTool/imageProcessor.ts` — Sharp/image-processor-napi 抽象

## 关键导出

### 函数

- `registerFileReadListener(listener)`：注册每次文件读取时调用的回调。返回取消订阅函数。
- `readImageWithTokenBudget(filePath, maxTokens, maxBytes?)`：读取图像文件并在需要时应用基于令牌的压缩。处理调整大小、强力压缩和回退路径。
- `getDefaultFileReadingLimits()`：返回当前 `FileReadingLimits` 配置的记忆化函数（优先级：env var > GrowthBook > 硬编码默认值）。

### 类

- `MaxFileReadTokenExceededError`：当文件内容超过令牌预算时抛出的自定义错误。包括 `tokenCount` 和 `maxTokens` 属性以供程序化处理。

### 类型

- `Input`：Zod 推断的输入 schema — `{ file_path: string, offset?: number, limit?: number, pages?: string }`
- `Output`：结果类型的区分联合：`text`、`image`、`notebook`、`pdf`、`parts`、`file_unchanged`
- `FileReadingLimits`：`{ maxTokens: number, maxSizeBytes: number, includeMaxSizeInPrompt?: boolean, targetedRangeNudge?: boolean }`
- `ImageResult`：`{ type: 'image', file: { base64, type, originalSize, dimensions? } }`

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `FILE_READ_TOOL_NAME` | `'Read'` | 内部工具标识符 |
| `FILE_UNCHANGED_STUB` | `'File unchanged since last read…'` | 去重命中时返回的消息 |
| `MAX_LINES_TO_READ` | `2000` | 未指定 `limit` 时的默认行数 |
| `DEFAULT_MAX_OUTPUT_TOKENS` | `25000` | 文本读取的默认令牌预算 |
| `DESCRIPTION` | `'Read a file from the local filesystem.'` | 短工具描述 |
| `LINE_FORMAT_INSTRUCTION` | `'Results are returned using cat -n format…'` | 行号说明 |
| `OFFSET_INSTRUCTION_DEFAULT` | `'- You can optionally specify a line offset…'` | 默认偏移指导 |
| `OFFSET_INSTRUCTION_TARGETED` | `'- When you already know which part…'` | 目标范围提示变体 |
| `CYBER_RISK_MITIGATION_REMINDER` | 系统提醒文本 | 附加到非豁免模型的结果文本 |
| `BLOCKED_DEVICE_PATHS` | 路径集合 | 会挂起或阻止的设备文件（例如 `/dev/zero`、`/dev/stdin`） |
| `IMAGE_EXTENSIONS` | `Set(['png','jpg','jpeg','gif','webp'])` | 识别的图像文件扩展名 |
| `MITIGATION_EXEMPT_MODELS` | `Set(['claude-opus-4-6'])` | 跳过网络风险提醒的模型 |

## 依赖

### 内部依赖

- `readFileInRange`（`src/utils/readFileInRange.ts`）— 带快速和流式代码路径的核心行导向文件读取器
- `checkReadPermissionForTool`（`src/utils/permissions/filesystem.ts`）— 权限评估
- `expandPath`（`src/utils/path.ts`）— 路径规范化（tilde 扩展、空白修剪、Windows 分隔符）
- `countTokensWithAPI`、`roughTokenCountEstimationForFileType`（`src/services/tokenEstimation.ts`）— 令牌预算
- `readNotebook`、`mapNotebookCellsToToolResult`（`src/utils/notebook.ts`）— Jupyter notebook 处理
- `extractPDFPages`、`getPDFPageCount`、`readPDF`（`src/utils/pdf.ts`）— PDF 读取和页面提取
- `maybeResizeAndDownsampleImageBuffer`、`compressImageBufferWithTokenLimit`（`src/utils/imageResizer.ts`）— 图像压缩管道
- `logFileOperation`（`src/utils/fileOperationAnalytics.ts`）— 文件操作遥测
- `discoverSkillDirsForPaths`、`activateConditionalSkillsForPaths`（`src/skills/loadSkillsDir.ts`）— 从文件路径进行技能发现
- `findSimilarFile`、`suggestPathUnderCwd`、`addLineNumbers`（`src/utils/file.ts`）— 文件未找到帮助和内容格式化
- `getFeatureValue_CACHED_MAY_BE_STALE`（`src/services/analytics/growthbook.ts`）— 用于限制和去重 kill-switch 的功能标志评估

### 外部依赖

- `zod/v4` — 输入/输出 schema 验证
- `fs/promises` — 异步文件系统操作
- `sharp` — 图像处理（调整大小、压缩、格式转换）
- `image-processor-napi` — 用于捆绑构建的原生图像处理器
- `lodash-es/memoize` — 限制配置的记忆化

## 实现细节

### 核心逻辑

工具使用 `buildTool()` 构建，遵循多阶段执行管道：

1. **输入验证**（`validateInput`）— 无 I/O 的纯检查：页面解析、拒绝规则、UNC 路径检测、二进制扩展名检查、阻止设备路径检查
2. **权限检查**（`checkPermissions`）— 根据配置的规则评估读取权限
3. **去重** — 检查 `readFileState` 缓存中相同的文件+范围+未更改-mtime
4. **技能发现** — 发现并激活匹配文件目录的技能（即发即忘）
5. **内部调用**（`callInner`）— 基于文件扩展名的类型特定读取调度

`callInner` 函数调度到四个不同的处理器：

| 文件类型 | 扩展名 | 处理器 | 输出类型 |
|---|---|---|---|
| Jupyter Notebook | `.ipynb` | `readNotebook()` | `notebook` |
| 图像 | `.png,.jpg,.jpeg,.gif,.webp` | `readImageWithTokenBudget()` | `image` |
| PDF | `.pdf` | `readPDF()` / `extractPDFPages()` | `pdf` 或 `parts` |
| 文本（默认） | 其他所有 | `readFileInRange()` | `text` |

### 输入 Schema

```typescript
z.strictObject({
  file_path: z.string(),                    // 文件的绝对路径
  offset: z.number().int().nonnegative(),   // 开始的 1-indexed 行
  limit: z.number().int().positive(),       // 读取的行数
  pages: z.string(),                        // PDF 页面范围（例如 "1-5"、"3"）
})
```

### 输出 Schema（区分联合）

```typescript
z.discriminatedUnion('type', [
  // 文本文件结果
  z.object({ type: 'text', file: { filePath, content, numLines, startLine, totalLines } }),
  // 带 base64 数据的图像结果
  z.object({ type: 'image', file: { base64, type, originalSize, dimensions? } }),
  // Jupyter notebook 结果
  z.object({ type: 'notebook', file: { filePath, cells } }),
  // 完整 PDF 结果
  z.object({ type: 'pdf', file: { filePath, base64, originalSize } }),
  // 提取的 PDF 页面
  z.object({ type: 'parts', file: { filePath, originalSize, count, outputDir } }),
  // 去重命中 — 文件自上次读取后未更改
  z.object({ type: 'file_unchanged', file: { filePath } }),
])
```

### 关键算法

#### 去重策略

当读取文件时，其内容、修改时间戳、偏移量和限制存储在 `readFileState`（`Map<string, FileState>`）中。在后续读取时：

1. 在 `readFileState` 中查找文件路径
2. 检查存储的条目是否是完整视图（不是来自 Edit/Write 的部分读取）
3. 将请求的 offset/limit 与存储的范围进行比较
4. 重新 stat 磁盘上的文件并将 `mtimeMs` 与存储的时间戳进行比较
5. 如果全部匹配，返回 `FILE_UNCHANGED_STUB` 而不是重新发送内容

此优化将 Read 调用的 cache_creation 令牌使用减少约 18%。由 `tengu_read_dedup_killswitch` 功能标志控制。

#### 令牌预算

两个上限适用于文本读取：

| 限制 | 默认 | 检查点 | 成本 | 超出时 |
|---|---|---|---|---|
| `maxSizeBytes` | 256 KB | 预读取（文件 stat） | 1 次 stat 调用 | 读取前抛出 |
| `maxTokens` | 25,000 | 读取后（内容分析） | API 往返 | 读取后抛出 |

令牌估计使用两阶段方法：
1. **快速估计**：`roughTokenCountEstimationForFileType()` — 基于文件类型的廉价启发式
2. **精确计数**：`countTokensWithAPI()` — 仅在快速估计超过 `maxTokens / 4` 时调用

这避免了，对明显在预算内的文件进行不必要的 API 调用。

#### 带令牌预算的图像处理

`readImageWithTokenBudget()` 函数实现分层压缩策略：

1. **读取一次**：文件读取到缓冲区一次（由 `maxBytes` 限制以防止 OOM）
2. **格式检测**：`detectImageFormatFromBuffer()` 从魔数识别图像格式
3. **标准调整大小**：`maybeResizeAndDownsampleImageBuffer()` 应用正常调整大小
4. **令牌检查**：估计令牌 = `base64.length * 0.125`
5. **强力压缩**：如果超出预算，`compressImageBufferWithTokenLimit()` 从同一缓冲区应用更强压缩（不重新读取）
6. **回退**：如果压缩失败，通过 sharp 生成硬编码的 400x400 JPEG，质量 20
7. **最终回退**：如果 sharp 失败，返回原始缓冲区

#### 截图路径解析（macOS）

macOS 截图文件名根据 OS 版本在 AM/PM 前使用常规空格或窄空格（U+202F）。`getAlternateScreenshotPath()` 检测此模式并返回备用路径，以在原始路径返回 ENOENT 时尝试。

### 边缘情况

- **阻止的设备路径**：像 `/dev/zero`、`/dev/random`、`/dev/stdin` 这样的文件通过路径检查被阻止（无 I/O）以防止挂起
- **UNC 路径**：网络路径（`\\server\share` 或 `//server/share`）通过验证但在实际 I/O 之前延迟到权限授予后（防止 NTLM 凭据泄漏）
- **二进制文件**：拒绝并显示有用消息，PDF、图像和 SVG 除外
- **空文件**：返回系统提醒警告而不是空内容
- **偏移量超出文件长度**：返回指示文件短于请求偏移量的警告
- **macOS 截图空格**：尝试 AM/PM 文件名中的备用空格字符（常规 vs 窄空格）
- **大型 PDF**：超过 `PDF_AT_MENTION_INLINE_THRESHOLD` 页的 PDF 需要 `pages` 参数
- **PDF 提取回退**：当 PDF.js 不支持或文件超过 `PDF_EXTRACT_SIZE_THRESHOLD` 时，通过 poppler-utils 回退到页面提取

## 数据流

```
User Request
    │
    ▼
┌─────────────────────┐
│  validateInput()    │  ← 纯检查：页面、拒绝规则、UNC、二进制、设备
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  checkPermissions() │  ← 根据规则进行权限评估
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Dedup Check        │  ← readFileState 缓存 + mtime 比较
│  (optional stub)    │
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  Skill Discovery    │  ← 即发即忘技能激活
└─────────┴───────────┘
          │
          ▼
┌─────────────────────┐
│  callInner()        │  ← 基于扩展名的调度
│  ┌───────────────┐  │
│  │ .ipynb → read │  │
│  │ image → read  │  │
│  │ .pdf → read   │  │
│  │ text → read   │  │
│  └───────────────┘  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Token Validation   │  ← 快速估计 → API 计数（如需要）
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Result Mapping     │  ← mapToolResultToToolResultBlockParam
│  + Line Numbers     │    + 行号 + 缓解说明
│  + Mitigation Note  │
└─────────┴───────────┘
          │
          ▼
    Tool Result
```

## 集成点

### FileReadListener 系统

其他服务可以通过 `registerFileReadListener()` 订阅文件读取事件。这被用于：
- 内存系统 — 跟踪哪些文件已被读取以用于上下文感知
- 分析 — 日志文件读取模式和会话文件访问

### 会话文件检测

`detectSessionFileType()` 识别 Claude 何时读取自己的会话文件：
- **会话内存**：`~/.claude/session-memory/*.md` — 带 `is_session_memory: true` 记录
- **会话转录**：`~/.claude/projects/*/*.jsonl` — 带 `is_session_transcript: true` 记录

这支持对自引用读取和潜在反馈循环的分析。

### 内存附件触发器

成功读取后，文件路径被添加到 `context.nestedMemoryAttachmentTriggers`，向内存系统发出信号以考虑此文件以供将来回合附加。

### 自动内存新鲜度

对于检测为自动内存文件（`isAutoMemFile()`）的文件，`WeakMap` 存储 mtime，稍后由 `memoryFileFreshnessPrefix()` 使用以将新鲜度说明附加到渲染结果。

## 配置

### 环境变量

| 变量 | 描述 |
|---|---|
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | 覆盖默认最大令牌预算（25,000）。必须是正整数。 |
| `CLAUDE_CODE_SIMPLE` | 设置时，在文件读取期间禁用技能发现。 |

### 功能标志

| 标志 | 描述 | 默认 |
|---|---|---|
| `tengu_amber_wren` | 自定义限制（`maxSizeBytes`、`maxTokens`、`includeMaxSizeInPrompt`、`targetedRangeNudge`）的 GrowthBook 实验 | `{}`（使用硬编码默认值） |
| `tengu_read_dedup_killswitch` | 如果导致模型混淆则禁用去重优化 | `false`（去重启用） |

### 限制优先级（maxTokens）

```
Env var → GrowthBook 标志 → DEFAULT_MAX_OUTPUT_TOKENS (25,000)
```

每个字段单独验证；无效值回退到下一层。没有到达 `maxTokens = 0` 的路径。

## 错误处理

### 验证错误（预 I/O）

| 错误代码 | 条件 | 消息 |
|---|---|---|
| 1 | 文件路径匹配拒绝规则 | "File is in a directory that is denied by your permission settings." |
| 4 | 二进制文件扩展名（非 PDF/图像） | "This tool cannot read binary files…" |
| 7 | 无效的 pages 参数格式 | "Invalid pages parameter…" |
| 8 | 页面范围超出 `PDF_MAX_PAGES_PER_READ` | "Page range exceeds maximum…" |
| 9 | 阻止的设备路径 | "Cannot read… this device file would block…" |

### 运行时错误

| 错误 | 条件 | 处理 |
|---|---|---|
| `MaxFileReadTokenExceededError` | 内容超出令牌预算 | 包括实际计数和最大值；建议使用 `offset`/`limit` |
| `ENOENT` | 文件未找到 | 尝试 macOS 截图备用路径，然后建议类似文件或 CWD 相对路径 |
| 图像调整大小错误 | 图像处理失败 | 回退到未压缩缓冲区，然后强力压缩，然后到原始缓冲区 |
| PDF 读取错误 | PDF 解析失败 | 抛出带描述性消息；建议页面提取或 poppler-utils 安装 |
| Notebook 大小错误 | 序列化单元格超出 `maxSizeBytes` | 建议使用 Bash 工具和 `jq` 读取特定部分 |

### 错误 UI 渲染

`UI.tsx` 中的 `renderToolUseErrorMessage()` 提供用户友好的错误显示：
- 检测"文件未找到"模式并显示简洁的"File not found"消息
- 检测 `<tool_use_error>` 标签并显示"Error reading file"
- 回退到 `FallbackToolUseErrorMessage` 用于其他错误

## 性能优化

### 单次读取图像管道

图像精确读取到缓冲区一次。所有压缩操作（标准调整大小、强压缩、回退）在同一缓冲区上操作——不重新读取。这消除了需要多次压缩尝试的图像的冗余 I/O。

### 快速与流式文件读取

`readFileInRange()` 内部在两种策略之间选择：
- **快速路径**：对于在大小限制内的文件，读取整个文件并在内存中切片行
- **流式路径**：对于大文件，使用 `createReadStream` 流式逐行读取，避免将整个文件加载到内存中

### 令牌估计缓存

两阶段令牌估计避免昂贵的 API 调用：
1. 快速启发式（`roughTokenCountEstimationForFileType`）首先运行 — O(n) 字符串扫描
2. API 调用（`countTokensWithAPI`）仅在估计超过 `maxTokens / 4` 时触发

### 记忆化限制配置

`getDefaultFileReadingLimits()` 通过 `lodash-es/memoize` 记忆化。GrowthBook 标志值在第一次调用时固定，防止 cap 在后台标志刷新时在会话中期更改。

### 内存文件 Mtime 的 WeakMap

`memoryFileMtimes` WeakMap 存储按结果数据对象身份键控的 mtime 数据。这避免了：
- 向输出 schema 添加仅表示字段（会流入 SDK 类型）
- 在结果映射器中进行同步 `fs.stat` 调用
- 内存泄漏（当数据对象变得不可访问时，WeakMap 条目自动 GC'd）

### 后台技能加载

技能发现（`discoverSkillDirsForPaths`）和激活（`activateConditionalSkillsForPaths`）是即发即忘操作，不会阻止文件读取响应。技能在后台异步加载。

## 测试

工具包括渲染保真度测试，验证 UI.tsx 摘要渲染——确保它显示元数据（"Read N lines"、"Read image (42KB)"）而不是原始内容。此测试捕获了初始实现错误，其中 `file.content` 错误地用于摘要 chrome。

## 相关模块

- [FileWriteTool](./FileWriteTool.md) — 互补的文件写入工具
- [FileEditTool](./FileEditTool.md) — 互补的文件编辑工具
- [BashTool](./BashTool.md) — 用于目录列表和二进制文件检查
- [readFileInRange](../05-utils/file-state-cache.md) — 核心行导向文件读取实用工具
- [tool-system](../01-core-modules/tool-system.md) — 工具抽象和注册表

## 注意事项

- 工具标记为 `isConcurrencySafe: true` 和 `isReadOnly: true`，表明它可以与其他工具并行调用并且不修改状态
- `maxResultSizeChars` 设置为 `Infinity`，因为输出由 `maxTokens` 验证绑定——将工具结果持久化到文件将是循环的
- `backfillObservableInput` 钩子将相对路径扩展为绝对路径，防止通过 `~` 或相对路径技巧绕过权限允许列表
- PDF 内容作为补充 `DocumentBlockParam` 发送到 API，而工具结果块仅包含元数据（文件路径和大小）
- 网络风险缓解提醒附加到非豁免模型（当前仅为 `claude-opus-4-6`）的文本文件结果
