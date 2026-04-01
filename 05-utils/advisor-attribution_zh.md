# 顾问与提交归属工具

## 目的

两个独立的工具模块：**顾问（Advisor）** 提供了一个工具，让 Claude 在进行实质性工作之前咨询更强的审查模型；**提交归属（Commit Attribution）** 跟踪并计算 Claude 对文件的字符级贡献，用于 git 提交 trailer，显示 AI 生成的变更百分比。

## 位置

- `restored-src/src/utils/advisor.ts` — 顾问工具类型、配置、模型支持、使用量提取
- `restored-src/src/utils/commitAttribution.ts` — 文件级归属跟踪、提交计算、快照持久化

## 主要导出

### 顾问（`advisor.ts`）

#### 类型
- `AdvisorServerToolUseBlock`: `{ type: 'server_tool_use', id, name: 'advisor', input }` — 来自模型的工具调用
- `AdvisorToolResultBlock`: `{ type: 'advisor_tool_result', tool_use_id, content }` — 来自顾问的结果。内容可以是 `advisor_result`（纯文本）、`advisor_redacted_result`（加密）或 `advisor_tool_result_error`。
- `AdvisorBlock`: 工具使用和工具结果块的联合类型

#### 函数
- `isAdvisorBlock(param)`: 类型守卫，检查块是否为顾问块
- `isAdvisorEnabled()`: 检查顾问工具是否启用。遵循环境变量 `CLAUDE_CODE_DISABLE_ADVISOR_TOOL`、仅限第一方的 beta 头信息以及 GrowthBook 配置 `tengu_sage_compass`。
- `canUserConfigureAdvisor()`: 检查用户是否可以配置自己的顾问模型
- `getExperimentAdvisorModels()`: 获取实验配置的基础模型和顾问模型对（当用户无法配置时）
- `modelSupportsAdvisor(model)`: 检查模型是否支持调用顾问工具（Opus 4.6、Sonnet 4.6 或 ant）
- `isValidAdvisorModel(model)`: 检查模型是否可以作为顾问模型（相同标准）
- `getInitialAdvisorSetting()`: 从设置中获取初始顾问模型
- `getAdvisorUsage(usage)`: 从 API 响应的 `usage.iterations` 数组中提取顾问迭代使用量

#### 常量
- `ADVISOR_TOOL_INSTRUCTIONS`: 顾问工具的完整系统提示指令。描述何时调用顾问（实质性工作之前、遇到困难时、完成时）、如何处理建议以及如何解决顾问指导与经验证据之间的冲突。

#### 配置
- 顾问配置来自 GrowthBook 特性 `tengu_sage_compass`，字段包括：`enabled`、`canUserConfigure`、`baseModel`、`advisorModel`

### 提交归属（`commitAttribution.ts`）

#### 类型
- `AttributionState`: 跟踪文件状态、会话基线、surface、起始 HEAD SHA、提示计数、权限提示计数和转义计数
- `AttributionSummary`: `{ claudePercent, claudeChars, humanChars, surfaces }` — 提交的摘要
- `FileAttribution`: `{ claudeChars, humanChars, percent, surface }` — 每个文件的详细信息
- `AttributionData`: git notes JSON 的完整归属数据（版本、摘要、文件、surface 细分、排除的生成文件、会话）

#### 常量
- `INTERNAL_MODEL_REPOS`: 允许内部模型名称出现在 git 提交 trailer 中的私有仓库列表。包括约 25 个仓库的 SSH 和 HTTPS URL 格式。

#### 仓库分类
- `getAttributionRepoRoot()`: 获取归属操作的仓库根目录（遵循 agent worktree 覆盖）
- `isInternalModelRepo`: 异步检查当前仓库是否在内部模型允许列表中。每个进程记忆化。
- `isInternalModelRepoCached()`: 同步缓存结果（安全默认值：false = 不泄露）
- `getRepoClassCached()`: 同步缓存分类：`'internal' | 'external' | 'none' | null`

