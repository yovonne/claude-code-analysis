# 主入口点 (main.tsx)

## 目的

Claude Code 的主 CLI 入口点。此文件协调整个应用生命周期：启动分析、CLI 参数解析（使用 Commander）、功能标志解析、认证、MCP 服务器初始化、会话管理（continue/resume/teleport/SSH）以及 Ink React 渲染。它是中央路由中心，根据命令行标志、环境变量和构建时功能切换将请求分发到正确的执行路径。

## 位置
`restored-src/src/main.tsx`

## 文件大小
~4,683 行（~800KB）

---

## 功能部分

### 1. 启动分析与早期初始化（第 1-210 行）

此部分通过顶层副作用在**所有其他导入之前**运行，以最小化启动延迟：

```typescript
// 1. 在重型模块评估之前标记入口点
profileCheckpoint('main_tsx_entry');

// 2. 与剩余导入并行启动 MDM 子进程（plutil/reg query）
startMdmRawRead();

// 3. 与剩余导入并行启动 macOS 钥匙串读取（OAuth + 旧 API 密钥）
//    否则 isRemoteManagedSettingsEligible() 会顺序读取（~65ms）
startKeychainPrefetch();
```

**关键优化：**
- **`profileCheckpoint`** 来自 `startupProfiler.ts`，使用 Node.js `performance.mark()` 标记计时检查点。两种模式：采样的 Statsig 日志记录（100% ant，0.5% 外部）和详细分析（`CLAUDE_CODE_PROFILE_STARTUP=1`）。
- **`startMdmRawRead()`** 提前启动 MDM（移动设备管理）子进程，以便在剩余导入的 ~135ms 内完成。
- **`startKeychainPrefetch()`** 并行启动两个 macOS 钥匙串读取，避免 ~65ms 的顺序惩罚。
- **`ensureKeychainPrefetchCompleted()`** 和 **`ensureMdmSettingsLoaded()`** 在 `preAction` 钩子中的 `init()` 之前等待。

**调试检测**（`isBeingDebugged()`，第 232-263 行）：检查 `--inspect`、`--debug`、`NODE_OPTIONS` inspect 标志和 `inspector.url()`。如果检测到则立即退出，以防止在生产构建中附加调试器。

**早期输入捕获**：`seedEarlyInput()` 捕获启动期间用户的键入（"用户正在输入"窗口），以便在模块加载时不会丢失输入。

### 2. 使用 Commander 的 CLI 设置（第 884-1008 行）

`run()` 函数构建 Commander CLI 程序：

```typescript
const program = new CommanderCommand()
  .configureHelp(createSortedHelpConfig())
  .enablePositionalOptions()
  .name('claude')
  .description('Claude Code - starts an interactive session by default...')
  .argument('[prompt]', 'Your prompt', String)
  .helpOption('-h, --help', 'Display help for command')
```

**全局选项包括：**
- `-d, --debug [filter]` — 带类别过滤的调试模式
- `-p, --print` — 打印响应并退出（无头/SDK 模式）
- `--bare` — 最小模式：跳过钩子、LSP、插件同步、归因、后台预取、钥匙串读取、CLAUDE.md 自动发现
- `--init`、`--init-only`、`--maintenance` — 钩子生命周期触发器
- `--output-format` — `text`、`json` 或 `stream-json`
- `--max-budget-usd`、`--task-budget` — 成本控制
- `--allowed-tools`、`--disallowed-tools`、`--tools` — 工具允许列表
- `--mcp-config` — 从 JSON 文件/字符串加载 MCP 服务器
- `--permission-mode` — 权限模式选择
- `-c, --continue` — 继续最近的对话
- `-r, --resume [value]` — 按会话 ID 或交互式选择器恢复
- `--model`、`--effort`、`--agent`、`--betas` — 模型/会话配置
- `--settings`、`--setting-sources` — 设置文件/源控制
- `--chrome` / `--no-chrome` — Chrome 中的 Claude 集成
- `--plugin-dir` — 从目录加载插件
- `--file` — 在启动时下载文件资源

**排序帮助配置**（第 890-901 行）：选项按长选项名称字母顺序排序以提高可读性。

### 3. 功能标志（第 21-82 行，全文）

功能标志使用 `bun:bundle` 的 `feature()` 进行构建时死代码消除：

