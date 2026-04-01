# 架构概述

## 系统概要

Claude Code CLI 是一个基于终端的 AI 编程助手，提供由 Anthropic 的 Claude API 驱动的交互式 REPL（读取-求值-打印循环）界面。该应用将丰富的终端 UI（使用 Ink/React 构建）与复杂的工具使用系统相结合，允许 AI 模型在用户环境中读取、编辑和执行代码。

系统支持两种主要模式：
- **交互模式**：完整 TUI，配备实时流式响应、权限提示、任务面板和可视化反馈
- **非交互/SDK 模式**（`-p/--print`）：无头模式，通过 stdin/stdout 进行程序化访问，支持结构化 JSON 流式输出

核心功能包括：
- 文件操作（读取、编辑、写入）配合安全控制
- 带沙箱和权限管理的 shell 命令执行
- 子 agent 生成用于并行任务执行（Agent 群/队友）
- MCP（Model Context Protocol）服务器集成实现可扩展工具
- 技能系统提供领域特定能力
- 第三方扩展的插件系统
- 会话持久化和恢复
- 上下文管理（压缩、折叠、裁剪）用于长对话
- 通过桥接进行远程控制（claude.ai Web 界面集成）
- 多种认证提供商（OAuth、API 密钥、Bedrock、Vertex）

## 入口点

代码库有多个入口点，每个服务不同的部署场景：

### 主要入口点

| 入口点 | 文件 | 用途 |
|---|---|---|
| **CLI 引导** | `src/entrypoints/cli.tsx` | 主 CLI 入口点。在进入完整 CLI 之前处理特殊命令（`--version`、`daemon`、`bridge`、`bg`、`templates` 等）的快速路径分发。所有导入都是动态的，以最小化模块评估。 |
| **MCP 服务器** | `src/entrypoints/mcp.ts` | 将 Claude Code 作为 MCP 服务器运行，以与其他 MCP 客户端集成。 |
| **SDK 类型** | `src/entrypoints/agentSdkTypes.ts` | Agent SDK 公共 API 的类型定义和存根函数。实际的 SDK 实现位于单独的文件中；此文件提供类型表面。 |

### 快速路径分发（cli.tsx）

CLI 引导实现了一个快速路径路由模式，避免为简单命令加载完整应用程序：

- `--version/-v/-V`：零导入版本输出
- `--dump-system-prompt`：系统提示提取（仅 ant）
- `--claude-in-chrome-mcp`：Chrome 集成 MCP 服务器
- `--daemon-worker=<kind>`：守护进程工作进程（内部）
- `remote-control/rc/remote/sync/bridge`：用于远程控制的桥接模式
- `daemon`：长期运行的监督进程
- `ps/logs/attach/kill/--bg`：后台会话管理
- `new/list/reply`：模板作业命令
- `environment-runner`：BYOC（Bring Your Own Compute）运行器
- `self-hosted-runner`：面向 API 的自托管运行器
- `--worktree --tmux`：快速 tmux worktree 执行

### 次要入口点

| 文件 | 用途 |
|---|---|
| `src/entrypoints/init.ts` | 初始化逻辑：配置验证、环境设置、遥测、优雅关闭注册。已缓存以防止重复初始化。 |

## 核心架构层

架构遵循分层设计，关注点分离清晰：

