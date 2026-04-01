# 类型系统

## 目的

构成应用程序类型安全基础的 TypeScript 类型定义。包括品牌化 ID、命令类型、hook 类型、日志/消息类型、权限类型、插件类型和文本输入类型。`generated/` 子目录包含 protobuf 生成的类型。

## 位置

`restored-src/src/types/`

## 关键文件

### `ids.ts` — 品牌化 ID 类型

使用 TypeScript 的品牌化类型模式防止在编译时意外混合会话 ID 和代理 ID。

**类型:**
- `SessionId`: `string & { readonly __brand: 'SessionId' }`
- `AgentId`: `string & { readonly __brand: 'AgentId' }`

**函数:**
- `asSessionId(id)`: 从 string 到 SessionId 的不安全转换
- `asAgentId(id)`: 从 string 到 AgentId 的不安全转换
- `toAgentId(s)`: 根据 `AGENT_ID_PATTERN` (`/^a(?:.+-)?[0-9a-f]{16}$/`) 验证 — 返回 `AgentId | null`

### `command.ts` — 命令类型

定义命令系统的类型层次结构。

**命令类型:**
- `PromptCommand`: 发送给模型的提示（类似技能）
- `LocalCommand`: 延迟加载的原生命令实现
- `LocalJSXCommand`: 延迟加载的 JSX 渲染命令（交互式对话框）
- `Command`: 所有三种与 `CommandBase` 的联合

**命令基属性:**
- `availability`: 命令可见的认证/提供者环境（`'claude-ai' | 'console'`）
- `isEnabled`: 动态启用检查（功能标志、环境检查）
- `isHidden`: 从类型提示/帮助中隐藏
- `isSensitive`: 参数从对话历史中删除
- `immediate`: 无需等待停止点即可执行
- `kind`: `'workflow'` 用于工作流支持的命令

**上下文类型:**
- `LocalJSXCommandContext`: 扩展 ToolUseContext，带 `canUseTool`、`setMessages`、`onChangeAPIKey`、`resume` 等
- `LocalJSXCommandOnDone`: 命令完成时的回调（带 display、shouldQuery、metaMessages 选项）
- `ResumeEntrypoint`: 会话如何恢复（`cli_flag`、`slash_command_picker` 等）

### `hooks.ts` — Hook 类型

Hook 系统的类型，包括回调 hooks、权限 hooks 和 hook 结果。

**关键类型:**
- `HookCallback`: SDK 回调 hook，带 `(input, toolUseID, abort, hookIndex, context) => Promise<HookJSONOutput>`
- `HookCallbackMatcher`: `{ matcher?, hooks[], pluginName? }` — 将事件匹配到回调
- `HookCallbackContext`: `{ getAppState, updateAttributionState }` — 回调的状态访问
- `HookProgress`: 运行中的 hooks 进度指示器
- `HookBlockingError`: Hook 执行中的阻塞错误
- `PermissionRequestResult`: PermissionRequest hook 的允许/拒绝决策
- `HookResult`: 单个 hook 执行的聚合结果
- `AggregatedHookResult`: 多个 hooks 的组合结果

**Hook 响应 schemas:**
- `promptRequestSchema` / `PromptRequest`: 提示获取协议
- `syncHookResponseSchema`: 带事件特定 `hookSpecificOutput` 的同步 hook 响应
- `hookJSONOutputSchema`: 异步和同步 hook 响应的联合

**`hookSpecificOutput` 覆盖的 Hook 事件:**
`PreToolUse`、`UserPromptSubmit`、`SessionStart`、`Setup`、`SubagentStart`、`PostToolUse`、`PostToolUseFailure`、`PermissionDenied`、`Notification`、`PermissionRequest`、`Elicitation`、`ElicitationResult`、`CwdChanged`、`FileChanged`、`WorktreeCreate`

### `logs.ts` — 日志和消息类型

定义用于会话持久化和恢复的 transcript/log 条目类型。

**核心类型:**
- `SerializedMessage`: 带会话元数据的消息（cwd、userType、sessionId、timestamp、version）
- `LogOption`: 带摘要、标签、PR 链接、文件历史、归属和上下文折叠数据的完整日志条目
- `TranscriptMessage`: 带父 UUID 和侧链信息的序列化消息

