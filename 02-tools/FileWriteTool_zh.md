# FileWriteTool

## 目的

FileWriteTool 在本地文件系统创建新文件或覆盖现有文件。它是完整文件内容替换的主要工具，用于模型从头创建新文件或执行完整重写。与 FileEditTool（仅发送差异）不同，FileWriteTool 在单个操作中替换整个文件内容。

## 位置

- `restored-src/src/tools/FileWriteTool/FileWriteTool.ts` — 主工具实现（约 435 行）
- `restored-src/src/tools/FileWriteTool/prompt.ts` — 工具名称和描述字符串（约 19 行）
- `restored-src/src/tools/FileWriteTool/UI.tsx` — React UI 渲染组件（约 405 行）

## 关键导出

### 函数

- `FileWriteTool`：通过 `buildTool()` 构建的完整工具定义
- `countLines(content)`：计算文件内容中的可见行数，将尾随换行符视为终止符
- `userFacingName(input)`：返回 `'Write'` 或 `'Updated plan'`（用于计划文件）
- `getWriteToolDescription()`：返回显示给模型的工具提示
- `getToolUseSummary(input)`：返回活动摘要的显示路径
- `isResultTruncated(output)`：确定 UI 中是否截断了创建的文件内容

### 类型

- `Output`：`{ type: 'create' | 'update', filePath, content, structuredPatch, originalFile, gitDiff? }`
- `FileWriteToolInput`：带 `file_path` 和 `content` 的 Zod 输入 schema 类型

### 常量

- `FILE_WRITE_TOOL_NAME`：`'Write'`
- `DESCRIPTION`：`'Write a file to the local filesystem.'`
- `MAX_LINES_TO_RENDER`：`10` — 压缩 UI 视图中显示的最大行数
- `FILE_UNEXPECTEDLY_MODIFIED_ERROR`：并发修改检测的错误消息

## 依赖

### 内部依赖

| 模块 | 用途 |
|--------|---------|
| `utils/file.ts` | `writeTextContent()`、`getFileModificationTime()`、`getDisplayPath()` |
| `utils/fileRead.ts` | `readFileSyncWithMetadata()` — 带编码检测的同步文件读取 |
| `utils/fsOperations.ts` | `getFsImplementation()` — 抽象的文件系统操作 |
| `utils/permissions/filesystem.ts` | `checkWritePermissionForTool()`、`matchingRuleForInput()` |
| `utils/permissions/shellRuleMatching.ts` | `matchWildcardPattern()` — glob 样式权限匹配 |
| `utils/path.ts` | `expandPath()` — 将 `~` 和相对路径解析为绝对路径 |
| `utils/diff.ts` | `getPatchForDisplay()`、`countLinesChanged()` |
| `utils/gitDiff.ts` | `fetchSingleFileGitDiff()` — 生成更改的 git diff |
| `utils/fileHistory.ts` | `fileHistoryEnabled()`、`fileHistoryTrackEdit()` — 备份跟踪 |
| `utils/fileOperationAnalytics.ts` | `logFileOperation()` — 分析日志 |
| `services/lsp/manager.js` | 文件更改的 LSP 服务器通知 |
| `services/lsp/LSPDiagnosticRegistry.js` | 写入前的诊断清除 |
| `services/mcp/vscodeSdkMcp.js` | VSCode 文件更改通知 |
| `services/teamMemorySync/teamMemSecretGuard.js` | 文件内容中的秘密检测 |
| `skills/loadSkillsDir.js` | 从文件路径进行动态技能发现 |
| `services/analytics/growthbook.js` | 功能标志评估 |
| `tools/FileEditTool/constants.js` | 修改文件的共享错误常量 |
| `tools/FileEditTool/types.js` | 补丁和 diff 的共享 Zod schema |

### 外部依赖

| 包 | 用途 |
|---------|---------|
| `zod/v4` | 输入/输出 schema 验证 |
| `path` (Node.js) | `dirname()`、`sep()`、路径操作 |
| `diff` | 结构化补丁生成 |

