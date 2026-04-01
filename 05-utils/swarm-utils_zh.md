# Swarm 工具

## 目的

Swarm 工具模块为 Claude Code 的 agent 团队（swarm）特性提供完整的基础设施。它处理团队配置、队友生成（通过 tmux/iTerm2 的基于进程和进程内）、权限同步、重连、布局管理和不同终端环境的后端检测。

## 位置

- `restored-src/src/utils/swarm/constants.ts` — Swarm 常量和环境变量
- `restored-src/src/utils/swarm/teamHelpers.ts` — 团队文件 CRUD、成员管理、清理
- `restored-src/src/utils/swarm/teammateModel.ts` — 队友的默认模型回退
- `restored-src/src/utils/swarm/teammateInit.ts` — 队友 hook 初始化
- `restored-src/src/utils/swarm/teammateLayoutManager.ts` — 颜色分配、窗格创建、后端委托
- `restored-src/src/utils/swarm/teammatePromptAddendum.ts` — 队友的系统提示附录
- `restored-src/src/utils/swarm/spawnUtils.ts` — 生成队友的 CLI 标志和环境变量继承
- `restored-src/src/utils/swarm/spawnInProcess.ts` — 进程内队友生成
- `restored-src/src/utils/swarm/inProcessRunner.ts` — 进程内队友执行循环
- `restored-src/src/utils/swarm/reconnection.ts` — 队友的会话重连
- `restored-src/src/utils/swarm/permissionSync.ts` — 权限请求/响应同步
- `restored-src/src/utils/swarm/leaderPermissionBridge.ts` — REPL 和进程内队友之间的桥接
- `restored-src/src/utils/swarm/It2SetupPrompt.tsx` — iTerm2 设置 UI 组件

### 后端子模块（`backends/`）
- `restored-src/src/utils/swarm/backends/types.ts` — 后端类型定义
- `restored-src/src/utils/swarm/backends/detection.ts` — 后端环境检测
- `restored-src/src/utils/swarm/backends/registry.ts` — 后端注册表
- `restored-src/src/utils/swarm/backends/ITermBackend.ts` — iTerm2 窗格后端
- `restored-src/src/utils/swarm/backends/TmuxBackend.ts` — tmux 窗格后端
- `restored-src/src/utils/swarm/backends/InProcessBackend.ts` — 进程内后端
- `restored-src/src/utils/swarm/backends/PaneBackendExecutor.ts` — 窗格执行
- `restored-src/src/utils/swarm/backends/it2Setup.ts` — iTerm2 CLI 设置
- `restored-src/src/utils/swarm/backends/teammateModeSnapshot.ts` — 模式快照

## 主要导出

### 常量（`constants.ts`）
- `TEAM_LEAD_NAME`（`'team-lead'`）、`SWARM_SESSION_NAME`、`SWARM_VIEW_WINDOW_NAME`、`TMUX_COMMAND`、`HIDDEN_SESSION_NAME`
- `getSwarmSocketName()`: 为外部 swarm 会话生成唯一套接字名称和 PID
- `TEAMMATE_COMMAND_ENV_VAR`、`TEAMMATE_COLOR_ENV_VAR`、`PLAN_MODE_REQUIRED_ENV_VAR`: 环境变量名称

### 团队辅助函数（`teamHelpers.ts`）

#### 类型
- `TeamFile`: 完整团队配置，包括成员、隐藏窗格、允许路径
- `TeamAllowedPath`: 带工具、agent 和时间戳的目录权限规则
- `SpawnTeamOutput`、`CleanupOutput`: 操作结果类型

#### 函数
- `sanitizeName(name)`: 为 tmux/文件路径清理名称（字母数字 + 短横线）
- `sanitizeAgentName(name)`: 清理 agent 名称（将 @ 替换为 -）
- `getTeamDir(teamName)`、`getTeamFilePath(teamName)`: 团队目录路径辅助函数
- `readTeamFile(teamName)`、`readTeamFileAsync(teamName)`: 读取团队配置（同步/异步）
- `writeTeamFile(teamName, teamFile)`、`writeTeamFileAsync(teamName, teamFile)`: 写团队配置
- `removeTeammateFromTeamFile(teamName, identifier)`: 按 agentId 或名称移除队友
- `addHiddenPaneId(teamName, paneId)`、`removeHiddenPaneId(teamName, paneId)`: 管理隐藏窗格
- `removeMemberFromTeam(teamName, tmuxPaneId)`、`removeMemberByAgentId(teamName, agentId)`: 移除成员
- `setMemberMode(teamName, memberName, mode)`、`setMultipleMemberModes(teamName, modeUpdates)`: 更新权限模式
- `syncTeammateMode(mode, teamNameOverride?)`: 将当前队友的模式同步到配置
- `setMemberActive(teamName, memberName, isActive)`: 更新活动/空闲状态
- `registerTeamForSessionCleanup(teamName)`、`unregisterTeamForSessionCleanup(teamName)`: 会话生命周期跟踪
- `cleanupSessionTeams()`: 退出时清理孤立的团队
- `cleanupTeamDirectories(teamName)`: 移除团队和任务目录，销毁 git worktree
- `destroyWorktree(worktreePath)`: 安全移除 git worktree

### 生成工具（`spawnUtils.ts`）
- `getTeammateCommand()`: 获取用于生成队友的命令（环境覆盖或当前可执行文件）
- `buildInheritedCliFlags(options?)`: 构建要传播到队友的 CLI 标志（权限、模型、设置、插件、模式、chrome）
- `buildInheritedEnvVars()`: 构建要转发的环境变量（API 提供商、代理、配置目录、CCR 标记）