**专门的消息类型（由 `type` 区分）:**
- `SummaryMessage`: AI 生成的会话摘要
- `CustomTitleMessage`: 用户设置的自定义标题
- `AiTitleMessage`: AI 生成的标题（与自定义不同）
- `LastPromptMessage`: 用于 `claude ps` 的最后用户提示
- `TaskSummaryMessage`: 定期任务摘要（每 min(5 步, 2min)）
- `TagMessage`: 会话标签
- `AgentNameMessage` / `AgentColorMessage` / `AgentSettingMessage`: 代理元数据
- `PRLinkMessage`: GitHub PR 链接
- `ModeEntry`: 会话模式（coordinator/normal）
- `WorktreeStateEntry`: 用于恢复的工作树会话状态
- `ContentReplacementEntry`: 用于提示缓存的内容块替换记录
- `FileHistorySnapshotMessage`: 文件状态快照
- `AttributionSnapshotMessage`: 每个文件字符贡献跟踪
- `SpeculationAcceptMessage`: 推测接受记录
- `ContextCollapseCommitEntry`: 持久化的上下文折叠提交（`type: 'marble-origami-commit'`）
- `ContextCollapseSnapshotEntry`: 暂存队列快照（`type: 'marble-origami-snapshot'`）

**条目联合:** 所有消息类型联合为 `Entry`

### `permissions.ts` — 权限类型

纯权限类型定义（无运行时依赖），提取以打破导入循环。

**权限模式:**
- 外部: `'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'`
- 内部: 外部 + `'auto' | 'bubble'`
- 运行时集: 外部 + `'auto'`（取决于 `TRANSCRIPT_CLASSIFIER` 标志）

**权限行为:** `'allow' | 'deny' | 'ask'`

**权限规则:**
- `PermissionRule`: `{ source, ruleBehavior, ruleValue }`
- `PermissionRuleSource`: 规则来源（userSettings、projectSettings、localSettings、flagSettings、policySettings、cliArg、command、session）
- `PermissionRuleValue`: `{ toolName, ruleContent? }`

**权限更新:**
- `PermissionUpdate`: 区分联合 — `addRules | replaceRules | removeRules | setMode | addDirectories | removeDirectories`
- `PermissionUpdateDestination`: 持久化位置（userSettings、projectSettings、localSettings、session、cliArg）

**权限决策:**
- `PermissionAllowDecision`: `{ behavior: 'allow', updatedInput?, userModified?, decisionReason?, ... }`
- `PermissionAskDecision`: `{ behavior: 'ask', message, suggestions?, pendingClassifierCheck?, ... }`
- `PermissionDenyDecision`: `{ behavior: 'deny', message, decisionReason }`
- `PermissionDecision`: allow/ask/deny 的联合
- `PermissionResult`: 决策 + `passthrough` 变体

**决策原因（区分联合）:**
`rule`、`mode`、`subcommandResults`、`permissionPromptTool`、`hook`、`asyncAgent`、`sandboxOverride`、`classifier`、`workingDir`、`safetyCheck`、`other`

**分类器类型:**
- `ClassifierResult`: `{ matches, matchedDescription?, confidence, reason }`
- `YoloClassifierResult`: 带思考、使用、持续时间、请求 ID、提示长度和阶段分解的详细两阶段分类器结果

**工具权限上下文:**
- `ToolPermissionContext`: 模式、规则（始终允许/始终拒绝/始终按来源询问）、工作目录和旁路标志的不可变快照

### `plugin.ts` — 插件类型

插件系统的类型，包括定义、已加载插件和错误。

**关键类型:**
- `BuiltinPluginDefinition`: 带名称、描述、技能、hooks、MCP 服务器的内置插件
- `PluginRepository`: Git 仓库信息（url、branch、commitSha）
- `LoadedPlugin`: 带所有组件路径和配置的完全加载插件
- `PluginComponent`: `'commands' | 'agents' | 'skills' | 'hooks' | 'output-styles'`

