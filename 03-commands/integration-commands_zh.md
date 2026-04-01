# 集成命令

## 目的

记录将 Claude Code 与外部平台和环境集成的 CLI 命令：GitHub Actions、Slack、Chrome 浏览器、桌面应用、IDE 和移动设备。这些命令将 Claude Code 的覆盖范围扩展到终端之外。

---

## /install-github-app（GitHub Actions 设置）

### 目的
通过多步交互式向导为仓库设置 Claude GitHub Actions。安装 Claude GitHub App，配置 API 密钥 secrets，并创建工作流文件。

### 位置
`restored-src/src/commands/install-github-app/index.ts`
`restored-src/src/commands/install-github-app/install-github-app.tsx`

以及步骤组件：
- `ApiKeyStep.tsx`、`CheckExistingSecretStep.tsx`、`CheckGitHubStep.tsx`
- `ChooseRepoStep.tsx`、`CreatingStep.tsx`、`ErrorStep.tsx`
- `ExistingWorkflowStep.tsx`、`InstallAppStep.tsx`、`OAuthFlowStep.tsx`
- `SuccessStep.tsx`、`WarningsStep.tsx`、`setupGitHubActions.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `install-github-app` |
| 类型 | `local-jsx` |
| 描述 | Set up Claude GitHub Actions for a repository |
| 可用性 | `['claude-ai', 'console']` |
| 启用 | 除非 `DISABLE_INSTALL_GITHUB_APP_COMMAND` 环境变量为真 |

### 关键导出

#### 函数
- `call(onDone)`：入口点；渲染 `InstallGitHubApp` 组件

#### 组件
- `InstallGitHubApp`：管理整个 GitHub App 安装流程的多步向导

### 实现细节

#### 多步流程
向导逐步执行：

| 步骤 | 描述 |
|------|-------------|
| `check-gh` | 检查 GitHub CLI 安装、认证和所需范围（`repo`、`workflow`） |
| `warnings` | 显示任何警告（缺少 CLI、未认证）以及说明 |
| `choose-repo` | 选择目标仓库（当前仓库或手动输入） |
| `install-app` | 在浏览器中打开 GitHub App 安装 URL |
| `check-existing-workflow` | 检查 Claude 工作流文件是否已存在 |
| `select-workflows` | 用于选择安装哪些工作流的对话框（`claude`、`claude-review`） |
| `check-existing-secret` | 检查仓库中是否存在 `ANTHROPIC_API_KEY` secret |
| `api-key` | 输入 API 密钥或创建 OAuth 令牌 |
| `oauth-flow` | OAuth 认证流程（启用 OAuth 时） |
| `creating` | 设置 GitHub Actions 时的进度显示 |
| `success` | 完成确认 |
| `error` | 错误显示及说明 |

#### GitHub CLI 检查
`checkGitHubCLI` 函数：
1. 运行 `gh --version` — 如果未找到则警告
2. 运行 `gh auth status -a` — 如果未认证则警告
3. 从 stdout 解析令牌范围 — 如果缺少 `repo` 或 `workflow` 范围则错误
4. 通过 `getGithubRepo()` 检测当前仓库
5. 如果检测到仓库则设置 `useCurrentRepo: true`

#### 仓库验证
- 接受 `owner/repo` 格式或完整 GitHub URL
- 通过正则表达式从 URL 提取仓库名称：`/github\.com[:/]([^/]+\/[^/]+)(\.git)?$/`
- 通过 `gh api repos/{repo} --jq '.permissions.admin'` 检查管理员权限
- 通过 `gh api repos/{repo}/contents/.github/workflows/claude.yml` 检查现有工作流
- 通过 `gh secret list --app actions --repo {repo}` 检查现有 secrets

#### API 密钥/OAuth 选项
三种认证方法：
1. **现有 API 密钥**：使用本地配置的 `ANTHROPIC_API_KEY`
2. **新 API 密钥**：用户手动输入 API 密钥
3. **OAuth 令牌**：通过交互式流程创建 OAuth 令牌（当 `isAnthropicAuthEnabled()` 时）

使用 OAuth 时，secret 名称更改为 `CLAUDE_CODE_OAUTH_TOKEN`，`authType` 设置为 `oauth_token`。

#### 工作流安装
- 支持两个工作流：`claude` 和 `claude-review`
- 如果工作流文件已存在，用户可以更新或跳过
- 通过 `currentWorkflowInstallStep` 计数器跟踪安装进度
- 使用 `setupGitHubActions()` 进行实际设置

#### 分析事件
| 事件 | 何时 |
|-------|------|
| `tengu_install_github_app_started` | 命令启动 |
| `tengu_install_github_app_step_completed` | 每步完成 |
| `tengu_install_github_app_error` | 发生错误（带原因） |
| `tengu_install_github_app_completed` | 成功完成 |

---

## /install-slack-app（Slack App 安装）

### 目的
在浏览器中打开 Slack 应用市场页面以安装 Claude Slack 应用集成。

### 位置
`restored-src/src/commands/install-slack-app/index.ts`
`restored-src/src/commands/install-slack-app/install-slack-app.ts`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `install-slack-app` |
| 类型 | `local` |
| 描述 | Install the Claude Slack app |
| 可用性 | `['claude-ai']` |
| 非交互式 | `false` |

### 关键导出

#### 函数
- `call()`：在浏览器中打开 Slack 应用市场并跟踪安装尝试

### 实现细节

#### 核心逻辑
1. 记录分析事件 `tengu_install_slack_app_clicked`
2. 在全局配置中增加 `slackAppInstallCount`（跟踪用户尝试安装的次数）
3. 在浏览器中打开 `https://slack.com/marketplace/A08SF47R6P4-claude`
4. 返回成功或回退 URL 消息

