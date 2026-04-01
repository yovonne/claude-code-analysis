# 压缩服务

## 目的

压缩服务管理对话上下文缩减，以防止模型的上下文窗口超出。它提供多种压缩策略 — 完整对话摘要、以 pivot 点为中心的部分压缩、基于会话记忆的压缩和微级工具结果修剪 — 每种由不同条件触发。该服务处理完整生命周期：预压缩钩子、基于 API 的摘要、压缩后上下文恢复以及失效缓存的清理。

## 位置

| 文件 | 行数 | 角色 |
|------|-------|------|
| `restored-src/src/services/compact/compact.ts` | 1706 | 核心压缩：完整和部分对话摘要 |
| `restored-src/src/services/compact/autoCompact.ts` | 352 | 基于令牌阈值的自动压缩触发 |
| `restored-src/src/services/compact/microCompact.ts` | 531 | 工具结果修剪（缓存和基于时间） |
| `restored-src/src/services/compact/sessionMemoryCompact.ts` | 631 | 基于会话记忆的压缩替代方案 |
| `restored-src/src/services/compact/prompt.ts` | 375 | 摘要提示模板和格式化 |
| `restored-src/src/services/compact/grouping.ts` | 64 | 用于压缩的 API 轮消息分组 |
| `restored-src/src/services/compact/timeBasedMCConfig.ts` | 44 | 基于时间的微压缩配置 |
| `restored-src/src/services/compact/postCompactCleanup.ts` | 78 | 压缩后缓存和状态清理 |
| `restored-src/src/services/compact/compactWarningState.ts` | 19 | 压缩警告抑制状态 |
| `restored-src/src/services/compact/compactWarningHook.ts` | 17 | 压缩警告抑制的 React 钩子 |
| `restored-src/src/services/compact/apiMicrocompact.ts` | 154 | 服务器端上下文管理 API 策略 |

## 关键导出

### compact.ts

| 导出 | 描述 |
|--------|-------------|
| `compactConversation(messages, context, ...)` | 完整对话压缩 — 摘要较旧的消息，保留最近的 |
| `partialCompactConversation(allMessages, pivotIndex, context, direction)` | 在 pivot 点周围的局部压缩（`'from'` 或 `'up_to'`） |
| `CompactionResult`（类型） | `{ boundaryMarker, summaryMessages, attachments, hookResults, messagesToKeep?, ... }` |
| `RecompactionInfo`（类型） | 用于分析的上下文：`isRecompactionInChain`、`turnsSincePreviousCompact` 等 |
| `buildPostCompactMessages(result)` | 构建一致的压缩后消息数组 |
| `annotateBoundaryWithPreservedSegment(boundary, anchorUuid, messagesToKeep)` | 为保留的消息段添加重新链接元数据 |
| `mergeHookInstructions(userInstructions, hookInstructions)` | 合并用户和钩子提供的自定义指令 |
| `stripImagesFromMessages(messages)` | 在压缩前移除图像/文档块 |
| `stripReinjectedAttachments(messages)` | 移除 skill_discovery/skill_listing 附件 |
| `truncateHeadForPTLRetry(messages, ptlResponse)` | 当压缩请求本身遇到提示过长时的紧急截断 |
| `createPostCompactFileAttachments(readFileState, context, maxFiles)` | 压缩后重新注入最近访问的文件 |
| `createPlanAttachmentIfNeeded(agentId)` | 跨压缩保留计划文件 |
| `createSkillAttachmentIfNeeded(agentId)` | 跨压缩保留调用的技能内容 |
| `createAsyncAgentAttachmentsIfNeeded(context)` | 跨压缩保留异步代理状态 |
| `createPlanModeAttachmentIfNeeded(context)` | 跨压缩保留计划模式指令 |

### autoCompact.ts