```
┌─────────────────────────────────────────────────────────────────┐
│  入口点 (cli.tsx, mcp.ts)                                       │
│  快速路径分发、动态导入                                          │
├─────────────────────────────────────────────────────────────────┤
│  CLI 层 (main.tsx)                                              │
│  Commander.js CLI 解析、选项处理、模式检测                       │
├─────────────────────────────────────────────────────────────────┤
│  设置层 (setup.ts)                                              │
│  环境设置、worktree、钩子、终端备份、UDS                         │
├─────────────────────────────────────────────────────────────────┤
│  交互层 (REPL.tsx + Ink)                                        │
│  终端 UI、消息渲染、输入处理、焦点管理                           │
├─────────────────────────────────────────────────────────────────┤
│  查询引擎层 (QueryEngine.ts, query.ts)                           │
│  API 交互、流式处理、工具执行、上下文管理                         │
├─────────────────────────────────────────────────────────────────┤
│  工具层 (Tool.ts, tools.ts, tools/*/)                           │
│  工具定义、权限检查、执行编排                                    │
├─────────────────────────────────────────────────────────────────┤
│  服务层 (services/*)                                            │
│  API 客户端、MCP、分析、压缩、LSP、插件、技能                    │
├─────────────────────────────────────────────────────────────────┤
│  状态层 (state/*, bootstrap/state.ts)                           │
│  AppState 存储、全局会话状态、引导状态                           │
├─────────────────────────────────────────────────────────────────┤
│  工具层 (utils/*)                                               │
│  共享辅助函数、git、认证、设置、权限等                           │
└─────────────────────────────────────────────────────────────────┘
```

### 层描述

**1. 入口点层**
- 带有动态导入的最小引导代码
- 为简单命令实现快速路径路由，避免加载完整应用程序
- 针对不同部署模式的功能门控代码路径

**2. CLI 层（`main.tsx`）**
- 基于 Commander.js 构建的参数解析
- 处理所有 CLI 标志、选项和子命令
- 确定交互式与非交互式模式
- 编排启动序列：设置加载 → 权限上下文 → 设置 → REPL/无头执行
- 大量功能标志门控用于条件导入（死代码消除）

**3. 设置层（`setup.ts`）**
- 信任建立后的环境初始化
- Worktree 创建和 tmux 会话管理
- 钩子系统初始化（Setup、SessionStart 钩子）
- 终端备份/恢复（iTerm2、Terminal.app）
- UDS（Unix Domain Socket）消息服务器启动
- 后台任务注册（归因钩子、团队内存监视器）

**4. 交互层（REPL + Ink）**
- 使用 Ink 框架的基于 React 的终端 UI
- 自定义 Ink 包装器（`src/ink.ts`）集成主题提供者
- 消息渲染、输入处理、焦点管理
- 终端视口跟踪、动画帧
- 组件包括：消息、工具、任务、队友、权限、模态框、叠加层

**5. 查询引擎层（`QueryEngine.ts`、`query.ts`）**
- 核心对话循环：向 API 发送消息、接收响应、执行工具
- `QueryEngine` 类：管理对话的生命周期（多轮、有状态）
- `query()` 函数：处理主查询循环生成器，包括：
  - 消息规范化 和上下文准备
  - 自动压缩、微压缩、裁剪、上下文折叠
  - API 流式处理与回退模型支持
  - 工具执行编排（流式和批量）
  - Token 预算跟踪和恢复
  - 后采样钩子和停止钩子
- `ask()` 便捷包装器：用于无头/SDK 模式的一次性查询

**6. 工具层（`Tool.ts`、`tools.ts`、`tools/*/`）**
- `Tool<T>` 类型：全面的工具接口，包含输入/输出模式、权限检查、渲染
- `buildTool()`：带有安全默认值的工厂函数
- `getTools()`：根据权限上下文和功能标志组装可用工具
- `assembleToolPool()`：将内置工具与 MCP 工具组合
- 每个工具都是自包含模块，包含：模式、描述、调用逻辑、权限检查、UI 渲染
- 60+ 内置工具：Bash、Read、Edit、Write、Glob、Grep、WebSearch、Agent、Task*、Skill 等

**7. 服务层（`services/*`）**
- **API**：Anthropic API 客户端、重试逻辑、日志记录、使用跟踪
- **MCP**：Model Context Protocol 服务器/客户端管理、OAuth、资源处理
- **分析**：GrowthBook 功能标志、事件日志、遥测（OpenTelemetry）
- **压缩**：自动压缩、微压缩、响应式压缩、会话内存
- **LSP**：用于诊断的语言服务器协议集成
- **插件**：插件加载、版本控制、市场集成
- **技能**：技能发现、加载、热重载检测

