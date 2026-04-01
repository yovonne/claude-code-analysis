# Hooks 工具

## 目的

Hooks 系统提供了一个全面的生命周期事件机制，允许外部命令、提示、HTTP 端点和 TypeScript 函数拦截并响应 Claude Code 事件。此模块系列涵盖 hook 注册、执行、配置管理、SSRF 保护、异步 hook 生命周期和事件广播。

## 位置

核心 hook 执行和管理：
- `restored-src/src/utils/hooks/sessionHooks.ts` — 会话范围的 hook 注册和检索
- `restored-src/src/utils/hooks/hooksConfigManager.ts` — Hook 事件元数据、分组和优先级排序
- `restored-src/src/utils/hooks/hooksSettings.ts` — Hook 源过滤（仅管理、仅插件、全部禁用策略）
- `restored-src/src/utils/hooks/hooksConfigSnapshot.ts` — 基于快照的配置缓存用于会话中 hook 显示
- `restored-src/src/utils/hooks/execAgentHook.ts` — 基于 agent 的 hook 执行（多轮 LLM 查询）
- `restored-src/src/utils/hooks/execHttpHook.ts` — 带 SSRF 保护的 HTTP hook 执行
- `restored-src/src/utils/hooks/execPromptHook.ts` — 基于提示的 hook 执行（单轮 LLM 查询）
- `restored-src/src/utils/hooks/hookHelpers.ts` — 共享工具：参数替换、结构化输出工具、模式
- `restored-src/src/utils/hooks/ssrfGuard.ts` — HTTP hooks 的 DNS 级 SSRF 保护

异步 hook 生命周期：
- `restored-src/src/utils/hooks/AsyncHookRegistry.ts` — 异步（后台）shell 命令 hook 的注册表
- `restored-src/src/utils/hooks/hookEvents.ts` — hook 启动/进度/响应事件的事件广播系统
- `restored-src/src/utils/hooks/postSamplingHooks.ts` — 采样后 hook 注册表（内部，不在设置中暴露）

注册和监视：
- `restored-src/src/utils/hooks/registerFrontmatterHooks.ts` — 从 agent/skill frontmatter 注册 hooks
- `restored-src/src/utils/hooks/registerSkillHooks.ts` — 从 skill frontmatter 注册 hooks（带 once: true 支持）
- `restored-src/src/utils/hooks/fileChangedWatcher.ts` — 用于 CwdChanged/FileChanged hooks 的基于 chokidar 的文件观察者
- `restored-src/src/utils/hooks/skillImprovement.ts` — 通过采样后 LLM 分析自动改进 skill 定义

## 主要导出

### 会话 Hooks（`sessionHooks.ts`）

#### 类型
- `FunctionHook`: 带有嵌入函数的内存中 TypeScript 回调 hook。仅会话范围，不能持久化。
- `FunctionHookCallback`: `(messages, signal?) => boolean | Promise<boolean>` — 返回 true 如果检查通过
- `SessionHooksState`: `Map<string, SessionStore>` — 使用 Map（不是 Record）以便在高并发下进行 O(1) 变更

#### 函数
- `addSessionHook(setAppState, sessionId, event, matcher, hook, onHookSuccess?, skillRoot?)`: 添加命令/提示/agent/http hook 到会话
- `addFunctionHook(setAppState, sessionId, event, matcher, callback, errorMessage, options?)`: 添加函数 hook；返回用于删除的 hook ID
- `removeFunctionHook(setAppState, sessionId, event, hookId)`: 按 ID 删除函数 hook
- `removeSessionHook(setAppState, sessionId, event, hook)`: 从会话中删除特定 hook
- `getSessionHooks(appState, sessionId, event?)`: 获取事件的所有会话 hook（不包括函数 hook）
- `getSessionFunctionHooks(appState, sessionId, event?)`: 单独获取所有会话函数 hook
- `getSessionHookCallback(appState, sessionId, event, matcher, hook)`: 获取完整 hook 条目包括回调
- `clearSessionHooks(setAppState, sessionId)`: 清除会话的所有会话 hook

### Hook 执行

#### Agent Hooks（`execAgentHook.ts`）
- `execAgentHook(hook, hookName, hookEvent, jsonInput, signal, toolUseContext, toolUseID, messages, agentName?)`: 使用多轮 LLM 查询（带 StructuredOutput 工具强制）执行基于 agent 的 hook。最多支持 50 轮，默认使用小型快速模型。