| 标志 | 用途 |
|------|------|
| `COORDINATOR_MODE` | 多 agent 协调器模式；条件导入 `coordinatorModeModule` |
| `KAIROS` | 助手模式（Agent SDK）；条件导入 `assistantModule` 和 `kairosGate` |
| `DIRECT_CONNECT` | 会话服务器连接（cc:// URL、`claude server`） |
| `SSH_REMOTE` | SSH 远程执行（`claude ssh <host>`） |
| `LODESTONE` | 深度链接 URI 处理（`--handle-uri`） |
| `BRIDGE_MODE` | 远程控制（`--remote-control` / `--rc`） |
| `TRANSCRIPT_CLASSIFIER` | 自动模式分类器 |
| `PROACTIVE` | 自主主动模式 |
| `KAIROS_BRIEF` | 简短模式（SendUserMessage 工具） |
| `KAIROS_CHANNELS` | MCP 通道通知 |
| `UDS_INBOX` | Unix 域套接字消息传递 |
| `CHICAGO_MCP` | 计算机使用 MCP（macOS） |
| `BG_SESSIONS` | 后台会话 |
| `HARD_FAIL` | 错误时崩溃而不是记录 |
| `AGENT_MEMORY_SNAPSHOT` | Agent 内存持久化 |
| `CCR_MIRROR` | Claude Code Remote 镜像 |
| `WEB_BROWSER_TOOL` | Web 浏览器集成 |
| `UPLOAD_USER_SETTINGS` | 设置同步到云 |
| `TRANSCRIPT_CLASSIFIER` | 自动模式 + 转录分类 |

**条件导入模式：**
```typescript
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null;
```

**构建变体检测：** `"external" === 'ant'` 检查这是内部 Anthropic 构建还是公共外部构建。这限制了仅 ant 的功能（任务、日志/错误命令、回滚、计算机使用等）。

### 4. 命令注册（第 3892-4512 行）

子命令在主命令的动作处理器**之后**注册。在打印模式（`-p`）中，完全跳过子命令注册（~65ms 节省）：

```typescript
const isPrintMode = process.argv.includes('-p') || process.argv.includes('--print');
if (isPrintMode && !isCcUrl) {
  await program.parseAsync(process.argv);
  return program;
}
```

**已注册的子命令组：**

| 命令 | 描述 | 功能门控 |
|---------|-------------|-------------|
| `claude mcp serve` | 启动 MCP 服务器 | 始终 |
| `claude mcp add/remove/list/get` | MCP 服务器管理 | 始终 |
| `claude mcp add-json` | 从 JSON 字符串添加 MCP | 始终 |
| `claude mcp add-from-claude-desktop` | 从 Claude Desktop 导入 | 始终 |
| `claude mcp reset-project-choices` | 重置项目 MCP 批准 | 始终 |
| `claude server` | 启动会话服务器 | `DIRECT_CONNECT` |
| `claude ssh <host> [dir]` | SSH 远程执行 | `SSH_REMOTE` |
| `claude open <cc-url>` | 连接到服务器（无头） | `DIRECT_CONNECT` |
| `claude auth login/status/logout` | 认证管理 | 始终 |
| `claude plugin validate/list/install/uninstall/enable/disable/update` | 插件管理 | 始终 |
| `claude plugin marketplace add/list/remove/update` | 市场管理 | 始终 |
| `claude setup-token` | 长期 auth 令牌设置 | 始终 |
| `claude agents` | 列出配置的 agent | 始终 |
| `claude auto-mode defaults/config/critique` | 自动模式检查 | `TRANSCRIPT_CLASSIFIER` |
| `claude remote-control / rc` | 远程控制桥接 | `BRIDGE_MODE` |
| `claude assistant [sessionId]` | 附加到桥接会话 | `KAIROS` |
| `claude doctor` | 健康检查 | 始终 |
| `claude update / upgrade` | 检查更新 | 始终 |
| `claude install` | 安装原生构建 | 始终 |
| `claude up` | 运行 CLAUDE.md 设置 | 仅 ant |
| `claude rollback` | 回滚版本 | 仅 ant |
| `claude log / error / export` | 对话日志管理 | 仅 ant |
| `claude task create/list/get/update/dir` | 任务管理 | 仅 ant |
| `claude completion <shell>` | Shell 补全脚本 | 仅 ant，隐藏 |