**8. 状态层**
- **AppState**（`state/AppStateStore.ts`）：包裹在 `DeepImmutable<>` 中的中央响应式状态树。包含：设置、工具、MCP 状态、插件、任务、待办事项、通知、桥接状态、推测状态等。
- **引导状态**（`bootstrap/state.ts`）：会话级数据的全局单例状态（成本、持续时间、会话 ID、模型、遥测计数器）。通过 getter/setter 函数访问。
- **Store**（`state/store.ts`）：通用状态存储实现（可能是 Zustand 或类似模式）

**9. 工具层（`utils/*`）**
- 200+ 工具模块覆盖：git 操作、认证、设置管理、权限设置、文件操作、字符串操作、平台检测和领域特定辅助函数

## 模块组织

代码库在 `src/` 下组织为以下顶级目录：

### 核心模块

| 目录 | 用途 | 关键文件 |
|---|---|---|
| `entrypoints/` | 应用程序入口点 | `cli.tsx`、`mcp.ts`、`init.ts`、`agentSdkTypes.ts` |
| `bootstrap/` | 全局会话状态管理 | `state.ts`（1500+ 行 getter/setter） |
| `state/` | 响应式应用程序状态 | `AppStateStore.ts`、`AppState.tsx`、`store.ts`、`selectors.ts` |
| `commands/` | 斜杠命令（/、本地命令） | 90+ 命令模块按功能组织 |
| `tools/` | 工具实现 | 60+ 工具模块，每个在各自目录 |
| `services/` | 外部服务集成 | API、MCP、分析、压缩、LSP、插件 |
| `components/` | React/Ink UI 组件 | 消息渲染、模态框、面板、对话框 |
| `ink/` | Ink 框架定制 | 自定义 Ink 组件、钩子、事件、DOM |
| `query/` | 查询循环内部 | `config.ts`、`deps.ts`、`stopHooks.ts`、`tokenBudget.ts`、`transitions.ts` |
| `types/` | TypeScript 类型定义 | 消息、权限、工具、钩子、命令 |
| `constants/` | 应用程序常量 | 系统提示、工具限制、XML 标签、API 限制 |
| `utils/` | 工具函数 | 200+ 辅助模块 |

### 功能特定模块

| 目录 | 用途 |
|---|---|
| `coordinator/` | 多 agent 编排的协调器模式 |
| `assistant/` | 助手模式（Kairos）- 主动助手功能 |
| `bridge/` | 远程控制桥接（claude.ai 集成） |
| `daemon/` | 长期运行的守护进程管理 |
| `buddy/` | 伴侣/精灵功能 |
| `vim/` | Vim 模式实现 |
| `voice/` | 语音模式实现 |
| `upstreamproxy/` | CCR 上游代理用于凭证注入 |
| `jobs/` | 模板作业分类 |
| `plugins/` | 插件系统核心 |
| `skills/` | 技能系统核心 |
| `memdir/` | 内存目录管理 |
| `native-ts/` | 本机模块类型定义 |
| `outputStyles/` | 输出样式定制 |
| `remote/` | 远程会话管理 |
| `server/` | 服务器端组件（会话、直接连接） |
| `self-hosted-runner/` | 自托管运行器实现 |
| `environment-runner/` | BYOC 环境运行器 |

### 子目录组织

**`tools/`** - 每个工具遵循一致的模式：
```
tools/
├── BashTool/
│   ├── BashTool.ts          # 主要工具实现
│   ├── prompt.ts            # 系统提示说明
│   ├── constants.ts         # 工具特定常量
│   ├── bashSecurity.ts      # 安全验证
│   ├── bashPermissions.ts   # 权限逻辑
│   └── ...                  # 额外的辅助函数
├── FileEditTool/
├── FileReadTool/
├── AgentTool/
└── ...
```