#### HTTP Hooks（`execHttpHook.ts`）
- `execHttpHook(hook, hookEvent, jsonInput, signal?)`: 通过 SSRF 保护将 hook 输入 JSON POST 到配置的 URL，在头中进行环境变量插值、沙箱代理路由和 URL 允许列表强制执行。

#### Prompt Hooks（`execPromptHook.ts`）
- `execPromptHook(hook, hookName, hookEvent, jsonInput, signal, toolUseContext, messages?, toolUseID?)`: 使用单轮 LLM 查询（带 JSON 模式输出强制）执行基于提示的 hook。

### SSRF 保护（`ssrfGuard.ts`）

#### 函数
- `isBlockedAddress(address)`: 如果地址在阻止范围内（私有、链路本地、CGNAT）则返回 true。允许回路（127.0.0.0/8、::1）用于本地开发 hooks。
- `ssrfGuardedLookup(hostname, options, callback)`: dns.lookup 兼容函数，在连接前验证解析的 IP。用作 axios `lookup` 选项。

#### 阻止范围
- IPv4: 0.0.0.0/8、10.0.0.0/8、100.64.0.0/10、169.254.0.0/16、172.16.0.0/12、192.168.0.0/16
- IPv6: ::、fc00::/7、fe80::/10、阻止范围内的 IPv4 映射地址
- 允许：127.0.0.0/8、::1、其他一切

### 异步 Hook 注册表（`AsyncHookRegistry.ts`）

#### 类型
- `PendingAsyncHook`: 跟踪带进程 ID、超时、进度间隔和响应状态的运行中后台 shell 命令 hook

#### 函数
- `registerPendingAsyncHook(...)`: 注册带超时和进度跟踪的新异步 hook
- `getPendingAsyncHooks()`: 获取所有尚未发送响应的待处理 hook
- `checkForAsyncHookResponses()`: 轮询所有待处理 hook 的完成响应；从 stdout 解析 JSON
- `removeDeliveredAsyncHooks(processIds)`: 删除响应已传递的 hook
- `finalizePendingAsyncHooks()`: 终止并结束所有剩余 hook（会话清理）

### Hook 事件（`hookEvents.ts`）

#### 类型
- `HookStartedEvent`、`HookProgressEvent`、`HookResponseEvent`: hook 生命週期的事件形状
- `HookExecutionEvent`: 所有事件类型的联合
- `HookEventHandler`: `(event: HookExecutionEvent) => void`

#### 函数
- `registerHookEventHandler(handler)`: 注册 hook 事件处理程序；立即排空待处理事件
- `emitHookStarted(hookId, hookName, hookEvent)`: 发出 hook 启动事件
- `emitHookProgress(data)`: 发出带 stdout/stderr 的 hook 进度事件
- `startHookProgressInterval(params)`: 开始定期进度轮询；返回停止函数
- `emitHookResponse(data)`: 发出带退出代码和结果的 hook 响应事件
- `setAllHookEventsEnabled(enabled)`: 启用所有 hook 事件类型的发出（超出 SessionStart/Setup）

### Hook 配置管理器（`hooksConfigManager.ts`）

#### 类型
- `HookEventMetadata`: 每个 hook 事件的摘要、描述和 matcher 元数据
- `MatcherMetadata`: 要匹配的字段和允许的值

#### 函数
- `getHookEventMetadata(toolNames)`: 所有 26+ hook 事件的记忆化元数据（PreToolUse、PostToolUse、Stop、SubagentStop 等）
- `groupHooksByEventAndMatcher(appState, toolNames)`: 按事件和 matcher 分组所有 hooks（设置 + 注册 + 插件）
- `getSortedMatchersForEvent(hooksByEventAndMatcher, event)`: 按源优先级对 matcher 排序
- `getHooksForMatcher(hooksByEventAndMatcher, event, matcher)`: 获取特定 event+matcher 的 hooks
- `getMatcherMetadata(event, toolNames)`: 获取特定 hook 事件 matcher 字段的元数据

### Hook 设置（`hooksSettings.ts`）

#### 类型
- `HookSource`: `'userSettings' | 'projectSettings' | 'localSettings' | 'policySettings' | 'pluginHook' | 'sessionHook' | 'builtinHook'`
- `IndividualHookConfig`: 带事件、命令、matcher 和源的规范化 hook 配置

#### 函数
- `isHookEqual(a, b)`: 通过命令/提示内容比较两个 hook（不是超时）
- `getHookDisplayText(hook)`: 获取 hook 的显示文本
- `getAllHooks(appState)`: 从所有允许的源收集 hooks，遵循策略限制
- `getHooksForEvent(appState, event)`: 过滤特定事件的 hooks
- `shouldAllowManagedHooksOnly()`: 检查是否仅应运行管理的 hooks
- `shouldDisableAllHooksIncludingManaged()`: 检查是否禁用所有 hooks（包括管理的）
- `sortMatchersByPriority(matchers, hooksByEventAndMatcher, selectedEvent)`: 按源优先级对 matcher 排序（用户 > 项目 > 本地 > 插件）