**预动作钩子**（第 907-967 行）：在任何命令执行之前运行：
1. 等待 MDM 和钥匙串预取完成
2. 调用 `init()`（配置、环境变量、优雅关闭、遥测）
3. 设置 `process.title = 'claude'`
4. 附加日志接收器
5. 将 `--plugin-dir` 连接到内联插件
6. 运行迁移
7. 启动远程托管设置和策略限制加载（非阻塞）
8. 上传用户设置（如果 `UPLOAD_USER_SETTINGS` 标志）

### 5. 认证流程

认证在**信任建立后**处理（或在非交互模式下隐式处理）：

**预认证设置：**
- `startKeychainPrefetch()` 在模块加载时触发（第 20 行）
- `ensureKeychainPrefetchCompleted()` 在 preAction 钩子中等待（第 914 行）
- `populateOAuthAccountInfoIfNeeded()` 在 `init()` 期间调用

**入职后 auth 刷新**（第 2279-2297 行）：
```typescript
if (onboardingShown) {
  void refreshRemoteManagedSettings();
  void refreshPolicyLimits();
  resetUserCache();
  refreshGrowthBookAfterAuthChange();
  void import('./bridge/trustedDevice.js').then(m => {
    m.clearTrustedDeviceToken();
    return m.enrollTrustedDevice();
  });
}
```

**组织验证**（第 2299-2305 行）：`validateForceLoginOrg()` 检查活动令牌的 org 是否与托管设置中的 `forceLoginOrgUUID` 匹配。

**会话入口 auth**（第 1309-1330 行）：`getSessionIngressAuthToken()` 为远程会话中的文件下载检索令牌。

### 6. MCP 初始化（第 1413-1630、2380-2430、2691-2808 行）

MCP 初始化分**三个阶段**发生：

**阶段 1 — 配置加载**（提前开始，第 1799-1814 行）：
```typescript
const mcpConfigPromise = (strictMcpConfig || isBareMode()
  ? Promise.resolve({ servers: {} })
  : getClaudeCodeMcpConfigs(dynamicMcpConfig)
).then(result => { ... });
```

**阶段 2 — 动态 MCP 配置解析**（第 1413-1523 行）：
- 将 `--mcp-config` 参数解析为 JSON 字符串或文件路径
- 针对保留名称验证（Chrome 中的 Claude、计算机使用）
- 应用企业策略过滤（`filterMcpServersByPolicy`）
- 向所有配置添加 `scope: 'dynamic'`

**阶段 3 — 资源预取**（第 2404-2430 行）：
```typescript
const localMcpPromise = isNonInteractiveSession
  ? Promise.resolve({ clients: [], tools: [], commands: [] })
  : prefetchAllMcpResources(regularMcpConfigs);
```
- 交互模式：在 REPL 渲染之前连接到所有 MCP 服务器
- 非交互模式：增量推送到 headlessStore
- Claude.ai 连接器与 5 秒超时并行获取

**特殊 MCP 集成：**
- **Chrome 中的 Claude**（第 1525-1577 行）：为浏览器集成设置 MCP 配置、允许的工具和系统提示
- **Chicago MCP / 计算机使用**（第 1597-1630 行）：macOS 独有，GrowthBook 门控的屏幕捕获
- **企业 MCP 配置**（第 1582-1595 行）：当企业配置存在时阻止动态 MCP
- **严格 MCP 配置**（第 1580 行）：`--strict-mcp-config` 忽略所有 MCP 除了 `--mcp-config`

### 7. 渲染管道（第 2213-2242、2926-3036、3760-3807 行）

**Ink 根创建**（第 2226-2229 行）：
```typescript
const { createRoot } = await import('./ink.js');
root = await createRoot(ctx.renderOptions);
```

**设置屏幕**（第 2239-2242 行）：
```typescript
const onboardingShown = await showSetupScreens(
  root, permissionMode, allowDangerouslySkipPermissions,
  commands, enableClaudeInChrome, devChannels
);
```
根据需要显示信任对话框、OAuth 登录、入职和恢复选择器。

**初始 AppState 构建**（第 2926-3036 行）：包含以下内容的全面状态对象：
- 设置、任务、agent 注册、详细模式
- 模型配置、简短模式状态
- MCP 状态（客户端、工具、命令、资源）
- 插件状态（启用、禁用、错误）
- 远程控制/桥接状态（replBridge 字段）
- 通知、征召队列、待办事项
- 文件历史、归因、思考状态
- 提示建议、推测状态
- 来自 CLI 提示的初始用户消息