**`services/`** - 按服务域组织：
```
services/
├── api/                     # Anthropic API 客户端
├── mcp/                     # Model Context Protocol
├── analytics/               # 功能标志、遥测
├── compact/                 # 上下文压缩策略
├── lsp/                     # 语言服务器协议
├── tips/                    # 用户提示系统
├── teamMemorySync/          # 团队内存同步
└── ...
```

**`commands/`** - 每个命令都是自包含的：
```
commands/
├── mcp/                     # MCP 子命令
│   ├── index.ts
│   └── addCommand.ts
├── plugin/
├── skills/
├── compact/
└── ...（每个在自己的目录或文件中）
```

## 关键设计模式

### 1. 带死代码消除的功能标志

代码库广泛使用 `bun:bundle` 的 `feature()` 进行编译时功能门控：

```typescript
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

这在构建时启用死代码消除（DCE）—— 当功能标志被禁用时，整个模块从外部构建中排除。这对于维护服务于内部（ant）和外部（客户）构建的单一代码库至关重要。

关键功能标志包括：
- `COORDINATOR_MODE` - 多 agent 协调器
- `KAIROS` - 助手/主动模式
- `BRIDGE_MODE` - 远程控制桥接
- `DAEMON` - 长期运行的守护进程
- `BG_SESSIONS` - 后台会话管理
- `HISTORY_SNIP` - 历史裁剪压缩
- `CONTEXT_COLLAPSE` - 上下文折叠功能
- `SSH_REMOTE` - SSH 远程会话
- `DIRECT_CONNECT` - 直接连接会话
- `WEB_BROWSER_TOOL` - Web 浏览器工具
- `WORKFLOW_SCRIPTS` - 工作流自动化
- `TRANSCRIPT_CLASSIFIER` - 自动模式分类器
- `UDS_INBOX` - Unix 域套接字消息
- `CHICAGO_MCP` - 计算机使用 MCP

### 2. 用于懒加载的动态导入

重型模块被动态加载以最小化启动时间：

```typescript
const { setup } = await import('./setup.js')
const { bridgeMain } = await import('../bridge/bridgeMain.js')
```

这与启动分析器（`profileCheckpoint`）结合使用来跟踪模块评估时间。

### 3. 缓存的全局状态

引导状态使用带 getter/setter 函数的单例模式：

```typescript
const STATE: State = getInitialState()
export function getSessionId(): SessionId {
  return STATE.sessionId
}
export function switchSession(sessionId: SessionId, projectDir: string | null): void {
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)
}
```

基于信号的响应式（`createSignal`）允许订阅者对状态变化做出反应。

### 4. 带 DeepImmutable 类型的不可变 AppState

AppState 被包裹在 `DeepImmutable<>` 中以在类型级别强制不可变性：

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  // ...
}> & {
  tasks: { [taskId: string]: TaskState }  // 从 DeepImmutable 排除（包含函数）
  // ...
}
```

状态更新使用函数式更新模式：`setAppState(prev => ({ ...prev, key: newValue }))`

### 5. 查询循环的异步生成器模式

查询系统使用异步生成器进行流式处理：

```typescript
export async function* query(params: QueryParams): AsyncGenerator<Message | StreamEvent, Terminal> {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // ...
}
```

这使得消息、工具执行和事件能够实时流式传输给消费者。

### 6. 工具抽象模式

工具遵循全面的接口，包括：
- 用于验证的 Zod 输入模式
- 动态描述生成
- 权限检查（`checkPermissions`）
- 并发安全标志
- UI 渲染方法（`renderToolUseMessage`、`renderToolResultMessage`）
- 进度报告
- 用于安全分类的自动分类器输入

### 7. 上下文传递模式

工具执行使用携带以下内容的 `ToolUseContext` 对象：
- 选项（命令、工具、模型、MCP 客户端）
- 中止控制器
- 状态访问器（`getAppState`、`setAppState`）
- 回调（JSX 渲染、通知、提示）
- 跟踪状态（查询链、技能发现、内存路径）

### 8. 钩子系统