### Hook 配置快照（`hooksConfigSnapshot.ts`）

#### 函数
- `captureHooksConfigSnapshot()`: 从允许的源捕获初始 hooks 配置
- `updateHooksConfigSnapshot()`: 刷新快照（首先重置设置缓存）
- `getHooksConfigFromSnapshot()`: 获取当前快照（如果不存在则延迟捕获）
- `resetHooksConfigSnapshot()`: 清除快照（测试）

### 文件更改观察者（`fileChangedWatcher.ts`）

#### 函数
- `initializeFileChangedWatcher(cwd)`: 为 CwdChanged/FileChanged hooks 初始化 chokidar 观察者
- `updateWatchPaths(paths)`: 从 hook 输出动态更新监视路径
- `onCwdChangedForHooks(oldCwd, newCwd)`: 处理工作目录更改；重新解析路径并重新执行 CwdChanged hooks
- `setEnvHookNotifier(cb)`: 设置环境 hook 通知的回调
- `resetFileChangedWatcherForTesting()`: 重置观察者状态

### Skill 改进（`skillImprovement.ts`）

#### 类型
- `SkillUpdate`: `{ section, change, reason }` — 对 skill 定义的建议改进

#### 函数
- `initSkillImprovement()`: 注册 skill 改进采样后 hook（由特性标志门控）
- `applySkillImprovement(skillName, updates)`: 通过 LLM 重写应用对 skill 文件的建议改进（fire-and-forget）

### 注册辅助函数

- `registerFrontmatterHooks(setAppState, sessionId, hooks, sourceName, isAgent?)`: 从 agent/skill frontmatter 注册 hooks；为 agents 转换 Stop→SubagentStop
- `registerSkillHooks(setAppState, sessionId, hooks, skillName, skillRoot?)`: 注册带 `once: true` 自动删除支持的 skill hooks

### Hook 辅助函数（`hookHelpers.ts`）

- `hookResponseSchema`: hook 响应的 Zod 模式（`{ ok: boolean, reason?: string }`）
- `addArgumentsToPrompt(prompt, jsonInput)`: 在 hook 提示中替换 `$ARGUMENTS` 占位符
- `createStructuredOutputTool()`: 为 hook 响应创建 StructuredOutput 工具
- `registerStructuredOutputEnforcement(setAppState, sessionId)`: 注册强制 StructuredOutput 工具使用的 Stop hook

### 采样后 Hooks（`postSamplingHooks.ts`）

- `registerPostSamplingHook(hook)`: 注册内部采样后 hook（不在设置中暴露）
- `executePostSamplingHooks(messages, systemPrompt, userContext, systemContext, toolUseContext, querySource?)`: 执行所有注册的 hooks
- `clearPostSamplingHooks()`: 清除所有 hooks（测试）

## Hook 事件类型

系统支持 26+ 事件类型，包括：PreToolUse、PostToolUse、PostToolUseFailure、PermissionDenied、Notification、UserPromptSubmit、SessionStart、SessionEnd、Stop、StopFailure、SubagentStart、SubagentStop、PreCompact、PostCompact、PermissionRequest、Setup、TeammateIdle、TaskCreated、TaskCompleted、Elicitation、ElicitationResult、ConfigChange、InstructionsLoaded、WorktreeCreate、WorktreeRemove、CwdChanged、FileChanged。

## 设计注意事项

- 会话 hooks 使用 `Map` 而不是 `Record` 以便在高并发下进行 O(1) 变更（并行 agents 在一个 tick 中注册 hooks）
- HTTP hooks 有全面的 SSRF 保护：URL 允许列表、DNS 级 IP 验证（阻止私有/链路本地范围）、头插值的环境变量允许列表和沙箱代理路由
- Agent hooks 作为多轮 LLM 查询运行，带 StructuredOutput 强制和 50 轮限制
- 异步 hooks（后台 shell 命令）在全局注册表中跟踪，带进度轮询和超时处理
- SSRF 保护故意允许回路（127.0.0.0/8、::1）— 本地开发策略服务器是主要的 HTTP hook 用例
- Skill 改进每 5 条用户消息作为采样后 hook 运行一次，分析对话以寻找应融入 skill 定义的首选项