### 进程内生成（`spawnInProcess.ts`）
- `spawnInProcessTeammate(config, context)`: 创建并注册带 AsyncLocalStorage 上下文隔离的进程内队友任务
- `killInProcessTeammate(taskId, setAppState)`: 中止并清理进程内队友

### 进程内运行器（`inProcessRunner.ts`）
- `runInProcessTeammate(config)`: 进程内队友的主要执行循环。用 AsyncLocalStorage 隔离、进度跟踪、基于邮箱的通信、计划模式支持和空闲通知包装 `runAgent()`。
- `createInProcessCanUseTool(identity, abortController, onPermissionWaitMs?)`: 创建一个通过 leader 的 ToolUseConfirm 对话框解析权限的 `canUseTool` 函数（带 worker 徽章）或回退到邮箱轮询。

### 权限同步（`permissionSync.ts`）
- `SwarmPermissionRequestSchema`: 权限请求的 Zod 模式
- `createPermissionRequest(params)`: 创建新的权限请求
- `writePermissionRequest(request)`: 通过文件锁定将请求写入待处理目录
- `readPendingPermissions(teamName?)`: 读取 leader 的所有待处理请求
- `readResolvedPermission(requestId, teamName?)`: 检查请求是否已解决
- `resolvePermission(requestId, resolution, teamName?)`: 解决请求（待处理 → 已解决）
- `cleanupOldResolutions(teamName?, maxAgeMs?)`: 清理旧的已解决文件
- `pollForResponse(requestId, agentName?, teamName?)`: worker 侧轮询响应
- `isTeamLeader(teamName?)`、`isSwarmWorker()`: 角色检测
- `sendPermissionRequestViaMailbox(request)`: 通过邮箱发送请求（新方法）
- `sendPermissionResponseViaMailbox(workerName, resolution, requestId, teamName?)`: 通过邮箱发送响应
- `sendSandboxPermissionRequestViaMailbox(host, requestId, teamName?)`: 沙箱网络访问请求
- `sendSandboxPermissionResponseViaMailbox(workerName, requestId, host, allow, teamName?)`: 沙箱网络访问响应

### Leader 权限桥（`leaderPermissionBridge.ts`）
- `registerLeaderToolUseConfirmQueue(setter)`、`getLeaderToolUseConfirmQueue()`: REPL 队列设置器的桥接
- `registerLeaderSetToolPermissionContext(setter)`、`getLeaderSetToolPermissionContext()`: 权限上下文设置器的桥接

### 队友布局管理器（`teammateLayoutManager.ts`）
- `assignTeammateColor(teammateId)`: 分配唯一颜色（循环）
- `getTeammateColor(teammateId)`: 获取分配的颜色
- `clearTeammateColors()`: 重置所有颜色分配
- `isInsideTmux()`: 检查是否在 tmux 内运行
- `createTeammatePaneInSwarmView(teammateName, teammateColor)`: 使用检测到的后端创建窗格
- `enablePaneBorderStatus(windowTarget?, useSwarmSocket?)`: 启用窗格标题
- `sendCommandToPane(paneId, command, useSwarmSocket?)`: 发送命令到特定窗格

### 重连（`reconnection.ts`）
- `computeInitialTeamContext()`: 从 CLI 参数计算 AppState 的 teamContext（同步，在首次渲染之前）
- `initializeTeammateContextFromSession(setAppState, teamName, agentName)`: 从恢复的会话初始化上下文

### 队友初始化（`teammateInit.ts`）
- `initializeTeammateHooks(setAppState, sessionId, teamInfo)`: 注册空闲通知的 Stop hook 并应用团队范围的允许路径

### 队友提示附录（`teammatePromptAddendum.ts`）
- `TEAMMATE_SYSTEM_PROMPT_ADDendum`: 追加到队友的系统提示文本，解释通过 SendMessage 工具进行通信

### 队友模型（`teammateModel.ts`）
- `getHardcodedTeammateModelFallback()`: 返回带提供商感知的默认模型（Opus 4.6）

### iTerm2 设置提示（`It2SetupPrompt.tsx`）
- `It2SetupPrompt`: React 组件，引导用户完成 iTerm2 `it2` CLI 安装，步骤：安装 → API 说明 → 验证 → 成功/失败

## 依赖

- `@anthropic-ai/sandbox-runtime` — 沙箱运行时（适配器）
- `chokidar`、`zod/v4`、`lodash-es` — 第三方库
- `../teammate.js`、`../teammateMailbox.js`、`../teammateContext.js` — 队友核心工具
- `../permissions/` — 权限系统
- `../telemetry/perfettoTracing.js` — 性能追踪
- `../../tasks/` — 任务系统

## 设计注意事项

- **双重生成**：队友可以作为单独进程（tmux/iTerm2 窗格）或进程内（AsyncLocalStorage 隔离）运行。进程内队友共享相同进程但具有独立身份、中止控制器和对话上下文。
- **邮箱通信**：队友通过基于文件的邮箱进行通信。leader 轮询消息；队友写入 leader 的邮箱并轮询自己的邮箱以获取响应。
- **权限转发**：当队友需要工具使用的权限时，它将请求转发给 leader，leader 显示 UI。存在两种机制：基于文件（待处理/已解决目录带锁定）和基于邮箱（更新的，替换基于文件）。
- **后端抽象**：后端系统（tmux、iTerm2、进程内）通过自动检测在注册表后抽象，允许相同的高级代码在不同终端环境中工作。
- **会话清理**：在会话期间创建的团队在退出时跟踪和清理，包括杀死孤立窗格、移除目录和销毁 git worktree。
- **颜色分配**：队友从调色板以循环顺序获得唯一颜色，以便在 UI 中视觉区分。
