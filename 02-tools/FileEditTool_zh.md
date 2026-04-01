# FileEditTool

## 目的

执行文件中的精确搜索和替换字符串替换。它是就地修改现有文件内容的主要工具，支持单个或多个替换、文件创建（通过空的 `old_string`）、引号规范化、补丁生成和文件历史跟踪。

## 位置

`restored-src/src/tools/FileEditTool/FileEditTool.ts`

支持文件：
- `types.ts` — Zod 输入/输出 schema 和 TypeScript 类型
- `utils.ts` — 核心编辑逻辑：`findActualString`、`preserveQuoteStyle`、`applyEditToFile`、`getPatchForEdits`、代码片段提取、输入规范化、等效性检查
- `UI.tsx` — 终端 UI 组件，用于工具使用、结果、拒绝和错误消息（包括内联差异渲染）
- `prompt.ts` — 呈现给模型的工具描述和使用说明
- `constants.ts` — 工具名称和错误消息常量

## 关键导出

### 函数

- `FileEditTool` — 通过 `buildTool()` 构建的工具定义对象，包含所有生命周期钩子
- `findActualString` — 使用引号规范化在文件内容中定位搜索字符串
- `preserveQuoteStyle` — 将 `actualOldString` 的弯引号样式应用于 `newString`
- `applyEditToFile` — 对内容应用单个字符串替换（单个或全局）
- `getPatchForEdit` / `getPatchForEdits` — 从前后内容生成统一的 diff 块
- `normalizeFileEditInput` — 预规范化输入（去规范化、尾随空白剥离）
- `areFileEditsEquivalent` — 通过应用两个编辑集并比较结果进行语义等效性检查
- `areFileEditsInputsEquivalent` — 两个完整工具输入的统一等效性检查
- `getSnippetForPatch` / `getSnippet` — 提取更改行周围的上下文窗口片段
- `getSnippetForTwoFileDiff` — 为附件生成有界 diff 片段
- `getEditsForPatch` — 从结构化补丁块反向工程 `FileEdit[]`
- `normalizeQuotes` — 将弯引号替换为直引号
- `stripTrailingWhitespace` — 每行移除尾随空白，同时保留行尾

### 类型

- `FileEditInput` — 解析的工具输入：`{ file_path, old_string, new_string, replace_all }`
- `FileEdit` — 不带文件路径的单个编辑：`{ old_string, new_string, replace_all }`
- `EditInput` — 与 `FileEdit` 相同，但有可选的 `replace_all`
- `FileEditOutput` — 工具结果：`{ filePath, oldString, newString, originalFile, structuredPatch, userModified, replaceAll, gitDiff? }`

### 常量

- `FILE_EDIT_TOOL_NAME` — `'Edit'`
- `FILE_UNEXPECTEDLY_MODIFIED_ERROR` — 并发文件修改的错误消息
- `MAX_EDIT_FILE_SIZE` — `1 GiB` (1024³ bytes)，防止 OOM
- `DIFF_SNIPPET_MAX_BYTES` — `8192`，附件 diff 片段大小上限
- `CONTEXT_LINES` — `4`，片段中更改周围的默认上下文行数
- `CHANGE_THRESHOLD` — `0.4`，词级与行级 diff 渲染的阈值
- 弯引号常量：`LEFT_SINGLE_CURLY_QUOTE`、`RIGHT_SINGLE_CURLY_QUOTE`、`LEFT_DOUBLE_CURLY_QUOTE`、`RIGHT_DOUBLE_CURLY_QUOTE`

## 依赖

### 内部依赖