### 数据流
```
User runs /install-slack-app
  → logEvent('tengu_install_slack_app_clicked')
  → saveGlobalConfig(slackAppInstallCount: count + 1)
  → openBrowser(SLACK_APP_URL)
  │
  ├─ Success → "Opening Slack app installation page in browser…"
  └─ Failed → "Couldn't open browser. Visit: <SLACK_APP_URL>"
```

---

## /chrome（Chrome 中的 Claude）

### 目的
管理 Chrome 中的 Claude（Beta）集成 — 一个 Chrome 扩展，让 Claude Code 直接控制浏览器，用于网页自动化、表单填写、截图和调试。

### 位置
`restored-src/src/commands/chrome/index.ts`
`restored-src/src/commands/chrome/chrome.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `chrome` |
| 类型 | `local-jsx` |
| 描述 | Claude in Chrome (Beta) settings |
| 可用性 | `['claude-ai']` |
| 启用 | 仅在交互式会话中 |

### 关键导出

#### 函数
- `call(onDone)`：入口点；检查扩展状态并渲染 `ClaudeInChromeMenu`

### 实现细节

#### 菜单选项
| 选项 | 动作 |
|--------|--------|
| Install Chrome extension | 在浏览器中打开扩展 URL |
| Manage permissions | 打开权限配置 URL |
| Reconnect extension | 打开重连 URL |
| Enabled by default: Yes/No | 切换 `claudeInChromeDefaultEnabled` 配置 |

#### URL 路由
- **Homespace**：直接使用 `openBrowser(url)`
- **非 Homespace**：使用 `openInChrome(url)` 在 Chrome 集成中打开 URL

#### 连接状态
菜单显示：
- **Status**：基于 MCP 客户端连接状态（`CLAUDE_IN_CHROME_MCP_SERVER_NAME`）的启用/禁用
- **Extension**：基于 `isChromeExtensionInstalled()` 检查的已安装/未检测到

#### 先决条件和警告
- WSL：显示错误"Claude in Chrome is not supported in WSL at this time"
- 订阅：显示错误"Claude in Chrome requires a claude.ai subscription"
- 扩展未安装：显示安装提示和相关选项的"requires extension"后缀

---

## /desktop（桌面应用交接）

### 目的
在 Claude 桌面应用中继续当前会话（仅 macOS 和 Windows x64）。

### 位置
`restored-src/src/commands/desktop/index.ts`
`restored-src/src/commands/desktop/desktop.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `desktop` |
| 别名 | `app` |
| 类型 | `local-jsx` |
| 描述 | Continue the current session in Claude Desktop |
| 可用性 | `['claude-ai']` |
| 启用 | 仅在 macOS（`darwin`）或 Windows x64（`win32` + `x64`）上 |
| 隐藏 | 在不支持的平台 |

### 实现细节

#### 平台支持
`isSupportedPlatform()` 函数检查：
- macOS：`process.platform === 'darwin'` → 支持
- Windows x64：`process.platform === 'win32' && process.arch === 'x64'` → 支持
- 所有其他平台 → 不支持（命令隐藏）

---

## /ide（IDE 集成）

### 目的
管理 IDE 集成 — 检测运行中带有 Claude Code 扩展/插件的 IDE，连接到它们以进行集成开发功能，并提供在检测到的 IDE 上安装扩展的选项。