| 导出 | 描述 |
|--------|-------------|
| `shouldAutoCompact(messages, model, querySource?, snipTokensFreed)` | 确定是否应触发自动压缩 |
| `autoCompactIfNeeded(messages, toolUseContext, cacheSafeParams, ...)` | 使用会话记忆回退和断路器执行自动压缩 |
| `isAutoCompactEnabled()` | 检查环境变量和用户配置的自动压缩启用 |
| `getAutoCompactThreshold(model)` | 计算自动压缩触发的令牌阈值 |
| `getEffectiveContextWindowSize(model)` | 上下文窗口减去保留的摘要令牌 |
| `calculateTokenWarningState(tokenUsage, model)` | 返回警告/错误/自动压缩阈值状态 |
| `AutoCompactTrackingState`（类型） | 跟踪 `compacted`、`turnCounter`、`turnId`、`consecutiveFailures` |

### microCompact.ts

| 导出 | 描述 |
|--------|-------------|
| `microcompactMessages(messages, toolUseContext?, querySource?)` | 主入口 — 尝试基于时间的，然后是缓存的 MC，如果不触发则返回不变 |
| `estimateMessageTokens(messages)` | 消息的粗略令牌估计（按 4/3 填充） |
| `evaluateTimeBasedTrigger(messages, querySource)` | 检查基于时间的触发器是否应触发 |
| `consumePendingCacheEdits()` | 返回并清除 API 层的待处理缓存编辑 |
| `getPinnedCacheEdits()` | 返回缓存命中所需的先前固定的缓存编辑 |
| `pinCacheEdits(userMessageIndex, block)` | 将 cache_edits 块固定到消息位置 |
| `markToolsSentToAPIState()` | 标记所有已注册的工具为已发送到 API |
| `resetMicrocompactState()` | 重置所有微压缩状态 |

### sessionMemoryCompact.ts

| 导出 | 描述 |
|--------|-------------|
| `trySessionMemoryCompaction(messages, agentId?, autoCompactThreshold?)` | 尝试基于会话记忆的压缩；如果不适用则返回 `null` |
| `shouldUseSessionMemoryCompaction()` | 检查功能标志和环境变量 |
| `calculateMessagesToKeepIndex(messages, lastSummarizedIndex)` | 计算保留消息的起始索引 |
| `adjustIndexToPreserveAPIInvariants(messages, startIndex)` | 确保 tool_use/tool_result 配对不被拆分 |
| `hasTextBlocks(message)` | 检查消息是否包含文本内容 |
| `SessionMemoryCompactConfig`（类型） | `{ minTokens, minTextBlockMessages, maxTokens }` |

### prompt.ts

| 导出 | 描述 |
|--------|-------------|
| `getCompactPrompt(customInstructions?)` | 返回完整压缩提示模板 |
| `getPartialCompactPrompt(customInstructions?, direction)` | 返回部分压缩提示（方向感知） |
| `formatCompactSummary(summary)` | 剥离 `<analysis>` 标签，将 `<summary>` 标签格式化为标题 |
| `getCompactUserSummaryMessage(summary, suppressFollowUpQuestions?, transcriptPath?, recentMessagesPreserved?)` | 包装摘要以注入上下文 |

### grouping.ts

| 导出 | 描述 |
|--------|-------------|
| `groupMessagesByApiRound(messages)` | 在 API 轮边界分组消息（每轮一次 API 往返） |

### postCompactCleanup.ts

| 导出 | 描述 |
|--------|-------------|
| `runPostCompactCleanup(querySource?)` | 压缩后清除缓存、跟踪状态和模块级状态 |

### timeBasedMCConfig.ts

| 导出 | 描述 |
|--------|-------------|
| `getTimeBasedMCConfig()` | 返回基于时间的微压缩的 GrowthBook 配置 |
| `TimeBasedMCConfig`（类型） | `{ enabled, gapThresholdMinutes, keepRecent }` |

### apiMicrocompact.ts

