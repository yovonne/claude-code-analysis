# 深度链接

## 目的

实现 `claude-cli://` 自定义 URI 方案，用于从外部来源（浏览器、其他应用）启动 Claude Code 会话。涵盖带安全验证的 URI 解析、OS 级协议处理器注册（macOS/Linux/Windows）、终端模拟器检测和启动，以及深度链接发起会话的安全横幅。

## 位置

- `restored-src/src/utils/deepLink/parseDeepLink.ts` — URI 解析和验证
- `restored-src/src/utils/deepLink/registerProtocol.ts` — OS 级协议处理器注册
- `restored-src/src/utils/deepLink/protocolHandler.ts` — `claude --handle-uri` 的入口点
- `restored-src/src/utils/deepLink/terminalLauncher.ts` — 终端模拟器检测和启动
- `restored-src/src/utils/deepLink/terminalPreference.ts` — 无头上下文的终端偏好捕获
- `restored-src/src/utils/deepLink/banner.ts` — 深度链接会话的安全警告横幅

## 主要导出

### URI 解析器（`parseDeepLink.ts`）

#### 类型
- `DeepLinkAction`: `{ query?, cwd?, repo? }` — 从 URI 解析的操作

#### 常量
- `DEEP_LINK_PROTOCOL`: `'claude-cli'`

#### 函数
- `parseDeepLink(uri)`: 将 `claude-cli://open` URI 解析为结构化操作。验证方案、主机名和所有参数。
- `buildDeepLink(action)`: 从操作对象构建 `claude-cli://` URL。

#### 验证规则
- `cwd`：必须是绝对路径，最多 4096 字符，无 ASCII 控制字符
- `repo`：必须匹配 `owner/repo` 模式（字母数字、点、短划线、下划线，正好一个斜杠）
- `q`（查询）：Unicode 清理，无控制字符，最多 5000 字符
- 所有字段拒绝控制字符（0x00-0x1F、0x7F）

### 协议处理器注册（`registerProtocol.ts`）

#### 常量
- `MACOS_BUNDLE_ID`: `'com.anthropic.claude-code-url-handler'`

#### 函数
- `registerProtocolHandler(claudePath?)`: 在 OS 中注册 `claude-cli://` 方案
- `isProtocolHandlerCurrent(claudePath)`: 检查 OS 注册是否指向预期二进制文件
- `ensureDeepLinkProtocolRegistered()`: 缺少或过时时自动注册（每个会话运行一次，fire-and-forget）

#### 平台详情
- **macOS**：在 `~/Applications` 创建最小 `.app` 包，带 Info.plist 中的 `CFBundleURLTypes`；符号链接到签名的 `claude` 二进制文件；用 LaunchServices 重新注册
- **Linux**：在 `$XDG_DATA_HOME/applications` 创建 `.desktop` 文件；用 `xdg-mime` 注册
- **Windows**：在 `HKEY_CURRENT_USER\Software\Classes\claude-cli` 下写注册表项

### 协议处理器（`protocolHandler.ts`）

#### 函数
- `handleDeepLinkUri(uri)`: 解析 URI，解析 cwd（显式 > repo MRU 克隆 > home），读取 FETCH_HEAD 年龄，在终端启动。返回退出代码。
- `handleUrlSchemeLaunch()`: 通过 NAPI 模块处理 macOS URL 方案启动。返回退出代码，如果不是 URL 启动则返回 null。
- `resolveCwd(action)`: 解析工作目录：显式 cwd > repo 查找（MRU 克隆）> home 目录

### 终端启动器（`terminalLauncher.ts`）

#### 类型
- `TerminalInfo`: `{ name, command }`

#### 函数
- `detectTerminal()`: 检测用户的首选终端模拟器（平台特定）
- `launchInTerminal(claudePath, action)`: 在检测到的终端中启动 Claude Code，带适当参数

#### 平台支持
- **macOS**：iTerm2、Ghostty、Kitty、Alacritty、WezTerm、Terminal.app（按偏好顺序）
- **Linux**：ghostty、kitty、alacritty、wezterm、gnome-terminal、konsole、xfce4-terminal、mate-terminal、tilix、xterm
- **Windows**：Windows Terminal（wt.exe）、PowerShell、cmd.exe

#### 安全：纯 argv vs Shell 字符串
- 纯 argv 路径（无 shell）：Ghostty、Alacritty、Kitty、WezTerm、所有 Linux 终端、Windows Terminal — 用户输入作为独立的 argv 元素传递
- Shell 字符串路径（shell 引号）：iTerm2、Terminal.app（AppleScript）、PowerShell、cmd.exe — 用户输入通过平台特定的引号函数进行 shell 转义

### 终端偏好（`terminalPreference.ts`）

#### 函数
- `updateDeepLinkTerminalPreference()`: 从 `TERM_PROGRAM` 环境变量捕获当前终端，并存储到全局配置中，供无头深度链接处理器稍后使用（仅 macOS）

### 深度链接横幅（`banner.ts`）

#### 类型
- `DeepLinkBannerInfo`: `{ cwd, prefillLength?, repo?, lastFetch? }`

#### 函数
- `buildDeepLinkBanner(info)`: 为深度链接发起的会话构建多行警告横幅。显示工作目录、repo slug、获取年龄和预填充审查提示。
- `readLastFetchTime(cwd)`: 读取 `.git/FETCH_HEAD` 的 mtime 以确定 repo 新鲜度。为 worktree 设置检查 worktree 和 common dir。

#### 安全设计
- 横幅要求用户在提交前按 Enter，提示他们审查预填充的提示
- 当预填充超过 1000 字符时从"仔细审查"切换到"滚动审查整个提示"（不适合一屏）
- 显示 repo 获取年龄，以便用户可以检测过时的 CLAUDE.md 文件

## 设计注意事项

- 协议注册是幂等的 — 工件检查（符号链接目标、.desktop Exec 行、注册表值）使其在首次成功运行后成为空操作
- 退避失败：EACCES/ENOSPC 错误通过标记文件限制为每 24 小时一次
- 深度链接处理器无头运行（无 TTY），因为 OS 直接启动二进制文件
- macOS 上的终端检测首先检查存储的偏好，然后是 `TERM_PROGRAM`，然后是 Spotlight/`/Applications` 查找
- 纯 argv 路径优先于 shell 字符串以提高安全性 — 无 shell 解释用户输入