| 模块 | 用途 |
|--------|---------|
| `Tool.ts` | 工具构建器、类型定义、权限框架 |
| `utils/fileRead.ts` | `readFileSyncWithMetadata`，编码和行尾检测 |
| `utils/file.ts` | `writeTextContent`、`findSimilarFile`、`suggestPathUnderCwd`、`getFileModificationTime`、`addLineNumbers`、`convertLeadingTabsToSpaces`、`readFileSyncCached` |
| `utils/diff.ts` | `getPatchForDisplay`、`getPatchFromContents`、`adjustHunkLineNumbers`、`countLinesChanged`、`DIFF_TIMEOUT_MS` |
| `utils/fileHistory.ts` | `fileHistoryEnabled`、`fileHistoryTrackEdit` 用于撤销支持 |
| `utils/gitDiff.ts` | `fetchSingleFileGitDiff` 用于远程 diff 生成 |
| `utils/permissions/filesystem.ts` | `checkWritePermissionForTool`、`matchingRuleForInput` |
| `utils/settings/validateEditTool.ts` | `validateInputForSettingsFileEdit` 用于 CLAUDE.md 保护 |
| `services/lsp/` | LSP 服务器通知（`changeFile`、`saveFile`、`clearDeliveredDiagnosticsForFile`） |
| `services/mcp/vscodeSdkMcp.ts` | `notifyVscodeFileUpdated` 用于 VS Code diff 视图 |
| `services/teamMemorySync/teamMemSecretGuard.ts` | `checkTeamMemSecrets` 用于秘密检测 |
| `skills/loadSkillsDir.ts` | 从编辑文件路径进行动态技能发现 |
| `components/StructuredDiff/` | 终端 UI 中的词级 diff 高亮 |

### 外部依赖

| 包 | 用途 |
|--------|---------|
| `diff` | `structuredPatch`、`diffWordsWithSpace` 用于统一 diff 生成和词级高亮 |
| `zod/v4` | 带 `semanticBoolean` 预处理的输入/输出 schema 验证 |

## 实现细节

### 搜索和替换实现

该工具使用**字面字符串搜索和替换**方法（不是基于正则表达式）。核心替换由 `utils.ts` 中的 `applyEditToFile` 执行：

```typescript
const f = replaceAll
  ? (content, search, replace) => content.replaceAll(search, () => replace)
  : (content, search, replace) => content.replace(search, () => replace)
```

使用回调函数（`() => replace`）而不是直接字符串，可以防止 `$&`、`$'`、`$`` 和其他 `String.prototype.replace` 替换模式被解释——确保替换文本逐字插入。

**搜索解析**遵循 `findActualString` 中的两阶段过程：

1. **精确匹配** — `fileContent.includes(searchString)` 用于最快路径
2. **引号规范化匹配** — 如果精确匹配失败，文件内容和搜索字符串都通过 `normalizeQuotes`（弯引号 → 直引号），然后使用规范化搜索的索引从原始文件内容中提取实际子字符串

这允许编辑成功，即使模型输出直引号（`'`、`"`）而文件包含弯引号（`''`、`""`）。

### Diff 生成

Diff 生成使用 `diff` 库的 `structuredPatch` 函数，通过两个实用函数包装：

**`getPatchForEdits`**（主要路径）：
1. 顺序应用每个编辑以构建 `updatedFile`
2. 检测编辑之间的子字符串冲突（如果 `old_string` 是前一个 `new_string` 的子字符串则抛出错误）
3. 验证每个编辑实际更改了内容
4. 调用 `getPatchFromContents` 使用转换为空格的制表符内容生成显示安全补丁

**`getPatchFromContents`**（在 `utils/diff.ts` 中）：
- 使用 `context: 3`（默认统一 diff 上下文）调用 `structuredPatch(oldPath, newPath, oldContent, newContent)`
- 将前导制表符转换为空格，以便补丁渲染与终端制表符宽度无关

生成的 `StructuredPatchHunk[]` 在工具输出中返回，并被 UI 组件用于终端 diff 渲染和远程代码路径的 git diff 生成。

### 冲突解决

多个冲突场景在不同阶段处理：

**验证阶段**（`validateInput`）：

| 场景 | 错误代码 | 行为 |
|----------|-----------|--------|
| `old_string === new_string` | 1 | 拒绝——没有更改要做 |
| 目录被权限拒绝 | 2 | 拒绝——权限设置阻止写入 |
| 文件太大（>1 GiB） | 10 | 拒绝——防止 OOM |
| 文件不存在，`old_string` 非空 | 4 | 拒绝——建议类似文件或 CWD 路径 |
| 文件存在，`old_string` 为空（不是创建） | 3 | 拒绝——文件已有内容 |
| Jupyter Notebook（`.ipynb`） | 5 | 拒绝——重定向到 `NotebookEditTool` |
| 文件尚未读取 | 6 | 拒绝——需要先调用 `FileReadTool` |
| 文件自读取后已修改 | 7 | 拒绝——需要重新读取 |
| `old_string` 未找到（引号规范化后） | 8 | 拒绝——字符串不在文件中 |
| 多个匹配，`replace_all` 为 false | 9 | 拒绝——请求更多上下文或 `replace_all: true` |