| 导出 | 描述 |
|--------|-------------|
| `getAPIContextManagement(options?)` | 返回服务器端上下文管理策略 |
| `ContextEditStrategy`（类型） | `clear_tool_uses_20250919` 或 `clear_thinking_20251015` 策略 |
| `ContextManagementConfig`（类型） | `{ edits: ContextEditStrategy[] }` |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `services/api/claude.ts` | `queryModelWithStreaming`、`getMaxOutputTokensForModel` |
| `services/api/errors.ts` | 提示过长检测和令牌间隙提取 |
| `services/api/promptCacheBreakDetection.ts` | 缓存中断通知 |
| `services/SessionMemory/prompts.ts` | 会话记忆内容和截断 |
| `services/SessionMemory/sessionMemoryUtils.ts` | 会话记忆状态管理 |
| `services/analytics/growthbook.ts` | 功能标志门 |
| `services/analytics/index.ts` | 事件日志 |
| `services/tokenEstimation.ts` | 粗略令牌计数 |
| `services/contextCollapse/` | 上下文折叠抑制（启用时） |
| `utils/forkedAgent.ts` | 用于缓存共享压缩的分叉代理 |
| `utils/messages.ts` | 消息创建、边界检测、规范化 |
| `utils/tokens.ts` | 来自 API 响应的令牌计数 |
| `utils/hooks.ts` | 预/后压缩钩子执行 |
| `utils/attachments.ts` | 压缩后文件恢复 |
| `utils/context.ts` | 上下文窗口大小 |
| `utils/contextAnalysis.ts` | 用于分析的令牌细分 |
| `utils/plans.ts` | 计划文件访问 |
| `utils/sessionStorage.ts` | 脚本路径、会话元数据 |
| `utils/sessionStart.ts` | 会话开始钩子处理 |
| `utils/config.ts` | 全局配置访问 |
| `utils/toolSearch.ts` | 压缩后的工具发现 |
| `tools/FileReadTool/` | 压缩后文件重新读取 |

### 外部

| 包 | 用途 |
|---------|---------|
| `@anthropic-ai/sdk` | API 客户端，`APIUserAbortError` |
| `lodash-es/uniqBy` | 工具去重 |
| `react` | 压缩警告钩子（隔离文件） |

## 实现细节

### 上下文压缩算法

#### 完整压缩 (`compactConversation`)

主要压缩策略在保留最近上下文的同时摘要整个对话：

```
1. 执行 PreCompact 钩子
2. 使用分析指令构建压缩提示
3. 通过 API 流式传输摘要（带缓存共享的分叉代理，或直接流式传输）
4. 处理提示过长重试（截断最旧的组）
5. 清除文件读取状态和已加载的记忆
6. 生成压缩后附件（文件、代理、计划、技能、工具、MCP）
7. 执行 SessionStart 钩子
8. 创建边界标记和摘要消息
9. 执行 PostCompact 钩子
10. 返回包含所有组件的 CompactionResult
```

**缓存共享优化**：当 `tengu_compact_cache_prefix` 启用（默认：true）时，使用重用主对话缓存提示前缀的分叉代理。失败时回退到直接流式传输。

**提示过长重试**：如果压缩请求本身超出提示限制：
- 逐步剥离最旧的 API 轮组直到低于限制
- 最多 3 次重试
- 如果第一个剩余消息是助手，则预先添加合成用户标记
- 当令牌间隙不可解析时回退到 20% 组删除

#### 部分压缩 (`partialCompactConversation`)

在用户选择的 pivot 点周围摘要对话的一部分：

**方向 `'from'`**（保留后缀）：
- 摘要 pivot 索引**之后**的消息
- 保留 pivot 索引**之前**的消息不变
- 保留消息的提示缓存被保留
- 摘要出现在保留消息之后

**方向 `'up_to'`**（保留前缀）：
- 摘要 pivot 索引**之前**的消息
- 保留 pivot 索引**之后**的消息不变
- 提示缓存失效（摘要在保留消息之前）
- 从保留消息中剥离旧的压缩边界
- 摘要出现在保留消息之前

#### 会话记忆压缩 (`trySessionMemoryCompaction`)

API 基于摘要的替代方案，使用预提取的会话记忆：