**REPL 启动**（第 3798-3807 行）：
```typescript
await launchRepl(root, {
  getFpsMetrics, stats, initialState
}, {
  ...sessionConfig,
  initialMessages,
  pendingHookMessages
}, renderAndRun);
```

**`renderAndRun`**（来自 `interactiveHelpers.tsx`）：渲染 React 树，启动延迟预取，等待退出，然后执行优雅关闭。

### 8. 会话管理（第 3101-3807 行）

主动作处理器分支到多个会话路径：

**继续**（`-c`，第 3101-3155 行）：
- 从磁盘加载最近的对话
- 清除会话缓存以进行新的文件/技能发现
- 用归因处理恢复的对话
- 使用恢复的消息和 agent 状态启动 REPL

**直接连接**（`cc://` URL，第 3156-3192 行）：
- 使用服务器 URL 和 auth 令牌创建 `DirectConnectSession`
- 从服务器配置设置工作目录
- 使用直接连接配置启动 REPL（无本地 MCP/工具）

**SSH 远程**（`claude ssh`，第 3193-3258 行）：
- 探测远程主机，必要时部署二进制文件
- 使用 unix-socket auth 隧道创建 SSH 会话
- `--local` 标志跳过探测/部署以进行 e2e 测试
- 使用 `\r` + 擦除进行原地更新的进度输出

**助手模式**（`claude assistant`，第 3259-3354 行）：
- 发现桥接会话或使用提供的会话 ID
- 如果未找到则安装助手（向导 + 守护进程等待）
- 设置 `kairosActive` 和 `userMsgOptIn` 用于简短模式
- 启动 REPL 作为纯查看器客户端

**恢复**（`-r`、`--from-pr`，第 3355-3759 行）：
- 支持 UUID、自定义标题搜索、ccshare URL、文件路径
- `--fork-session` 从恢复点创建新会话
- `--from-pr` 按链接的 PR 过滤会话
- 当没有特定会话时使用带 worktree 支持的交互式选择器
- 通过 `matchedLog` 进行跨-worktree 恢复

**传送**（`--teleport`，第 3504-3579 行）：
- 交互模式：显示任务选择器，检出分支，处理消息
- 直接模式：获取会话，验证仓库，处理仓库不匹配
- 使用进度 UI 组件进行视觉反馈

**远程**（`--remote`，第 3409-3503 行）：
- 通过 CCR（Claude Code Remote）创建远程会话
- TUI 模式检查：除非 `tengu_remote_backend` 门打开，否则需要描述
- 原始行为：打印会话信息并退出
- 新行为：使用 CCR 引擎启动本地 TUI

**新鲜会话**（第 3760-3807 行）：
- 当没有 resume/continue/teleport 标志时的默认路径
- 将未解析的钩子承诺传递给 REPL 以进行非阻塞渲染
- 如果通过协议处理程序启动则显示深度链接横幅
- 持久化模式以供将来恢复

### 9. REPL 启动器（第 2213-2242、3134-3807 行）