**执行阶段**（`call`）：

| 场景 | 处理 |
|----------|----------|
| 验证和写入之间文件意外修改 | 抛出 `FILE_UNEXPECTEDLY_MODIFIED_ERROR` |
| 编辑的 `old_string` 是前一个编辑 `new_string` 的子字符串 | 在 `getPatchForEdits` 中抛出错误 |
| 编辑没有产生更改 | 抛出"字符串在文件中未找到" |
| 所有编辑已应用但文件未更改 | 抛出"原始文件和编辑文件完全匹配" |

**内容级冲突** — 当 `replace_all` 为 false 且搜索字符串出现多次时，工具拒绝猜测要替换哪个匹配项。模型必须提供更多周围上下文以唯一标识目标。

### 撤销支持

文件历史通过 `fileHistory.ts` 管理：

1. **预编辑备份** — `fileHistoryTrackEdit` 在编辑前调用，捕获按内容哈希键控的预编辑内容。这是幂等的——如果相同内容已备份，则是无操作。
2. **功能门控** — 仅在 `fileHistoryEnabled()` 返回 true 时激活
3. **状态跟踪** — `updateFileHistoryState` 回调接收文件路径和父消息 UUID 以进行关联
4. **备份时间** — 在陈旧性检查之前调用，因此即使陈旧性验证失败，未使用的备份也是无害的（不是损坏状态）

文件历史系统通过维护内容快照链来实现 `/rewind` 命令和其他撤销操作。

### 多重编辑处理

`getPatchForEdits` 函数支持对单个文件顺序应用多个编辑：

1. 编辑**顺序**应用，每个都建立在前一个编辑的结果之上
2. **子字符串冲突检测** — 在应用每个编辑之前，检查 `old_string` 是否与所有先前应用的 `new_string` 值匹配。如果是任何先前替换的子字符串，则抛出错误以防止级联损坏
3. **更改验证** — 每个编辑后，将结果与预编辑内容进行比较。如果没有发生更改，则编辑被视为失败
4. **最终验证** — 所有编辑后，将最终内容与原始内容进行比较。如果相同，则抛出错误

`FileEditTool.ts` 中的单编辑 `call` 方法将单个 `old_string`/`new_string` 对包装成单元素数组以供 `getPatchForEdits` 使用。

### 错误处理

错误按其来源分类：

**验证错误** — 从 `validateInput` 返回 `{ result: false, behavior: 'ask', message, errorCode }`。这些显示给用户并带有重试选项。

**执行错误** — 在 `call()` 期间作为 `Error` 对象抛出。关键错误：
- `FILE_UNEXPECTEDLY_MODIFIED_ERROR` — 文件在读取和写入之间被外部修改

**权限错误** — 由权限框架处理：
- UNC 路径（`\\` 或 `//`）在文件系统检查期间被跳过，以防止 Windows 上的 NTLM 凭据泄漏
- 目录拒绝规则在任何文件系统访问之前检查
- 通过 `checkWritePermissionForTool` 验证写入权限

**UI 错误渲染** — `renderToolUseErrorMessage` 简化常见错误：
- "File has not been read yet" → "File must be read first"（暗淡）
- "File does not exist" → "File not found"（错误颜色）
- 其他错误 → "Error editing file"（错误颜色）

**去规范化回退** — 当精确字符串匹配失败时，`normalizeFileEditInput` 尝试使用查找表对 `old_string` 进行去规范化（例如 `<fnr>` → `<function_results>`、`<n>` → `<name>`）。如果去规范化成功，相同的替换将应用于 `new_string`。

## 数据流