**PluginError:** 25+ 错误类型的区分联合：
- 加载: `path-not-found`、`git-auth-failed`、`git-timeout`、`network-error`
- 清单: `manifest-parse-error`、`manifest-validation-error`
- 市场: `plugin-not-found`、`marketplace-not-found`、`marketplace-load-failed`、`marketplace-blocked-by-policy`
- MCP: `mcp-config-invalid`、`mcp-server-suppressed-duplicate`、`mcpb-download-failed`、`mcpb-extract-failed`、`mcpb-invalid-manifest`
- LSP: `lsp-config-invalid`、`lsp-server-start-failed`、`lsp-server-crashed`、`lsp-request-timeout`、`lsp-request-failed`
- Hooks: `hook-load-failed`
- 组件: `component-load-failed`
- 依赖: `dependency-unsatisfied`
- 缓存: `plugin-cache-miss`
- 通用: `generic-error`

**帮助器:**
- `getPluginErrorMessage(error)`: 为任何 PluginError 类型返回人类可读的错误消息

### `textInputTypes.ts` — 文本输入类型

终端文本输入系统的类型，包括 vim 模式和命令队列。

**关键类型:**
- `BaseTextInputProps`: 文本输入的 30+ 属性（value、onChange、onSubmit、cursor、highlights、ghost text、paste 处理等）
- `VimTextInputProps`: 扩展基础支持 vim 模式
- `VimMode`: `'INSERT' | 'NORMAL'`
- `TextInputState` / `VimInputState`: 带光标位置、视口、粘贴状态的输入状态
- `PromptInputMode`: `'bash' | 'prompt' | 'orphaned-permission' | 'task-notification'`
- `QueuePriority`: `'now' | 'next' | 'later'` — 命令队列的中断级别
- `QueuedCommand`: 带值、模式、优先级、粘贴内容、来源、工作负载、agentId 的命令
- `OrphanedPermission`: 无活动工具上下文的权限结果

**帮助器:**
- `isValidImagePaste(c)`: 非空图像粘贴的类型守卫
- `getImagePasteIds(pastedContents)`: 从命令中提取图像粘贴 ID

### `generated/` — 生成的类型

按命名空间组织的 Protobuf 生成类型：

- `events_mono/claude_code/` — Claude Code 事件类型
- `events_mono/common/` — 通用事件类型
- `events_mono/growthbook/` — GrowthBook 功能标志类型
- `events_mono/protobuf/` — Protobuf 消息类型
- `google/` — Google protobuf 已知类型

## 依赖

### 内部依赖

- `Tool.js` — ToolUseContext 引用
- `state/AppState.js` — AppState 引用
- `utils/permissions/*` — 权限规则/行为 schemas
- `entrypoints/agentSdkTypes.js` — Hook 事件、权限类型
- `ink.js` — Key 类型

### 外部依赖

- `@anthropic-ai/sdk` — ContentBlockParam 类型
- `crypto` — UUID 类型
- `type-fest` — 用于编译时断言的 `IsEqual`
- `bun:bundle` — 功能标志检查

## 实现细节

### 品牌化类型

`SessionId` 和 `AgentId` 使用通过与品牌对象交叉的 TypeScript 名义类型。这防止在期望 SessionId 的地方传递原始字符串，而无需显式转换。

### 区分联合

大多数复杂类型使用带 `type` 或 `behavior` 字段的区分联合进行穷尽类型检查：
- `PermissionDecision`: 由 `behavior` 区分
- `PermissionUpdate`: 由 `type` 区分
- `PermissionDecisionReason`: 由 `type` 区分
- `PluginError`: 由 `type` 区分
- `Entry`（日志消息）: 由 `type` 区分

### 编译时断言

`types/hooks.ts` 使用 `type-fest.IsEqual` 在编译时断言 SDK 类型与 Zod schema 类型匹配：
```typescript
type _assertSDKTypesMatch = Assert<IsEqual<SchemaHookJSONOutput, HookJSONOutput>>
```

## 相关模块

- [Schemas](./schemas.md) — Zod schemas 根据这些类型验证
- [引导](./bootstrap.md) — 使用在各处使用的 SessionId 和 AgentId 品牌类型
- [Hooks 系统](./hooks-system.md) — 消费 hook 和权限类型