关键生命周期点的可扩展钩子：
- `Setup` 钩子：在初始化期间运行
- `SessionStart` 钩子：在会话开始时运行
- `PreToolUse`/`PostToolUse` 钩子：在工具执行周围运行
- `PreCompact`/`PostCompact` 钩子：在压缩周围运行
- `Stop` 钩子：在模型停止生成时运行

### 9. 预取模式

通过积极预取优化启动性能：
- 早期预取：MDM 设置、钥匙串、API 预连接
- 延迟预取：用户上下文、提示、文件计数、模型能力
- 并行执行：独立预取并发运行
- 信任门控：某些预取仅在信任对话框接受后运行

### 10. 命令模式

斜杠命令遵循带类型的一致模式：
- `prompt` 命令：展开为发送给模型的文本
- `local` 命令：在本地执行（文本输出）
- `local-jsx` 命令：渲染 Ink UI 组件

## 数据流概述

### 交互会话流

```
用户输入
    │
    ▼
┌──────────────┐
│  CLI 解析器   │ (Commander.js - main.tsx)
│  (标志、选项) │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│  设置加载     │────▶│ 权限上下文   │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│  设置         │────▶│ 工具池组装   │
│  (钩子、环境) │     └──────┬───────┘
└──────┬───────┘            │
       │                    ▼
       ▼         ┌─────────────────────────────────────┐
┌──────────────┐ │  REPL（交互式 TUI）                   │
│  初始化完成   │ │  ┌───────────────────────────────┐  │
└──────────────┘ │  │  消息队列                      │  │
                 │  │  ↓                             │  │
                 │  │  QueryEngine.submitMessage()  │  │
                 │  │  ↓                             │  │
                 │  │  query() 循环：                │  │
                 │  │    1. 上下文准备               │  │
                 │  │    2. API 流式处理             │  │
                 │  │    3. 工具执行                 │  │
                 │  │    4. 钩子执行                 │  │
                 │  │    5. 重复直到完成              │  │
                 │  └───────────────────────────────┘  │
                 └─────────────────────────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  转录持久化   │
                    └──────────────┘
```

### 非交互式（SDK）流

```
stdin (stream-json)
    │
    ▼
┌──────────────┐
│  CLI 解析器   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  QueryEngine │
│  submitMessage│
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  query() 循环 │
│  (与交互式    │
│   相同)      │
└──────┬───────┘
       │
       ▼
stdout (stream-json)
```

### 工具执行流

```
模型请求工具使用
    │
    ▼
┌──────────────┐
│  validateInput│ (工具特定验证)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  checkPermissions│ (权限规则评估)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  canUseTool   │ (钩子执行、分类器)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  tool.call()  │ (实际执行)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  ToolResult   │ (结果 + 可选的新消息)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  API 响应     │ (工具结果发送回模型)
└──────────────┘
```

## 功能标志

功能标志系统在两个级别上运行：

### 构建时标志（`feature()` 来自 `bun:bundle`）

这些在构建时评估并启用死代码消除：