```
Model Output
    │
    ▼
┌─────────────────────────────────┐
│  normalizeFileEditInput          │  去规范化、剥离尾随空白
│  (utils.ts)                      │  （跳过 .md/.mdx）
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  validateInput                   │  检查权限、文件状态、
│  (FileEditTool.ts)               │  字符串匹配、多重性
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  checkPermissions                │  写入权限验证
│  (FileEditTool.ts)               │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  call()                          │
│  ┌───────────────────────────┐  │
│  │ fileHistoryTrackEdit      │  │  备份预编辑内容
│  ├───────────────────────────┤  │
│  │ readFileForEdit           │  │  带编码+行尾读取
│  ├───────────────────────────┤  │
│  │ Staleness check           │  │  验证文件自读取后未更改
│  ├───────────────────────────┤  │
│  │ findActualString          │  │  定位字符串（带引号规范）
│  ├───────────────────────────┤  │
│  │ preserveQuoteStyle        │  │  将弯引号样式应用于 new
│  ├───────────────────────────┤  │
│  │ getPatchForEdit           │  │  生成统一 diff 块
│  ├───────────────────────────┤  │
│  │ writeTextContent          │  │  原子写入保留编码
│  ├───────────────────────────┤  │
│  │ LSP notification          │  │  didChange + didSave
│  ├───────────────────────────┤  │
│  │ VS Code notification      │  │  notifyVscodeFileUpdated
│  ├───────────────────────────┤  │
│  │ Update readFileState      │  │  使陈旧写入无效
│  ├───────────────────────────┤  │
│  │ Analytics logging         │  │  行数、字节长度
│  ├───────────────────────────┤  │
│  │ Git diff (remote only)    │  │  fetchSingleFileGitDiff
│  └───────────────────────────┘  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  renderToolResultMessage         │  终端 diff 显示 via
│  (UI.tsx)                        │  FileEditToolUpdatedMessage
└─────────────────────────────────┘
```

## 集成点

### LSP（语言服务器协议）

写入后，工具通知 LSP 服务器：
1. `clearDeliveredDiagnosticsForFile` — 清除之前的诊断，以便显示新的诊断
2. `lspManager.changeFile` — 发送 `textDocument/didChange` 通知
3. `lspManager.saveFile` — 发送 `textDocument/didSave` 通知（触发 TypeScript 诊断）

两个 LSP 调用都是即发即忘，带错误日志——失败不会阻止工具结果。

### VS Code 集成

`notifyVscodeFileUpdated` 使用文件路径、原始内容和更新内容调用。这使得当 Claude Code 作为 VS Code 扩展运行时，VS Code 能够显示更改的 diff 视图。

### 技能发现

编辑文件后，工具发现并激活与文件路径相关的技能：
- `discoverSkillDirsForPaths` — 查找与编辑文件匹配的技能目录
- `addSkillDirectories` — 注册新的技能目录（后台、非阻塞）
- `activateConditionalSkillsForPaths` — 激活路径模式匹配的技能

在简单模式下跳过（`CLAUDE_CODE_SIMPLE` 环境变量）。

### 文件读取状态

工具维护跟踪的 `readFileState` 映射：
- `timestamp` — 文件上次读取的时间
- `content` — 读取时的内容（对于完整读取）
- `offset` / `limit` — 对于部分读取（分块读取）

这支持陈旧性检测：如果文件的修改时间超过读取时间戳，则拒绝写入，除非验证内容未更改（完整读取比较回退）。

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | 为真时禁用技能发现 |
| `CLAUDE_CODE_REMOTE` | 为真时启用 git diff 生成 |

### 功能标志

| 标志 | 用途 |
|------|---------|
| `tengu_quartz_lantern` | 控制远程会话的 git diff 计算 |

### 大小限制

| 常量 | 值 | 用途 |
|----------|-------|---------|
| `MAX_EDIT_FILE_SIZE` | 1 GiB | 可编辑的最大文件大小（防止 OOM） |
| `maxResultSizeChars` | 100,000 | 工具结果中的最大字符数 |
| `DIFF_SNIPPET_MAX_BYTES` | 8,192 | 附件中 diff 片段的最大大小 |

## 错误处理

### 错误代码参考