## 实现细节

### 文件创建

工具区分两种操作类型：

1. **创建** — 写入不存在的文件（stat 上为 `isENOENT`）
2. **更新** — 覆盖现有文件

区别在 `validateInput()` 期间通过尝试 `fs.stat(fullFilePath)` 进行：

```
fs.stat(fullFilePath)
  ├── ENOENT → 新文件创建（validateInput 返回 { result: true }）
  ├── stat 成功 → 现有文件（继续进行陈旧性检查）
  └── 其他错误 → 重新抛出
```

对于新文件，输出设置 `type: 'create'`、`originalFile: null` 和 `structuredPatch: []`。对于更新，它包含原始内容和结构化 diff。

### 写入实现

写入路径遵循严格序列以确保数据完整性：

```
1. expandPath(file_path) → fullFilePath
2. dirname(fullFilePath) → dir
3. discoverSkillDirsForPaths([fullFilePath], cwd) → newSkillDirs（异步）
4. activateConditionalSkillsForPaths([fullFilePath], cwd)
5. diagnosticTracker.beforeFileEdited(fullFilePath)
6. fs.mkdir(dir) → 确保父目录存在
7. fileHistoryTrackEdit() → 备份预编辑内容（如果启用）
8. readFileSyncWithMetadata(fullFilePath) → meta（当前文件状态）
9. 陈旧性检查：比较 mtime 与 readFileState 时间戳
10. writeTextContent(fullFilePath, content, enc, 'LF') → 原子写入
11. LSP 通知：clearDeliveredDiagnostics + changeFile + saveFile
12. notifyVscodeFileUpdated(fullFilePath, oldContent, content)
13. 使用新内容和时间戳更新 readFileState
14. 记录 CLAUDE.md 写入（分析事件）
15. 计算 git diff（如果远程 + 功能标志启用）
16. 返回结果：type、content、structuredPatch、originalFile、gitDiff
```

**关键部分**：步骤 8-10 形成原子读-修改-写部分。在陈旧性检查（步骤 9）和实际写入（步骤 10）之间不允许异步操作，以防止与并发编辑的竞态条件。

**行尾策略**：工具按模型提供的精确内容写入，并明确强制执行 `'LF'` 行尾。它不会保留旧文件的行尾或采样仓库的行尾约定。这防止了文件静默损坏（例如，在覆盖 CRLF 文件时，Linux 上的 bash 脚本带有 `\r`）。

### 目录创建

写入前，工具确保父目录存在：

```typescript
await getFsImplementation().mkdir(dir)
```

此调用放在关键部分之外（在陈旧性检查和写入之间），有两个原因：

1. 检查和写入之间的 `yield` 将允许并发编辑交错
2. `writeTextContent` 内部的延迟 mkdir-on-ENOENT 会在实际 ENOENT 传播之前触发虚假的 `tengu_atomic_write_error`

`mkdir` 调用是幂等的——如果目录已存在则静默成功。

### 权限处理

工具实现多层权限系统：

#### 1. 写入权限检查（`checkPermissions`）

委托给 `checkWritePermissionForTool()` 进行评估：
- 始终允许规则
- 始终拒绝规则
- 权限模式（bypassPermissions、acceptEdits、auto 等）
- 钩子（PreToolUse/PostToolUse）
- 自动模式分类器

#### 2. 权限匹配器（`preparePermissionMatcher`）

返回一个将通配符模式与文件路径匹配的函数：

```typescript
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}
```

这支持细粒度权限规则，如 `Write(src/**/*.ts)` 仅允许写入 `src/` 中的 TypeScript 文件。

#### 3. 路径扩展（`backfillObservableInput`）

在权限评估**之前**将 `~` 和相对路径扩展为绝对路径。这防止通过路径技巧绕过权限允许列表：