```
1. 检查功能标志（tengu_session_memory + tengu_sm_compact）
2. 等待进行中的会话记忆提取
3. 获取会话记忆内容和 lastSummarizedMessageId
4. 计算要保留的消息（扩展以满足最小值）：
   - 至少 minTokens（默认：10,000）
   - 至少 minTextBlockMessages（默认：5）
   - 最多 maxTokens（默认：40,000）
5. 调整索引以保留 tool_use/tool_result 配对
6. 从保留消息中过滤掉旧的压缩边界
7. 使用会话记忆作为摘要创建 CompactionResult
8. 根据自动压缩阈值检查压缩后大小
```

**两种场景**：
- **正常**：`lastSummarizedMessageId` 已设置 — 仅保留该 ID 之后的消息
- **恢复的会话**：没有 `lastSummarizedMessageId` 但会话记忆有内容 — 保留所有消息但使用会话记忆作为摘要

#### 微压缩 (`microcompactMessages`)

轻量级、每轮工具结果修剪，在每个 API 调用之前运行：

**基于时间的触发**（首先运行短路）：
- 当自上次助手消息以来的间隙超过 `gapThresholdMinutes`（默认：60 分钟）时触发
- 清除内容除了最近 N 个可压缩工具结果
- 重置缓存的 MC 状态（内容更改后缓存变冷）
- 直接改变消息内容

**缓存微压缩**（仅限 ant，功能门控）：
- 使用缓存编辑 API 删除工具结果而不使缓存前缀失效
- **不**修改本地消息内容 — 在 API 层添加 `cache_reference` 和 `cache_edits`
- 基于计数的触发器：当注册的工具结果超过阈值时，最旧的要删除
- 按用户消息分组跟踪工具结果
- 返回要包含在下一个 API 请求中的 `pendingCacheEdits`

**可压缩工具**：FileRead、Bash、PowerShell、REPL、Grep、Glob、WebSearch、WebFetch、FileEdit、FileWrite

### 消息摘要

#### 提示结构

压缩提示指示模型生成具有 9 个部分的结构化摘要：

1. **主要请求和意图** — 用户明确的请求
2. **关键技术概念** — 技术、框架、模式
3. **文件和代码段** — 检查/修改的文件及代码片段
4. **错误和修复** — 遇到的错误及解决方法
5. **问题解决** — 已解决的问题和故障排除
6. **所有用户消息** — 非工具结果的用户消息
7. **待处理任务** — 剩余工作项
8. **当前工作** — 压缩时正在处理的内容
9. **可选的下一步** — 下一个操作及逐字引用

#### 分析阶段

提示包含 `<analysis>` 部分作为起草草稿，在摘要进入上下文之前被剥离。这通过强制模型在产生最终输出之前组织思路来提高摘要质量。

#### 无工具强制

前导和尾部都明确禁止在压缩期间调用工具：
- 前导："CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."
- 尾部："REMINDER: Do NOT call any tools. Respond with plain text only."

这防止模型将单个回合浪费在工具调用上，否则将导致没有文本输出。

#### 摘要格式化

`formatCompactSummary()` 处理原始 API 响应：
1. 完全剥离 `<analysis>...</analysis>` 块
2. 将 `<summary>...</summary>` 标签替换为 `Summary:\n` 标题
3. 清理各部分之间的多余空白

### 令牌优化策略

#### 压缩后上下文恢复

压缩后，以下附件被重新注入以恢复模型上下文：

| 附件 | 用途 | 令牌预算 |
|-----------|---------|-------------|
| 最近文件（最多 5 个） | 重新读取最近访问的文件 | 总计 50,000，每文件 5,000 |
| 计划文件 | 保留当前计划 | 无限制 |
| 调用的技能 | 保留技能指南 | 总计 25,000，每技能 5,000 |
| 计划模式 | 保留计划模式状态 | 无限制 |
| 异步代理 | 保留代理状态 | 无限制 |
| 延迟工具增量 | 重新宣布可用工具 | 计算得出 |
| 代理列表增量 | 重新宣布可用代理 | 计算得出 |
| MCP 指令增量 | 重新宣布 MCP 工具 | 计算得出 |