REPL 通过 `replLauncher.tsx` 的 `launchRepl()` 启动：

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>
): Promise<void>
```

**启动路径及其配置：**

| 路径 | 初始消息 | 工具 | MCP |
|------|-----------------|-------|-----|
| 新鲜会话 | 钩子消息 + 深度链接横幅 | 完整工具集 | 已连接 |
| 继续 | 恢复的消息 | 完整工具集 | 已连接 |
| 直接连接 | 连接信息消息 | 空 | 无（远程） |
| SSH | SSH 信息消息 | 空 | 无（远程） |
| 助手 | 会话附加消息 | 空 | 无（远程） |
| 恢复 | 恢复的消息 | 完整工具集 | 已连接 |
| 传送 | 传送的消息 | 完整工具集 | 已连接 |
| 远程 | 远程信息 + 用户消息 | 过滤（远程安全） | 无（远程） |

---

## 关键导出

| 导出 | 类型 | 描述 |
|--------|------|-------------|
| `main()` | `async function` | 从 CLI 引导调用的顶层入口点 |
| `startDeferredPrefetches()` | `async function` | 在首次渲染后调用以预热缓存 |

## 依赖

### 内部
- `./bootstrap/state.ts` — 全局可变状态（cwd、会话 ID、客户端类型等）
- `./entrypoints/init.ts` — `init()` 函数（配置、环境变量、遥测、优雅关闭）
- `./setup.ts` — `setup()` 函数（worktree、UDS、git、内存、钩子）
- `./replLauncher.tsx` — 用于 Ink 渲染的 `launchRepl()`
- `./interactiveHelpers.tsx` — `showSetupScreens()`、`exitWithError()`、`renderAndRun()`
- `./commands.ts` — `getCommands()`、`filterCommandsForRemoteMode()`
- `./services/mcp/client.ts` — `getMcpToolsCommandsAndResources()`、`prefetchAllMcpResources()`
- `./services/mcp/config.ts` — `getClaudeCodeMcpConfigs()`、`parseMcpConfig()`、策略过滤
- `./services/analytics/growthbook.ts` — 功能标志初始化和刷新
- `./state/AppStateStore.ts` — `getDefaultAppState()`、`AppState` 类型
- `./state/store.ts` — `createStore()` 用于状态管理
- `./utils/startupProfiler.ts` — `profileCheckpoint()`、`profileReport()`
- `./utils/permissions/permissionSetup.ts` — 工具权限上下文初始化
- `./utils/sessionRestore.ts` — `processResumedConversation()`
- `./utils/teleport.ts` — 传送会话管理
- `./server/createDirectConnectSession.ts` — 直接连接会话
- `./ssh/createSSHSession.ts` — SSH 远程会话
- `./bridge/bridgeEnabled.js` — 远程控制桥接门控
- `./dialogLaunchers.ts` — 恢复选择器、传送包装器、助手安装器

### 外部
- `@commander-js/extra-typings` — typed CLI 参数解析
- `bun:bundle` — 用于构建时功能标志的 `feature()`
- `react` — 使用 Ink 的 UI 渲染
- `chalk` — 终端颜色输出
- `lodash-es` — `mapValues`、`pickBy`、`uniqBy`

## 数据流

```
CLI 调用
    │
    ▼
main.tsx 模块加载
    ├── profileCheckpoint('main_tsx_entry')
    ├── startMdmRawRead()          ← 与导入并行
    ├── startKeychainPrefetch()    ← 与导入并行
    └── 导入所有模块（~135ms）
    │
    ▼
profileCheckpoint('main_tsx_imports_loaded')
    │
    ▼
main() 函数
    ├── 设置 NoDefaultCurrentDirectoryInExePath（Windows 安全）
    ├── 初始化警告处理器
    ├── 解析特殊 URL（cc://、深度链接、ssh、助手）
    ├── 确定 isNonInteractive（-p、--print、--init-only、无 TTY）
    ├── 设置 clientType（cli、sdk、remote、github-action 等）
    ├── eagerLoadSettings() — 解析 --settings 和 --setting-sources
    │
    ▼
run() — Commander 程序设置
    ├── 定义程序、名称、描述、所有全局选项
    ├── 注册 preAction 钩子：
    │   ├── await ensureMdmSettingsLoaded()
    │   ├── await ensureKeychainPrefetchCompleted()
    │   ├── await init() — 配置、环境、遥测、优雅关闭
    │   ├── 设置 process.title
    │   ├── 附加日志接收器
    │   ├── 连接 --plugin-dir
    │   ├── runMigrations()
    │   ├── loadRemoteManagedSettings()（非阻塞）
    │   └── loadPolicyLimits()（非阻塞）
    │
    ▼
    ├── 在打印模式中跳过子命令 → parseAsync → 退出
    │
    ▼
    └── 注册所有子命令（mcp、auth、plugin、doctor 等）
    │
    ▼
    └── program.parseAsync(process.argv)
        │
        ▼
        主动作处理器（默认命令）
            ├── 解析选项（调试、工具、MCP、权限、模型等）
            ├── maybeActivateProactive() / maybeActivateBrief()
            ├── initializeToolPermissionContext()
            ├── 启动 MCP 配置加载（并行）
            ├── setup() — worktree、git、内存、钩子
            ├── 加载命令 + agent 定义（与 setup 并行）
            ├── 计算有效模型、顾问模型、思考配置
            ├── 构建初始 AppState
            │
            ▼
            └── 按会话路径分支：
                ├── --continue     → loadConversationForResume → launchRepl
                ├── cc:// URL      → createDirectConnectSession → launchRepl
                ├── ssh <host>     → createSSHSession → launchRepl
                ├── assistant      → 发现会话 → launchRepl（查看器）
                ├── --resume/-r    → loadConversationForResume → launchRepl
                ├── --teleport     → 获取会话 → teleportWithProgress → launchRepl
                ├── --remote       → teleportToRemote → launchRepl
                └──（默认）      → launchRepl（新会话）
                    │
                    ▼
                    launchRepl(root, appProps, replProps, renderAndRun)
                        ├── 动态导入 App + REPL 组件
                        └── renderAndRun(root, <App><REPL /></App>)
                            ├── root.render(element)
                            ├── startDeferredPrefetches()
                            └── 等待退出 → 优雅关闭