| 代码 | 条件 | 用户消息 |
|------|-----------|-------------|
| 0 | 在团队内存文件中检测到秘密 | 秘密保护错误消息 |
| 1 | `old_string === new_string` | "No changes to make: old_string and new_string are exactly the same." |
| 2 | 目录被权限拒绝 | "File is in a directory that is denied by your permission settings." |
| 3 | 非空文件上的空 `old_string` | "Cannot create new file - file already exists." |
| 4 | 文件不存在 | "File does not exist."（+ 建议） |
| 5 | Jupyter Notebook 文件 | "File is a Jupyter Notebook. Use the NotebookEdit tool." |
| 6 | 编辑前未读取文件 | "File has not been read yet. Read it first before writing to it." |
| 7 | 文件自读取后已修改 | "File has been modified since read... Read it again before attempting to write it." |
| 8 | 未找到 `old_string` | "String to replace not found in file." |
| 9 | 多个匹配，`replace_all` false | "Found N matches... set replace_all to true" |
| 10 | 文件超过 1 GiB | "File is too large to edit (X). Maximum editable file size is 1 GiB." |

### 陈旧性检测（双重检查模式）

工具在**两个点**执行陈旧性检查：

1. **`validateInput`** — 使用 `readFileState` 时间戳和内容比较在权限检查之前
2. **`call()`** — 在获取当前状态后使用第二个时间戳检查

在这两个点之间，`readFileForEdit` 执行同步读取，创建了一个狭窄的关键部分。代码中的注释明确警告：*"Please avoid async operations between here and writing to disk to preserve atomicity."*

内容比较回退防止 Windows 上的误报，因为时间戳可以在没有内容更改的情况下更改（云同步、防病毒扫描）。

## 边缘情况

### 编码处理

**检测** — `readFileSyncWithMetadata` 读取前 4096 字节并检查：
- UTF-16 LE BOM（`0xFF 0xFE`）→ `'utf16le'`
- UTF-8 BOM（`0xEF 0xBB 0xBF`）→ `'utf8'`
- 默认 → `'utf8'`（ASCII 的超集，处理所有 Unicode）
- 空文件 → `'utf8'`（防止 emoji/CJK 损坏）

**保留** — 检测到的编码传递给 `writeTextContent`，以相同编码写回文件。这确保 UTF-16 文件在编辑后保持 UTF-16。

**验证阶段** — 将文件作为字节读取并以相同方式检测编码，将 `\r\n` → `\n` 规范化以获得一致的内部表示。

### 行尾处理

**检测** — `detectLineEndingsForString` 对前 4096 个字符进行采样并计算 CRLF 与 LF 的出现次数。记录主导样式。

**规范化** — 文件内容通过 `readFileSyncWithMetadata` 在内部规范化为 LF（`\n`）（`raw.replaceAll('\r\n', '\n')`）。

**保留** — 检测到的行尾样式（`'CRLF'` 或 `'LF'`）传递给 `writeTextContent`，在写入前转换回来。这确保 Windows 文件（CRLF）在编辑后保持 CRLF。

**`stripTrailingWhitespace`** — 通过在 `/(\r\n|\n|\r)/` 上分割来保留行尾，仅剥离内容行，不剥离行尾分隔符。

### 引号规范化

**问题** — Claude 模型无法输出弯/智能引号。如果文件包含 `''hello''` 且模型请求替换 `'hello'`，直引号将不匹配。

**解决方案** — 三阶段管道：
1. `findActualString` — 将文件和搜索都规范化为直引号进行匹配，然后从原始文件提取实际子字符串
2. `preserveQuoteStyle` — 检测匹配 `actualOldString` 中的弯引号，并将相同的开/闭弯引号样式应用于 `newString`
3. 开/闭启发式 — 前导空白、字符串开头或开头标点符号 `([{` 的引号被视为开引号；否则为闭引号

**缩写处理** — `applyCurlySingleQuotes` 检测两个字母之间的撇号（例如 `don't`、`it's`），始终使用右单弯引号（`'`），而不是左引号。

### Markdown 尾随空白

Markdown 和 MDX 文件使用两个尾随空格作为硬换行符。`normalizeFileEditInput` 跳过 `.md`/`.mdx` 文件的 `stripTrailingWhitespace` 以保留此语义。

### 空字符串编辑

- **创建新文件** — 在不存在的文件上 `old_string: ''` 是有效的，并使用 `new_string` 作为内容创建文件
- **写入空文件** — 在现有空文件上 `old_string: ''` 是有效的
- **写入非空文件** — 在有内容的文件上 `old_string: ''` 被拒绝（错误代码 3）
- **删除内容** — `new_string: ''` 是有效的，并移除匹配的 `old_string`
- **删除时尾随换行符** — 当 `new_string` 为空且 `old_string` 不以 `\n` 结尾但文件有 `old_string + '\n'` 时，尾随换行符也被移除