```typescript
backfillObservableInput(input) {
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)
  }
}
```

#### 4. 拒绝规则验证（`validateInput`）

检查目标路径是否匹配任何拒绝规则：

```typescript
const denyRule = matchingRuleForInput(
  fullFilePath,
  appState.toolPermissionContext,
  'edit',
  'deny',
)
if (denyRule !== null) {
  return { result: false, message: 'File is in a directory that is denied...', errorCode: 1 }
}
```

### 安全检查

工具实现多种安全机制以防止数据丢失和损坏：

#### 1. 写前读取要求

文件必须在可以写入之前读取。`validateInput()` 函数检查 `readFileState`：

```typescript
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
    errorCode: 2,
  }
}
```

部分视图（offset/limited reads）被视为未读取——模型必须在写入之前执行完整读取。

#### 2. 修改时间陈旧性检查

工具检测文件自上次读取以来是否已修改：

```typescript
const lastWriteTime = Math.floor(fileMtimeMs)
if (lastWriteTime > readTimestamp.timestamp) {
  return {
    result: false,
    message: 'File has been modified since read...',
    errorCode: 3,
  }
}
```

#### 3. 写入前双重检查（竞态条件预防）

在关键部分内部的写入之前立即进行第二次陈旧性检查：

```typescript
if (meta !== null) {
  const lastWriteTime = getFileModificationTime(fullFilePath)
  const lastRead = readFileState.get(fullFilePath)
  if (!lastRead || lastWriteTime > lastRead.timestamp) {
    // 完整读取的内容比较回退
    const isFullRead = lastRead && lastRead.offset === undefined && lastRead.limit === undefined
    if (!isFullRead || meta.content !== lastRead.content) {
      throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
    }
  }
}
```

这种双重检查模式（TOCTOU 缓解）捕获在 `validateInput()` 和 `call()` 之间发生的修改。对于完整读取，它比较实际内容作为回退，以避免仅基于时间戳的误报（云同步、Windows 上的防病毒扫描）。

#### 4. UNC 路径保护

在 Windows 上，UNC 路径（`\\server\share` 或 `//server/share`）触发 SMB 身份验证，这可能会泄漏凭据。工具跳过 UNC 路径的文件系统操作，并让权限系统处理它们：

```typescript
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true } // 跳过 stat 检查，让权限检查处理
}
```

#### 5. 团队内存秘密保护

写入前，工具检查内容是否包含不应写入团队内存文件的秘密：

```typescript
const secretError = checkTeamMemSecrets(fullFilePath, content)
if (secretError) {
  return { result: false, message: secretError, errorCode: 0 }
}
```

#### 6. 路径验证

- `file_path` 必须是绝对路径（由 Zod schema 描述强制执行）
- 路径通过 `expandPath()` 扩展以解析 `~`、环境变量和相对组件

### 错误处理

工具在 `validateInput()` 中使用结构化错误代码系统：

| 错误代码 | 条件 | 消息 |
|------------|-----------|--------|
| `0` | 在团队内存文件中检测到秘密 | 特定于秘密的错误消息 |
| `1` | 路径匹配拒绝规则 | "File is in a directory that is denied by your permission settings." |
| `2` | 写入前文件未读取 | "File has not been read yet. Read it first before writing to it." |
| `3` | 文件自读取后已修改 | "File has been modified since read, either by the user or by a linter..." |

**运行时错误**（在 `call()` 期间抛出）：

| 错误 | 条件 |
|-------|-----------|
| `FILE_UNEXPECTEDLY_MODIFIED_ERROR` | 读取和写入之间文件内容已更改（双重检查失败） |
| FS 错误来自 `mkdir()` | 父目录创建失败 |
| FS 错误来自 `writeTextContent()` | 写入失败（权限、磁盘满等） |

**错误 UI 渲染**：

- 非详细模式：红色显示"Error writing file"
- 详细模式：通过 `<FallbackToolUseErrorMessage />` 显示完整回退错误消息