### 位置
`restored-src/src/commands/ide/index.ts`
`restored-src/src/commands/ide/ide.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `ide` |
| 类型 | `local-jsx` |
| 描述 | Manage IDE integrations and show status |
| 参数提示 | `[open]` |

### 关键导出

#### 函数
- `call(onDone, context, args)`：入口点；检测 IDE 并渲染适当的 UI
- `formatWorkspaceFolders(folders, maxLength?)`：格式化工作区文件夹路径以进行显示
- `findCurrentIDE(availableIDEs, dynamicMcpConfig?)`：从动态 MCP 配置中查找当前连接的 IDE

#### 组件
- `IDEScreen`：主 IDE 选择屏幕
- `IDEOpenSelection`：用于打开项目的 IDE 选择
- `RunningIDESelector`：运行多个 IDE 时的选择器（用于扩展安装）
- `InstallOnMount`：挂载时自动安装扩展
- `IDECommandFlow`：编排 IDE 连接的主流程

### 实现细节

#### 两种模式

**模式 1：`/ide open` — 在 IDE 中打开项目**
1. 获取当前 worktree 会话路径（或 cwd）
2. 检测带有 Claude Code 扩展的可用 IDE
3. 显示 IDE 选择对话框
4. 选择时：
   - VS Code 类 IDE（VS Code、Cursor、Windsurf）：通过 `code <path>` 打开
   - JetBrains IDE：提示用户手动打开
5. 显示成功或失败消息

**模式 2：`/ide` — 连接到 IDE**
1. 检测可用的 IDE（带有扩展）和不可用的 IDE（运行但工作区错误）
2. 从动态 MCP 配置中查找当前连接的 IDE
3. 渲染 `IDECommandFlow`，它：
   - 在选择对话框中显示可用 IDE
   - 选择时：更新动态 MCP 配置以连接到 IDE
   - 监控连接状态，超时 35 秒
   - 断开连接时：清除 MCP 配置并从状态中移除 IDE 客户端

#### IDE 检测
- **可用 IDE**：运行带有 Claude Code 扩展/插件**且**匹配工作区目录的 IDE
- **不可用 IDE**：运行但工作区/项目目录与当前 cwd 不匹配的 IDE
- 当未检测到带扩展的 IDE 时，检查运行中的 IDE 并提供安装扩展选项

#### MCP 连接
IDE 通过动态 MCP 服务器配置连接：
- `sse-ide` 类型用于 SSE 连接
- `ws-ide` 类型用于 WebSocket 连接
- 配置包括：URL、IDE 名称、认证令牌、Windows 检测标志

---

## /mobile（移动应用二维码）

### 目的
显示从 App Store（iOS）或 Google Play（Android）下载 Claude 移动应用的二维码。

### 位置
`restored-src/src/commands/mobile/index.ts`
`restored-src/src/commands/mobile/mobile.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `mobile` |
| 别名 | `ios`, `android` |
| 类型 | `local-jsx` |
| 描述 | Show QR code to download the Claude mobile app |

### 关键导出

#### 函数
- `call(onDone)`：入口点；渲染 `MobileQRCode` 组件

### 实现细节

#### 平台 URL
| 平台 | URL |
|----------|-----|
| iOS | `https://apps.apple.com/app/claude-by-anthropic/id6473753684` |
| Android | `https://play.google.com/store/apps/details?id=com.anthropic.claude` |

#### 二维码生成
- 在挂载时为两个平台生成二维码（预生成两者）
- 使用 UTF8 格式和低错误纠正的 `qrcode.toString()`
- 在状态中存储两个二维码以便即时平台切换

---

## 集成命令概述

这六个命令将 Claude Code 连接到外部平台：

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ /install-github   │  │ /install-slack    │  │    /chrome      │
│      -app         │  │      -app         │  │ (browser control │
│ (GitHub Actions)  │  │  (Slack app)      │  │  via extension)  │
└──────────────────┘  └──────────────────┘  └──────────────────┘

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   /desktop        │  │      /ide         │  │    /mobile       │
│ (Desktop app      │  │ (IDE integration  │  │ (mobile app QR   │
│  handoff)         │  │  VS Code, JB)     │  │  code)           │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 平台可用性
| 命令 | 平台 | 要求 |
|---------|-----------|-------------|
| `/install-github-app` | 所有 | GitHub CLI、`gh` 认证带有 repo+workflow 范围 |
| `/install-slack-app` | 所有 | 浏览器访问 |
| `/chrome` | 所有（除 WSL） | claude.ai 订阅、Chrome 扩展 |
| `/desktop` | macOS、Windows x64 | 已安装 Claude Desktop |
| `/ide` | 所有 | 带 Claude Code 扩展/插件的 IDE |
| `/mobile` | 所有 | 二维码显示能力 |