**文件选择**：按最近程度排序，过滤掉计划文件和 CLAUDE.md 文件，与已保留消息中的 Read 工具结果去重，然后按预算约束。

**技能截断**：每个技能截断保留技能文件的头部（设置/使用说明）而不是丢弃整个技能。使用 `~4 chars/token` 估计。

#### 图像剥离

在发送到摘要 API 之前：
- 图像块替换为 `[image]` 文本标记
- 文档块替换为 `[document]` 文本标记
- `tool_result` 内容中的图像也被剥离
- 防止压缩 API 调用本身遇到提示过长

#### 重新注入附件剥离

`skill_discovery` 和 `skill_listing` 附件在压缩前被移除，因为它们通过 `resetSentSkillNames()` 和下一轮发现信号重新出现。将它们提供给摘要者会浪费令牌并用陈旧的建议污染摘要。

### 何时触发压缩

#### 自动压缩

当令牌使用超过自动压缩阈值时触发：

```
threshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
effectiveContextWindow = contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY
AUTOCOMPACT_BUFFER_TOKENS = 13,000
```

**抑制条件**（自动压缩不会触发）：
- `DISABLE_COMPACT` 或 `DISABLE_AUTO_COMPACT` 环境变量
- 用户配置 `autoCompactEnabled: false`
- `querySource` 是 `session_memory`、`compact` 或 `marble_origami`（防止死锁）
- `REACTIVE_COMPACT` 功能标志与 `tengu_cobalt_raccoon` 启用
- `CONTEXT_COLLAPSE` 功能启用（折叠拥有上下文管理）
- 断路器：3+ 次连续失败

**执行流程**：
```
1. shouldAutoCompact() — 检查令牌计数与阈值
2. trySessionMemoryCompaction() — 尝试 SM 优先（如果不适用则返回 null）
3. compactConversation() — 回退到完整 API 压缩
4. runPostCompactCleanup() — 清除缓存和跟踪状态
5. 跟踪 consecutiveFailures 用于断路器
```

#### 手动压缩

由 `/compact` 命令触发。相同的 `compactConversation()` 路径，但：
- **不**抑制用户问题
- 失败时显示错误通知（自动压缩不会）
- 无 recompaction 信息跟踪

#### 部分压缩

由消息选择器 UI 触发。用户选择 pivot 消息和方向。

#### 反应式压缩

当 API 返回提示过长错误时触发。如果启用，回退到自动压缩。

#### 微压缩（每轮）

在主线程上每个 API 调用之前运行：
- 基于时间：空闲间隙后触发（默认：60 分钟）
- 缓存 MC：当工具结果计数超过阈值时触发

### 压缩模式

| 模式 | 触发器 | 摘要内容 | 缓存影响 |
|------|---------|-------------------|-------------|
| **完整** | 自动或手动 | 所有消息 | 缓存失效（新系统前缀） |
| **部分（from）** | 消息选择器 | pivot 后的消息 | 保留前缀的缓存被保留 |
| **部分（up_to）** | 消息选择器 | pivot 前的消息 | 缓存失效（摘要在保留之前） |
| **会话记忆** | 自动（SM 优先） | lastSummarizedId 前的消息 | 缓存失效 |
| **基于时间的 MC** | 空闲间隙（60 分钟） | 旧工具结果内容 | 缓存失效（内容更改） |
| **缓存 MC** | 工具计数阈值 | 通过缓存编辑的工具结果 | 缓存被保留（API 层编辑） |
| **API 上下文管理** | 服务器端令牌限制 | 思考块、工具使用 | 服务器管理 |

### 压缩后清理

`runPostCompactCleanup()` 使依赖于预压缩上下文的所有状态失效：