### 原子写入

写入操作使用执行原子写入的 `writeTextContent()`：

1. 内容写入同一目录中的临时文件
2. 临时文件刷新到磁盘（`fsync`）
3. 临时文件重命名覆盖目标文件（POSIX 和 Windows 上的原子操作）

这确保了：
- 目标文件永远不会处于部分写入状态
- 写入期间断电或崩溃不会损坏文件
- 并发读者看到旧内容或新内容，绝不会是混合内容

原子写入受陈旧性检查保护——如果另一个进程在检查和重命名之间修改了文件，写入抛出 `FILE_UNEXPECTEDLY_MODIFIED_ERROR`，而不是静默覆盖。

## 数据流

```
Model 返回 tool_use { name: 'Write', input: { file_path, content } }
  |
  v
backfillObservableInput() → 将路径扩展为绝对路径
  |
  v
preparePermissionMatcher() → 为路径构建通配符匹配器
  |
  v
checkPermissions() → 评估权限规则
  |
  v
validateInput() → 安全检查：
  ├── checkTeamMemSecrets() → 如果检测到秘密则拒绝
  ├── matchingRuleForInput() → 如果路径被拒绝则拒绝
  ├── UNC 路径检查 → 跳过 UNC 路径的 stat
  ├── fs.stat() → 检查文件是否存在
  ├── readFileState.get() → 验证文件已读取
  └── mtime 比较 → 验证文件未修改
  |
  v
call() → 执行写入：
  ├── discoverSkillDirsForPaths() → 后台技能发现
  ├── activateConditionalSkillsForPaths() → 激活匹配技能
  ├── diagnosticTracker.beforeFileEdited() → LSP 准备
  ├── fs.mkdir(dir) → 确保父目录
  ├── fileHistoryTrackEdit() → 备份（如果启用）
  ├── readFileSyncWithMetadata() → 读取当前状态
  ├── 陈旧性双重检查 → 验证无并发修改
  ├── writeTextContent() → 带 LF 行尾的原子写入
  ├── LSP 通知 → changeFile + saveFile
  ├── notifyVscodeFileUpdated() → VSCode 集成
  ├── readFileState.set() → 更新读取缓存
  ├── logEvent('tengu_write_claudemd') → 如果是 CLAUDE.md
  ├── fetchSingleFileGitDiff() → 计算 diff（如果远程）
  └── 返回 { type, filePath, content, structuredPatch, originalFile, gitDiff }
  |
  v
mapToolResultToToolResultBlockParam() → 序列化为 API
  |
  v
renderToolResultMessage() → 在终端 UI 中渲染
```

## 集成点

### LSP（语言服务器协议）

写入后，工具通知 LSP 服务器：

1. **清除诊断** — `clearDeliveredDiagnosticsForFile()` 移除陈旧诊断
2. **内容更改** — `lspManager.changeFile()` 发送 `textDocument/didChange`
3. **文件保存** — `lspManager.saveFile()` 发送 `textDocument/didSave`（触发 TypeScript 服务器诊断）

LSP 通知中的错误被捕获并记录，但不会导致写入操作失败。

### VSCode 集成

`notifyVscodeFileUpdated(fullFilePath, oldContent, content)` 通知 VSCode 关于文件更改，启用 diff 视图和外部编辑器同步。

### 文件历史

当文件历史启用（`fileHistoryEnabled()`）时，工具通过 `fileHistoryTrackEdit()` 跟踪编辑。这创建了按内容哈希键控的备份，支持撤销功能。备份在陈旧性检查之前创建——如果陈旧性稍后失败，未使用的备份是无害的。

### Git Diff 计算