```

## 配置

### 环境变量

| 变量 | 用途 |
|----------|-------------|
| `CLAUDE_CODE_PROFILE_STARTUP` | 启用详细启动分析 |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | 在首次渲染后退出（基准测试） |
| `CLAUDE_CODE_SIMPLE` | 由 `--bare` 设置；禁用钩子、LSP、插件等 |
| `CLAUDE_CODE_ENTRYPOINT` | 标识调用者：`cli`、`sdk-cli`、`sdk-ts`、`sdk-py`、`remote`、`claude-vscode`、`claude-desktop`、`local-agent`、`mcp`、`claude-code-github-action` |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | `bridge` 用于远程控制会话 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | 会话入口 auth 令牌 |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | 远程会话标识符 |
| `CLAUDE_CODE_QUESTION_PREVIEW_FORMAT` | `markdown` 或 `html` |
| `CLAUDE_CODE_AGENT` | Agent 覆盖（由 `--agent` 与 `BG_SESSIONS` 设置） |
| `CLAUDE_CODE_TASK_LIST_ID` | 任务模式的任务列表 ID |
| `CLAUDE_CODE_PROACTIVE` | 启用主动模式 |
| `CLAUDE_CODE_BRIEF` | 启用简短模式 |
| `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` | 云提供商选择 |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` / `CLAUDE_CODE_SKIP_VERTEX_AUTH` | 跳过凭证预取 |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | 跳过设置 process.title |
| `CLAUDE_CODE_REMOTE` | 远程模式（启用所有钩子事件） |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | 启用部分消息输出 |
| `CLAUDE_CODE_COORDINATOR_MODE` | 启用协调器模式工具过滤 |
| `CLAUDE_CODE_TERMINAL_RECORDING` | 启用 asciicast 录制（仅 ant） |
| `CLAUDE_CODE_DISABLE_SESSION_DATA_UPLOAD` | 禁用会话数据上传器 |
| `CLAUDE_CODE_MESSAGING_SOCKET` | UDS 消息传递套接字路径 |
| `NODE_EXTRA_CA_CERTS` | 额外 CA 证书 |
| `CLAUDE_CODE_CLIENT_CERT` | mTLS 客户端证书 |
| `NoDefaultCurrentDirectoryInExePath` | Windows 安全（设置为 `1`） |
| `GITHUB_ACTIONS` | 检测 GitHub Actions 环境 |
| `USER_TYPE` | `ant` 用于内部构建 |

### 迁移系统

当前迁移版本：**11**。迁移在全局配置中的 `migrationVersion` 过期时在启动时同步运行：

1. `migrateAutoUpdatesToSettings` — 将自动更新配置移至设置
2. `migrateBypassPermissionsAcceptedToSettings` — 移动权限接受
3. `migrateEnableAllProjectMcpServersToSettings` — 移动 MCP 服务器批准
4. `resetProToOpusDefault` — 将默认模型从 Pro 更改为 Opus
5. `migrateSonnet1mToSonnet45` — 模型字符串迁移
6. `migrateLegacyOpusToCurrent` — 旧 Opus → 当前模型
7. `migrateSonnet45ToSonnet46` — 模型字符串迁移
8. `migrateOpusToOpus1m` — 模型字符串迁移
9. `migrateReplBridgeEnabledToRemoteControlAtStartup` — 桥接配置迁移
10. `resetAutoModeOptInForDefaultOffer` — 自动模式 opt-in 重置（TRANSCRIPT_CLASSIFIER）
11. `migrateFennecToOpus` — 内部模型迁移（仅 ant）

异步迁移：`migrateChangelogFromConfig`（即发即忘）。

## 相关模块

- [工具系统](./tool-system.md)
- [命令系统](./command-system.md)
- [状态管理](./state-management.md)
- [MCP 系统](./mcp-system.md)
- [会话管理](./session-management.md)
- [认证](./authentication.md)
- [功能标志](./feature-flags.md)
- [启动性能](./startup-performance.md)