### 去规范化

当 Claude 输出包含规范化标记（由 API 转义）时，`normalizeFileEditInput` 反转它们：

| 规范化后 | 原始 |
|-----------|----------|
| `<fnr>` | `<function_results>` |
| `<n>` / `</n>` | `<name>` / `</name>` |
| `<o>` / `</o>` | `<output>` / `</output>` |
| `<e>` / `</e>` | `<error>` / `</error>` |
| `<s>` / `</s>` | `<system>` / `</system>` |
| `<r>` / `</r>` | `<result>` / `</result>` |
| `< META_START >` | `<META_START>` |
| `< META_END >` | `<META_END>` |
| `< EOT >` | `<EOT>` |
| `< META >` | `<META>` |
| `< SOS >` | `<SOS>` |
| `\n\nH:` | `\n\nHuman:` |
| `\n\nA:` | `\n\nAssistant:` |

### UNC 路径安全

在 Windows 上，以 `\\` 或 `//` 开头的路径是 UNC 路径。对 UNC 路径调用 `fs.existsSync()` 会触发 SMB 身份验证，这可能会向恶意服务器泄漏 NTLM 凭据。工具跳过 UNC 路径的文件系统检查，并让权限框架处理它们。

### V8 字符串长度限制

1 GiB 文件大小限制源自 V8/Bun 的约 2³⁰ 个字符（~10 亿）的字符串长度限制。对于 ASCII/Latin-1 文件，1 字节 = 1 个字符，因此磁盘上的 1 GiB ≈ 10 亿个字符。多字节 UTF-8 文件每个字符在磁盘上可能更大，但 1 GiB 是一个安全的字节级保护。

### Jupyter Notebooks

`.ipynb` 文件被明确拒绝并重定向到 `NotebookEditTool`。Jupyter Notebooks 是 JSON 结构，不是纯文本，需要单元格感知编辑。

### 设置文件保护

`validateInputForSettingsFileEdit` 对 Claude 设置文件（例如 `CLAUDE.md`）应用额外验证。它模拟编辑并根据 JSON/schema 约束验证结果，以防止配置文件损坏。

## 测试

工具的行为通过以下方式验证：

- **输入等效性检查** — `areFileEditsInputsEquivalent` 通过将两个编辑都应用于原始内容并比较结果来比较两个编辑输入的语义，处理不同编辑规范产生相同结果的情况
- **错误代码特异性** — 每个验证失败返回不同的错误代码以进行精确的错误处理和分析
- **补丁往返** — `getEditsForPatch` 可以从结构化补丁反向工程编辑，实现补丁正确性的验证

## 相关模块

- [FileReadTool](./FileReadTool.md) — 文件编辑的必需前提条件
- [FileWriteTool](./FileWriteTool.md) — 完整文件替换的替代方案
- [NotebookEditTool](./NotebookEditTool.md) — Jupyter Notebooks 的专用编辑器
- [tool-system](../01-core-modules/tool-system.md) — 工具抽象框架
- [StructuredDiff](../06-ui/StructuredDiff.md) — 终端 diff 渲染组件

## 注意事项

- 工具要求在编辑之前读取文件。这由 `readFileState` 跟踪强制执行，并防止盲目覆盖。
- 工具定义上的 `strict: true` 标志强制执行严格的 Zod schema 验证。
- `semanticBoolean` 预处理 `replace_all` 允许模型传递真/假值（例如 `0`、`1`、`"true"`），这些值被强制转换为正确的布尔值。
- Git diff 生成（`fetchSingleFileGitDiff`）仅在远程会话（`CLAUDE_CODE_REMOTE`）中且 `tengu_quartz_lantern` 功能标志启用时执行。
- 工具在陈旧性检查和写入之间的关键部分使用同步文件读取，以防止与并发编辑的 TOCTOU 竞态条件。
- 日志分析事件：`tengu_write_claudemd`（用于 CLAUDE.md 编辑）、`tengu_edit_string_lengths`（旧/新字符串的字节长度）、`tengu_tool_use_diff_computed`（git diff 时序）。