| 标志 | 用途 | 构建影响 |
|---|---|---|
| `COORDINATOR_MODE` | 多 agent 协调器 | 排除协调器模块 |
| `KAIROS` | 助手/主动模式 | 排除助手模块 |
| `BRIDGE_MODE` | 远程控制 | 排除桥接模块 |
| `DAEMON` | 守护进程 | 排除守护进程模块 |
| `BG_SESSIONS` | 后台会话 | 排除后台管理 |
| `HISTORY_SNIP` | 历史裁剪 | 排除裁剪模块 |
| `CONTEXT_COLLAPSE` | 上下文折叠 | 排除折叠模块 |
| `SSH_REMOTE` | SSH 会话 | 排除 SSH 模块 |
| `DIRECT_CONNECT` | 直接连接 | 排除直接连接 |
| `WEB_BROWSER_TOOL` | 浏览器工具 | 排除浏览器工具 |
| `WORKFLOW_SCRIPTS` | 工作流 | 排除工作流模块 |
| `TRANSCRIPT_CLASSIFIER` | 自动模式 | 排除分类器 |
| `UDS_INBOX` | UDS 消息 | 排除 UDS 模块 |
| `CHICAGO_MCP` | 计算机使用 | 排除 CU 模块 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 | 排除推送模块 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub Webhooks | 排除 Webhook 模块 |
| `AGENT_TRIGGERS` | Cron 触发器 | 排除 Cron 模块 |
| `MONITOR_TOOL` | 监控工具 | 排除监控模块 |
| `OVERFLOW_TEST_TOOL` | 测试工具 | 排除测试模块 |
| `TERMINAL_PANEL` | 终端面板 | 排除终端捕获 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索 | 排除技能搜索 |
| `CCR_REMOTE_SETUP` | 远程设置 | 排除远程设置 |
| `FORK_SUBAGENT` | 分叉子 agent | 排除分叉模块 |
| `BUDDY` | 伴侣功能 | 排除伴侣模块 |
| `TORCH` | Torch 功能 | 排除 Torch 模块 |
| `ULTRAPLAN` | Ultraplan 功能 | 排除 Ultraplan 模块 |
| `PROACTIVE` | 主动模式 | 排除主动模块 |
| `REACTIVE_COMPACT` | 响应式压缩 | 排除响应式压缩 |
| `CACHED_MICROCOMPACT` | 缓存微压缩 | 排除缓存 MC |
| `TEMPLATES` | 模板 | 排除模板模块 |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC 运行器 | 排除 BYOC 模块 |
| `SELF_HOSTED_RUNNER` | 自托管运行器 | 排除 SH 运行器 |
| `ABLATION_BASELINE` | 消融测试 | 排除消融代码 |
| `DUMP_SYSTEM_PROMPT` | 系统提示转储 | 排除转储命令 |
| `BREAK_CACHE_COMMAND` | 缓存破坏 | 排除缓存破坏 |
| `MCP_SKILLS` | MCP 技能 | 排除 MCP 技能代码 |
| `TEAMMEM` | 团队内存 | 排除团队内存 |
| `COMMIT_ATTRIBUTION` | 提交归因 | 排除归因 |
| `TOKEN_BUDGET` | Token 预算 | 排除预算代码 |
| `LODESTONE` | 深度链接 | 排除深度链接处理程序 |

### 运行时标志（GrowthBook）

从 GrowthBook 在运行时获取的功能值：
- `tengu_kairos` - 助手模式门控
- `tengu_ccr_bridge` - 桥接门控
- `tengu_tool_pear` - 工具严格模式
- `tengu_otk_slot_v1` - 输出 Token 升级
- 各种配置驱动的功能值

### 环境变量标志