对于启用了 `tengu_quartz_lantern` 功能标志的远程会话，工具计算更改的 git diff：

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_quartz_lantern', false)) {
  const diff = await fetchSingleFileGitDiff(fullFilePath)
  if (diff) gitDiff = diff
}
```

记录持续时间以进行性能监控。

### 动态技能发现

写入文件可以触发技能发现：

1. `discoverSkillDirsForPaths()` — 查找与文件路径匹配的技能目录
2. `addSkillDirectories()` — 注册新技能（后台、非阻塞）
3. `activateConditionalSkillsForPaths()` — 激活路径模式匹配的技能

这支持基于模型正在编辑的文件进行上下文感知技能加载。

### 分析

- 文件操作通过 `logFileOperation()` 记录，操作类型、工具名称、文件路径和创建/更新分类
- CLAUDE.md 写入触发特殊 `tengu_write_claudemd` 事件
- Git diff 计算持续时间通过 `tengu_tool_use_diff_computed` 跟踪

## 配置

### 功能标志

| 标志 | 用途 |
|------|---------|
| `tengu_quartz_lantern` | 为远程会话启用 git diff 计算 |

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_CODE_REMOTE` | 为真时，启用 git diff 计算 |

### 工具限制

| 设置 | 值 |
|---------|-------|
| `maxResultSizeChars` | 100,000 |
| `strict` | `true`（tengu_tool_pear 的严格模式） |

## 错误处理

### 验证错误（预执行）

验证错误在权限检查之前被捕获，并作为错误消息返回给模型。模型可以在解决该问题后重试（例如，先读取文件）。

### 并发修改错误

工具实现双重检查模式以检测并发修改：

1. **`validateInput()` 中的第一次检查** — 将文件 mtime 与读取时间戳进行比较
2. **`call()` 中的第二次检查** — 如果时间戳不同则重新读取文件并比较内容

此 TOCTOU（检查时间-使用时间）缓解捕获：
- 用户在外部编辑器中的编辑
- Linter 自动修复
- 云同步更改
- 防病毒文件修改

### 写入失败

写入失败（磁盘满、权限拒绝等）作为抛出错误传播。查询引擎捕获这些错误并通过 `renderToolUseErrorMessage()` 渲染它们。

## 测试

### 错误代码测试

每个验证错误代码可以通过设置相应前置条件来测试：

| 代码 | 测试设置 |
|------|------------|
| `0` | 写入包含秘密的团队内存文件 |
| `1` | 写入与拒绝规则匹配的路径 |
| `2` | 写入从未读取的文件 |
| `3` | 在读取后但在写入前修改文件 |

### 边缘情况

- **UNC 路径**：应跳过 stat 检查并让权限系统处理
- **部分读取**：应视为未读取（写入前需要完整读取）
- **新文件**：应自动创建父目录
- **现有文件**：应需要先读取和陈旧性检查
- **CLAUDE.md**：应触发特殊分析事件
- **行尾**：模型提供的行尾应保留（不重写）

## 相关模块

- [FileReadTool](./FileReadTool.md) — 互补读取工具；文件必须在写入前读取
- [FileEditTool](./FileEditTool.md) — 基于 diff 的编辑；修改现有文件的首选工具
- [Tool System](../01-core-modules/tool-system.md) — 核心工具抽象和注册表
- [Permission System](../01-core-modules/permission-system.md) — 权限评估和规则
- [File History](../05-utils/file-history.md) — 备份和撤销功能

## 注意事项

- 工具名称是 `'Write'`（不是 `FileWriteTool`）—— 这是模型看到的名称
- 计划文件（在 plans 目录中）获得特殊 UI 处理："Updated plan" 名称和 `/plan to preview` 提示
- 创建的文件在 UI 中被截断为 10 行，带有"... +N lines"指示器
- 更新的文件显示结构化 diff 而不是完整内容
- 工具使用 `z.strictObject()` 进行输入验证——未知字段被拒绝
- 输出 schema 包含 `gitDiff` 作为可选字段——仅在功能标志启用时存在
- `backfillObservableInput` 变异对安全至关重要——它防止通过路径技巧绕过权限