| 清理 | 条件 | 用途 |
|---------|-----------|---------|
| `resetMicrocompactState()` | 始终 | 清除过时的工具注册 |
| `resetContextCollapse()` | 仅主线程 | 重置折叠跟踪 |
| `getUserContext.cache.clear()` | 仅主线程 | 强制 CLAUDE.md 重新加载 |
| `resetGetMemoryFilesCache()` | 仅主线程 | 强制记忆文件重新读取 |
| `clearSystemPromptSections()` | 始终 | 强制系统提示重建 |
| `clearClassifierApprovals()` | 始终 | 清除权限分类器缓存 |
| `clearSpeculativeChecks()` | 始终 | 清除推测权限检查 |
| `clearBetaTracingState()` | 始终 | 重置 beta 跟踪 |
| `sweepFileContentCache()` | 启用归因时 | 清除归因文件缓存 |
| `clearSessionMessagesCache()` | 始终 | 清除会话消息缓存 |

**故意不清理**：`sentSkillNames` — 在压缩后重新注入完整 skill_listing（约 4K 令牌）是纯粹的缓存创建浪费。模型仍在其模式中保留 SkillTool，并且 invoked_skills 附件保留使用的技能内容。

### 边界消息

每次压缩都会创建一个具有元数据的 `SystemCompactBoundaryMessage`：

```typescript
{
  type: 'system',
  system: 'compact_boundary',
  compactMetadata: {
    compactType: 'auto' | 'manual',
    preCompactTokenCount: number,
    lastPreCompactUuid: UUID,
    preCompactDiscoveredTools: string[],  // 压缩前发现的工具
    preservedSegment?: {                  // 用于部分压缩
      headUuid: UUID,
      anchorUuid: UUID,
      tailUuid: UUID,
    },
  },
}
```

边界标记使消息加载器能够跨压缩不连续性重建消息链，包括重新链接保留的段。

### 错误处理

| 错误 | 处理 |
|-------|----------|
| 消息不足 | 抛出 `ERROR_MESSAGE_NOT_ENOUGH_MESSAGES` |
| 提示过长 | 用截断重试（最多 3 次），然后抛出 |
| API 错误 | 抛出带 API 错误前缀 |
| 用户中止（ESC） | 抛出 `ERROR_MESSAGE_USER_ABORT`，无通知 |
| 不完整响应 | 抛出 `ERROR_MESSAGE_INCOMPLETE_RESPONSE` |
| 无摘要文本 | 抛出"Failed to generate conversation summary" |
| 自动压缩失败 | 记录，下一回合重试，无用户通知 |
| 手动压缩失败 | 向用户显示错误通知 |
| 会话记忆错误 | 回退到传统压缩 |

### 配置

#### 环境变量

| 变量 | 效果 |
|----------|----------|
| `DISABLE_COMPACT` | 禁用所有压缩 |
| `DISABLE_AUTO_COMPACT` | 仅禁用自动压缩（手动仍可工作） |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 覆盖用于自动压缩计算的上下文窗口 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 覆盖自动压缩阈值作为百分比 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 覆盖阻塞令牌限制 |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | 强制启用会话记忆压缩 |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | 强制禁用会话记忆压缩 |
| `USE_API_CLEAR_TOOL_RESULTS` | 启用服务器端工具结果清除（仅限 ant） |
| `USE_API_CLEAR_TOOL_USES` | 启用服务器端工具使用清除（仅限 ant） |
| `API_MAX_INPUT_TOKENS` | 覆盖用于 API 上下文管理的触发阈值 |
| `API_TARGET_INPUT_TOKENS` | 覆盖用于 API 上下文管理的目标令牌计数 |

#### 令牌常量

| 常量 | 值 | 用途 |
|----------|-------|---------|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | 为压缩输出保留 |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | 上下文窗口下方的缓冲区 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 警告阈值缓冲区 |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | 错误阈值缓冲区 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 阻塞限制缓冲区 |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | 压缩后最大恢复文件数 |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | 总文件恢复预算 |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | 每文件令牌限制 |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | 每技能令牌限制 |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | 总技能恢复预算 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 断路器阈值 |
| `MAX_PTL_RETRIES` | 3 | 提示过长重试限制 |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | 流式传输重试限制（启用时） |
| `IMAGE_MAX_TOKEN_SIZE` | 2,000 | 每个图像/文档的估计令牌 |

#### 会话记忆压缩配置

