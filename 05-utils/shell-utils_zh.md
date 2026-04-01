# Shell 工具

**目录：** `restored-src/src/utils/shell/`

## 目的

提供命令执行的 shell 抽象，包括 shell 提供商接口、平台特定 shell 实现、输出限制和只读命令验证。

## shellProvider.ts

### ShellProvider 接口

```typescript
{
  type: ShellType  // 'bash' | 'powershell'
  shellPath: string
  detached: boolean

  buildExecCommand(command, opts): Promise<{ commandString, cwdFilePath }>
  getSpawnArgs(commandString): string[]
  getEnvironmentOverrides(command): Promise<Record<string, string>>
}
```

### Shell 类型

- `SHELL_TYPES` - `['bash', 'powershell']`
- `DEFAULT_HOOK_SHELL` - `'bash'`
- `SHELL_TOOL_NAMES` - shell 工具名称数组（BashTool、PowerShellTool）

### 关键函数

- `isPowerShellToolEnabled()` - PowerShellTool 的运行时门控
  - 仅 Windows、ant 默认开启（选择退出）、外部默认关闭（选择加入）
  - 检查 `CLAUDE_CODE_USE_POWERSHELL_TOOL` 环境变量

## bashProvider.ts

### Bash Shell 提供商

- `createBashShellProvider(shellPath, options?)` - 创建 Bash ShellProvider
  - 创建环境捕获的 shell 快照
  - 使用以下内容构建执行命令：
    1. 源快照文件（环境状态）
    2. 源会话环境脚本
    3. 禁用扩展 glob 模式（安全）
    4. Eval 包装命令
    5. 通过 `pwd -P` 的 PWD 跟踪
  - 处理带 POSIX tmpdir 路径的沙箱模式
  - 应用 CLAUDE_CODE_SHELL_PREFIX（如果设置）
  - ant 用户的 TMUX 套接字隔离

### 安全特性

- `getDisableExtglobCommand(shellPath)` - 禁用 bash extglob 和 zsh EXTENDED_GLOB
  - 防止恶意文件名利用
  - 处理 shell 前缀包装器（双 shell 模式）

### 环境覆盖

- TMUX 环境变量覆盖用于隔离套接字
- TMPDIR/CLAUDE_CODE_TMPDIR 用于沙箱模式
- TMPPREFIX 用于 zsh heredoc 支持
- 来自 /env 命令的会话环境变量

## powershellProvider.ts

### PowerShell Shell 提供商

- `createPowerShellProvider(shellPath)` - 创建 PowerShell ShellProvider
  - 使用 UTF-16LE base64 编码用于 -EncodedCommand
  - 避免 shell 引号损坏问题
  - 退出代码捕获：首选 $LASTEXITCODE，回退到 $?

### 命令构建

- `buildPowerShellArgs(cmd)` - 返回 `['-NoProfile', '-NonInteractive', '-Command', cmd]`
- `encodePowerShellCommand(psCommand)` - Base64 编码 UTF-16LE 字符串
- 沙箱模式：带完整标志集的自调用 pwsh
- 非沙箱：返回裸命令，标志由 getSpawnArgs 添加

### CWD 跟踪

- 将退出代码捕获和路径跟踪追加到命令
- 处理用于 cwd 文件放置的沙箱 tmpdir

## resolveDefaultShell.ts

### 默认 Shell 解析

- `resolveDefaultShell()` - 返回 `'bash'` 或 `'powershell'`
  - 首先检查 `settings.defaultShell`
  - 回退到 `'bash'`（Windows 不会自动翻转）
  - 在所有平台上保持一致

## readOnlyCommandValidation.ts

### 只读命令验证

Shell 工具的综合命令验证映射：

#### GIT_READ_ONLY_COMMANDS

带允许标志的安全 git 子命令：
- `git diff` - 显示和比较标志
- `git log` - 显示、遍历、格式、pickaxe 标志
- `git show` - 显示和 diff 标志
- `git status` - 输出格式、未跟踪、忽略选项
- `git blame` - 行范围、输出格式、忽略选项
- `git ls-files` - 文件选择、输出格式、排除模式
- `git config --get` - 配置作用域和类型标志
- `git branch` - 仅列表模式（创建被回调阻止）
- `git tag` - 仅列表模式（创建被回调阻止）
- `git remote` / `git remote show` - 仅显示
- `git ls-remote` - 分支/标签过滤（出于安全考虑排除服务器选项）
- `git rev-parse` - SHA 解析、路径查询、布尔查询
- `git rev-list` - 提交枚举、遍历、格式化
- `git describe` - 基于标签的描述
- `git cat-file` - 对象检查（仅批处理检查）
- `git for-each-ref` - 带格式化的 ref 迭代
- `git grep` - 跟踪文件中的模式搜索
- `git stash list/show` - 储藏显示
- `git worktree list` - Worktree 列表
- `git merge-base` - 共同祖先查找
- `git shortlog` / `git reflog` - 日志摘要

每个命令有：
- `safeFlags` - 带参数类型的允许标志（无、数字、字符串、字符）
- `additionalCommandIsDangerousCallback` - 动态危险检查
- `respectsDoubleDash` - `--` 是否终止标志解析

#### GH_READ_ONLY_COMMANDS（仅 ant）

安全的 GitHub CLI 命令：
- `gh pr view/list/diff/checks/status`
- `gh issue view/list/status`
- `gh repo view`
- `gh run list/view`
- `gh auth status`
- `gh release list/view`
- `gh workflow list/view`
- `gh label list`
- `gh search repos/issues/prs/commits/code`

所有都使用 `ghIsDangerousCallback` 防止网络外泄：
- 拒绝 HOST/OWNER/REPO 格式（3+ 段）
- 拒绝 URL（://）和 SSH 样式（@）

#### 其他只读命令

- `DOCKER_READ_ONLY_COMMANDS` - docker logs、docker inspect
- `EXTERNAL_READONLY_COMMANDS` - 跨 shell 安全命令（cat、ls、echo 等）

### UNC 路径检测

- `containsVulnerableUncPath(path)` - 检测可能泄露凭证的 UNC 路径

## outputLimits.ts

### 输出长度限制

- `BASH_MAX_OUTPUT_UPPER_LIMIT` - 150,000 字符
- `BASH_MAX_OUTPUT_DEFAULT` - 30,000 字符
- `getMaxOutputLength()` - 从 `BASH_MAX_OUTPUT_LENGTH` 环境变量返回有效限制

## prefix.ts / specPrefix.ts

### 命令前缀提取

- `getCommandPrefixStatic(command)` - 从命令提取权限规则前缀
  - 使用 tree-sitter 解析器进行基于 AST 的提取
  - 处理包装命令（nice、sudo、timeout）
  - 递归处理嵌套命令
  - 包含环境变量前缀

- `getCompoundCommandPrefixesStatic(command, excludeSubcommand?)` - 复合命令的前缀
  - 在 && / || / ; 上拆分
  - 通过词对齐 LCP 折叠共享相同根的前缀

- `longestCommonPrefix(strings)` - 词对齐 LCP 计算

## shellToolUtils.ts

### Shell 工具实用函数

- `SHELL_TOOL_NAMES` - shell 工具名称数组
- `isPowerShellToolEnabled()` - PowerShell 可用性检查

## 依赖

- `bash/` - Bash 解析和分析
- `envUtils.js` - 环境变量检查
- `platform.js` - 平台检测
- `sessionEnvVars.js` - 会话环境变量
- `tmuxSocket.js` - TMUX 套接字隔离（仅 ant）
- `settings/settings.js` - 默认 shell 解析