#### 模型名称规范化
- `sanitizeModelName(shortName)`: 将内部模型变体映射到公共名称（例如 `opus-4-6` → `claude-opus-4-6`）
- `sanitizeSurfaceKey(surfaceKey)`: 规范化 surface 键（例如 `cli/opus-4-5-fast` → `cli/claude-opus-4-5`）

#### 文件跟踪
- `createEmptyAttributionState()`: 为新会话创建初始归属状态
- `trackFileModification(state, filePath, oldContent, newContent, userModified, mtime?)`: 跟踪 Claude 的文件编辑
- `trackFileCreation(state, filePath, content, mtime?)`: 跟踪 Claude 创建的文件（通过 bash 或其他非跟踪机制）
- `trackFileDeletion(state, filePath, oldContent)`: 跟踪 Claude 删除的文件
- `trackBulkFileChanges(state, changes)`: 批量跟踪多个文件更改（避免大 diff 的 O(n²) Map 复制）
- `getFileMtime(filePath)`: 获取文件 mtimeMs 用于在进入同步回调之前进行预计算

#### 提交计算
- `calculateCommitAttribution(states, stagedFiles)`: 计算暂存文件的最终归属。将会话基线与已提交状态进行比较。并行处理文件。排除生成文件。
- `getGitDiffSize(filePath)`: 从 git diff stat 输出获取更改大小
- `isFileDeleted(filePath)`: 检查文件是否在暂存更改中被删除
- `getStagedFiles()`: 从 git 获取暂存文件列表

#### Surface 键
- `getClientSurface()`: 从 `CLAUDE_CODE_ENTRYPOINT` 环境变量获取当前客户端 surface（默认为 `'cli'`）
- `buildSurfaceKey(surface, model)`: 构建包含模型名称的 surface 键（格式：`surface/model`）

#### 哈希和路径规范化
- `computeContentHash(content)`: 内容的 SHA-256 哈希
- `normalizeFilePath(filePath)`: 规范化到仓库根目录的相对路径。为 macOS `/tmp` 与 `/private/tmp` 的一致性解析符号链接。
- `expandFilePath(filePath)`: 展开到仓库根目录的绝对路径

#### 快照持久化
- `stateToSnapshotMessage(state, messageId)`: 将归属状态转换为日志持久化的快照消息
- `restoreAttributionStateFromSnapshots(snapshots)`: 从会话恢复时的日志快照恢复状态。只使用最后一个快照（不是求和），以避免二次增长问题。
- `attributionRestoreStateFromLog(attributionSnapshots, onUpdateState)`: 恢复并应用状态更新的包装器
- `incrementPromptCount(attribution, saveSnapshot)`: 增加提示计数并保存快照。用于在压缩期间保持计数持久化。

#### Git 状态
- `isGitTransientState()`: 检查是否处于瞬态 git 状态（rebase、merge、cherry-pick、bisect）

## 设计注意事项

- **顾问工具**：SDK 尚无顾问块的类型——当前类型是标记为 TODO 的占位符。顾问工具不需要参数；整个对话历史会自动转发。
- **归属字符计数**：使用公共前缀/后缀匹配来查找实际更改区域，正确处理相同长度的替换（例如 "Esc" → "esc"），其中 `Math.abs(newLen - oldLen)` 将为 0。
- **快照恢复 bug 修复**：代码只使用最后一个快照（不是对所有快照求和），以避免二次增长——837 个快照 × 280 个文件在 5 天会话中为 5KB 文件产生了 1.15 万亿"字符"。
- **内部模型名称保护**：模型名称在除特定允许列表确认的私有仓库外的所有仓库中规范化为公共等价物。这故意使用仓库允许列表，而不是组织范围的检查，因为 anthropics 组织包含公开仓库，其中 undercover 模式必须保持开启。
- **批量文件更改**：`trackBulkFileChanges` 创建 Map 的一个副本并为每个文件修改它，避免在处理大型 git diff（jj 操作涉及数十万文件）时产生 O(n²) 成本。
- **Surface 键** 包含模型名称（例如 `cli/claude-sonnet-4-6`），以便归属可以按做出更改的模型细分。