| 字段 | 默认 | 描述 |
|-------|---------|-------------|
| `minTokens` | 10,000 | 压缩后保留的最少令牌 |
| `minTextBlockMessages` | 5 | 保留的最少文本块消息数 |
| `maxTokens` | 40,000 | 保留的最大令牌（硬上限） |

#### 基于时间的微压缩配置

| 字段 | 默认 | 描述 |
|-------|---------|-------------|
| `enabled` | false | 主开关 |
| `gapThresholdMinutes` | 60 | 当空闲间隙超过此时触发 |
| `keepRecent` | 5 | 保留最近工具结果数 |

### 数据流

```
用户查询 / API 调用
    │
    ▼
microcompactMessages() — 每轮修剪
    ├── 基于时间的触发器（空闲间隙 > 阈值）
    │   └── 清除旧工具结果内容
    └── 缓存 MC 触发器（工具计数 > 阈值）
        └── 为 API 层排队 cache_edits
    │
    ▼
shouldAutoCompact() — 检查令牌阈值
    │
    ├── 低于阈值 → 正常继续
    │
    └── 高于阈值 → autoCompactIfNeeded()
        │
        ├── trySessionMemoryCompaction()
        │   ├── 会话记忆存在且不为空？
        │   │   ├── 计算要保留的消息
        │   │   ├── 从 SM 创建边界 + 摘要
        │   │   └── 返回 CompactionResult
        │   └── 落入完整压缩
        │
        └── compactConversation()
            ├── PreCompact 钩子
            ├── 流式摘要（分叉代理或直接）
            │   ├── PTL 重试：截断最旧的组
            │   └── 流式传输重试（如果启用）
            ├── 清除文件状态
            ├── 生成压缩后附件
            │   ├── 最近文件（按预算约束）
            │   ├── 异步代理
            │   ├── 计划文件
            │   ├── 计划模式
            │   ├── 调用的技能
            │   ├── 延迟工具增量
            │   ├── 代理列表增量
            │   └── MCP 指令增量
            ├── SessionStart 钩子
            ├── 创建边界标记
            ├── PostCompact 钩子
            └── 返回 CompactionResult
    │
    ▼
runPostCompactCleanup()
    ├── 重置微压缩状态
    ├── 清除上下文折叠
    ├── 清除用户上下文缓存
    ├── 清除系统提示部分
    ├── 清除分类器批准
    └── 清除会话消息缓存
    │
    ▼
REPL 用压缩后结果替换消息
```

## 备注

- `stripImagesFromMessages` 处理嵌套在 `tool_result` 内容数组中的图像，而不仅仅是顶级消息内容
- 用于缓存共享的分叉代理路径**不**设置 `maxOutputTokens` — 这样做会限制 `budget_tokens` 并创建使缓存失效的思考配置不匹配
- 带方向 `'up_to'` 的 `partialCompactConversation` 必须从保留消息中剥离旧的压缩边界；`'from'` 保留它们，因为向后扫描仍然有效
- `adjustIndexToPreserveAPIInvariants` 处理两种场景：tool_use/tool_result 对保留以及与保留助手消息共享 message.id 的思考块（流式传输为每个内容块产生单独消息）
- 会话记忆压缩跟踪 `lastSummarizedMessageId` 以了解哪些消息已被摘要；在恢复的会话上此 ID 缺失，系统回退到保留所有消息并使用 SM 作为摘要
- 当截断留下助手优先序列时（API 要求用户优先），预置 `PTL_RETRY_MARKER` 合成用户消息；在后续重试中将其剥离以防止停滞
- 当不需要压缩时，微压缩返回不变的消息 — 调用者不应假设变异
- 缓存 MC 仅限 ant，需要 `CACHED_MICROCOMPACT` 功能标志以及模型对缓存编辑的支持
- API 上下文管理（`apiMicrocompact.ts`）是一种服务器端策略，发送编辑指令给 API 而不是修改客户端消息
- 压缩服务与 LSP 诊断系统集成 — 诊断作为压缩后附件传递，LSP `closeFile()` 集成标记为 TODO