通过环境变量的运行时切换：
- `CLAUDE_CODE_SIMPLE` - 最小模式
- `CLAUDE_CODE_REMOTE` - 远程模式
- `CLAUDE_CODE_TASK_LIST_ID` - 任务列表模式
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS` - 禁用 CLAUDE.md
- `CLAUDE_CODE_USE_BEDROCK` - Bedrock 提供商
- `CLAUDE_CODE_USE_VERTEX` - Vertex 提供商
- `USER_TYPE=ant` - 内部 Anthropic 构建
- `NODE_ENV=test` - 测试模式

## 外部依赖

### 核心框架依赖

| 依赖 | 用途 |
|---|---|
| `@anthropic-ai/sdk` | 用于 Claude 消息的 Anthropic API 客户端 |
| `@commander-js/extra-typings` | 带 TypeScript 支持的 CLI 参数解析 |
| `react` + `ink` | 终端 UI 渲染框架 |
| `zod/v4` | 工具输入的运行时模式验证 |
| `@modelcontextprotocol/sdk` | MCP 协议实现 |
| `@opentelemetry/*` | 遥测（指标、日志、跟踪） |
| `bun:bundle` | 构建时功能标志（Bun 特定） |
| `chalk` | 终端颜色输出 |
| `lodash-es` | 工具函数（记忆化、mapValues、pickBy 等） |

### 基础设施依赖

| 依赖 | 用途 |
|---|---|
| `@anthropic-ai/sdk` | 与 Claude 的 API 通信 |
| `@modelcontextprotocol/sdk` | MCP 服务器/客户端协议 |
| OpenTelemetry SDK | 客户面向遥测 |
| GrowthBook | 功能标志评估 |
| Datadog | 内部分析（仅 ant） |

### 系统依赖

| 依赖 | 用途 |
|---|---|
| `node:crypto` | UUID 生成、随机字节 |
| `node:fs` | 文件系统操作 |
| `node:child_process` | Shell 命令执行 |
| `node:vm` | REPL 工具 VM 上下文 |
| `node:net` | Unix 域套接字、网络 |
| `node:readline` | 终端输入处理 |
| `node:events` | 事件发射器模式 |

## 模块关系

### 依赖图（高级）

```
cli.tsx (入口点)
    │
    ├──► main.tsx (CLI 编排)
    │       │
    │       ├──► bootstrap/state.ts (全局会话状态)
    │       ├──► setup.ts (环境设置)
    │       ├──► commands.ts (命令注册)
    │       ├──► tools.ts (工具组装)
    │       │       └──► Tool.ts (工具类型定义)
    │       │               └──► tools/* (各个工具)
    │       ├──► QueryEngine.ts (查询引擎)
    │       │       └──► query.ts (查询循环)
    │       │               └──► services/api/claude.ts (API 客户端)
    │       │               └──► services/tools/toolOrchestration.ts
    │       ├──► state/AppStateStore.ts (响应式状态)
    │       ├──► context.ts (系统/用户上下文)
    │       └──► interactiveHelpers.ts (REPL 辅助函数)
    │
    ├──► services/mcp/ (MCP 集成)
    ├──► services/analytics/ (遥测、GrowthBook)
    ├──► services/compact/ (上下文管理)
    ├──► utils/* (200+ 工具模块)
    └──► components/* (React/Ink UI 组件)
```

### 关键模块关系

**main.tsx → setup.ts → bootstrap/state.ts**
- `main.tsx` 编排启动，在解析 CLI 选项后调用 `setup()`
- `setup.ts` 初始化环境、钩子和 worktree
- 两者都读取/写入 `bootstrap/state.ts` 以获取会话级状态

**main.tsx → tools.ts → Tool.ts → tools/**
- `main.tsx` 调用 `getTools()` 来组装工具池
- `tools.ts` 导入各个工具实现并按权限过滤
- `Tool.ts` 定义工具接口和 `buildTool()` 工厂
- `tools/` 中的每个工具实现 `Tool` 接口

**QueryEngine.ts → query.ts → services/api/claude.ts**
- `QueryEngine.submitMessage()` 包装 `query()` 生成器
- `query()` 处理主循环：API 调用 → 工具执行 → 重复
- `services/api/claude.ts` 处理实际 API 流式处理

**state/AppStateStore.ts ↔ components/**
- `AppState` 是响应式 UI 状态的单一真相来源
- 组件通过选择器或直接访问从 AppState 读取
- 状态更新通过 `setAppState()` 函数更新流动

**commands.ts ↔ skills/ ↔ plugins/**
- 命令从多个来源加载：内置、技能、插件
- `getCommands()` 合并所有来源并进行可用性过滤
- 技能和插件可以在运行时动态添加命令

### 循环依赖管理

代码库使用多种模式来管理循环依赖：
1. **懒导入**：`require()` 在函数内部打破循环
2. **仅类型导入**：`import type` 用于类型引用
3. **集中式类型定义**：`types/` 目录中的共享类型
4. **功能门控导入**：基于 `feature()` 标志的条件导入

### 导入 DAG 强制

`bootstrap/` 目录被视为 DAG 叶节点——它不能从更高级别的模块导入。这通过 `bootstrap-isolation` ESLint 规则强制执行。引导状态使用显式 getter/setter 函数，而不是从应用程序层导入复杂类型。
