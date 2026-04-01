# PowerShellTool

## 用途

PowerShellTool 是 Claude Code 的 Windows shell 命令执行引擎。它提供统一的接口来运行 PowerShell 命令，具有全面的安全控制、沙箱隔离（POSIX 平台上）、输出流式传输、超时管理、后台执行和 PowerShell 版本感知语法指导。它反映了针对 PowerShell 语义调整的 BashTool 架构。

## 位置

- `restored-src/src/tools/PowerShellTool/PowerShellTool.tsx` — 主要工具定义和执行逻辑（约 1001 行）
- `restored-src/src/tools/PowerShellTool/powershellPermissions.ts` — 权限检查
- `restored-src/src/tools/PowerShellTool/powershellSecurity.ts` — 安全验证器
- `restored-src/src/tools/PowerShellTool/readOnlyValidation.ts` — 只读命令白名单
- `restored-src/src/tools/PowerShellTool/pathValidation.ts` — 路径约束验证
- `restored-src/src/tools/PowerShellTool/modeValidation.ts` — 模式特定权限处理
- `restored-src/src/tools/PowerShellTool/commandSemantics.ts` — 退出码解释
- `restored-src/src/tools/PowerShellTool/destructiveCommandWarning.ts` — 破坏性模式检测
- `restored-src/src/tools/PowerShellTool/gitSafety.ts` — Git 操作安全
- `restored-src/src/tools/PowerShellTool/clmTypes.ts` — 类型定义
- `restored-src/src/tools/PowerShellTool/commonParameters.ts` — 公共参数处理
- `restored-src/src/tools/PowerShellTool/toolName.ts` — 工具名称常量
- `restored-src/src/tools/PowerShellTool/prompt.ts` — 工具提示生成（146 行）
- `restored-src/src/tools/PowerShellTool/UI.tsx` — 终端 UI 渲染

## 主要导出

| 导出 | 描述 |
|--------|-------------|
| `PowerShellTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `PowerShellToolInput` | Zod 推断的输入类型：`{ command, timeout?, description?, run_in_background?, dangerouslyDisableSandbox? }` |
| `Out` | 输出类型：`{ stdout, stderr, interrupted, returnCodeInterpretation?, isImage?, ... }` |
| `PowerShellProgress` | 用于流式更新的进度数据类型 |
| `detectBlockedSleepPattern(command)` | 检测独立的 `Start-Sleep N`（N ≥ 2）模式 |

## 输入/输出 Schema

### 输入 Schema

```typescript
{
  command: string,                     // 必需：PowerShell 命令
  timeout?: number,                    // 可选：毫秒超时（最大 getMaxTimeoutMs()）
  description?: string,                // 可选：人类可读的描述
  run_in_background?: boolean,         // 可选：作为后台任务运行
  dangerouslyDisableSandbox?: boolean, // 可选：绕过沙箱
}
```

当设置 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` 时，`run_in_background` 从 schema 中省略。

### 输出 Schema

```typescript
{
  stdout: string,                         // 命令标准输出
  stderr: string,                         // Shell 重置消息
  interrupted: boolean,                   // 命令是否被中断
  returnCodeInterpretation?: string,      // 非错误退出码的语义含义
  isImage?: boolean,                      // stdout 是否为数据 URI 图像
  persistedOutputPath?: string,           // 磁盘上完整输出的路径
  persistedOutputSize?: number,           // 总输出大小（字节）
  backgroundTaskId?: string,              // 后台任务的 ID
  backgroundedByUser?: boolean,           // 用户按下了 Ctrl+B
  assistantAutoBackgrounded?: boolean,     // 在助手模式中自动后台化
}
```

## PowerShell 命令分类

### 搜索命令（grep 等效）

| Cmdlet | 描述 |
|--------|-------------|
| `Select-String` | grep 等效 |
| `Get-ChildItem`（带 -Recurse） | find 等效 |
| `findstr` | 本机 Windows 搜索 |
| `where.exe` | 本机 Windows which |

### 读取命令

| Cmdlet | 描述 |
|--------|-------------|
| `Get-Content` | cat 等效 |
| `Get-Item` | 文件信息 |
| `Test-Path` | test -e 等效 |
| `Resolve-Path` | realpath 等效 |
| `Get-Process` | ps 等效 |
| `Get-Service` | 系统信息 |
| `Get-ChildItem` | ls/dir 等效 |
| `Get-Location` | pwd 等效 |
| `Get-FileHash` | 校验和 |
| `Get-Acl` | 权限信息 |
| `Format-Hex` | Hexdump 等效 |

### 语义中性命令

| Cmdlet | 描述 |
|--------|-------------|
| `Write-Output` | echo 等效 |
| `Write-Host` | 控制台输出 |

## PowerShell 版本支持

该工具检测 PowerShell 版本并提供版本特定的指导：

### Windows PowerShell 5.1 (powershell.exe)

- 管道链运算符 `&&` 和 `||` 不可用
- 三元运算符（`?:`）、空值合并（`??`）、空值条件（`?.`）运算符不可用
- 避免在本地可执行文件上使用 `2>&1`（将 stderr 包装在 ErrorRecord 中）
- 默认文件编码是 UTF-16 LE（带 BOM）
- `ConvertFrom-Json` 返回 PSCustomObject，而不是 hashtable

### PowerShell 7+ (pwsh)

- 管道链运算符 `&&` 和 `||` 可用
- 三元、空值合并和空值条件运算符可用
- 默认文件编码是 UTF-8 不带 BOM

## 执行管道

```
1. 接收输入
   - 模型返回带有 { command, timeout?, description?, ... } 的 PowerShell tool_use

2. 验证
   - validateInput()：检查 Windows 沙箱策略、阻止的睡眠模式
   - 根据 Zod schema 解析输入

3. 权限检查
   - powershellToolHasPermission() 运行权限链
   - 通过 isReadOnlyCommand() 自动允许只读命令
   - 决策：allow / deny / ask

4. 执行（runPowerShellCommand 生成器）
   - 检测 PowerShell 路径（getCachedPowerShellPath）
   - 确定沙箱模式（平台相关）
   - 设置超时
   - 通过 exec() 生成命令
   - 通过 onProgress 回调流式输出
   - 处理后台化（显式、自动、超时、中断）

5. 结果处理
   - 通过 EndTruncatingAccumulator 累积输出
   - 解释命令语义（interpretCommandResult）
   - 大输出持久化到磁盘
   - 提取并剥离 Claude Code 提示
   - 调整图像输出大小并设置上限
   - Git 操作跟踪
   - 结果返回给模型
```

## Windows 沙箱策略

在原生 Windows 上，沙箱不可用（bwrap/sandbox-exec 仅限 POSIX）。如果企业策略需要沙箱且禁止无沙箱命令，PowerShellTool 拒绝执行：

```
"Enterprise policy requires sandboxing, but sandboxing is not available on native Windows. Shell command execution is blocked on this platform by policy."
```

此检查在 `validateInput()` 和 `call()` 中都运行，作为深度防御。

## 睡眠模式阻止

独立 `Start-Sleep N`（其中 N ≥ 2）作为第一条语句被阻止：

- `Start-Sleep 5` → 被阻止
- `Start-Sleep 5; Get-Process` → 被阻止（建议使用 Monitor 工具或 run_in_background）
- `Start-Sleep 1` → 允许（子 2s 速率限制没问题）
- `Start-Sleep -Milliseconds 500` → 允许
- `Start-Sleep` 在脚本块/子 shell 内 → 允许

## 后台执行

PowerShellTool 支持与 BashTool 相同的后台执行模式：

| 模式 | 触发器 | 行为 |
|------|---------|----------|
| 显式 | `run_in_background: true` | 生成后台任务，立即返回 |
| 用户发起 | Ctrl+B | 将前台转换为后台 |
| 超时 | 命令超过超时 | 如果允许则自动后台化 |
| 助手模式 | 15s 阻塞预算 | 在 KAIROS 模式中自动后台化 |

不允许自动后台化的命令：`Start-Sleep`、`sleep`。

## 交互式命令防止

该工具使用 `-NonInteractive` 标志运行。这些命令被禁止：

- `Read-Host`、`Get-Credential`、`Out-GridView`、`$Host.UI.PromptForChoice`、`pause`
- 破坏性 cmdlet 可能提示确认——添加 `-Confirm:$false`
- 永远不要使用 `git rebase -i`、`git add -i` 或其他交互式编辑器命令

## 多行字符串处理

对于传递给本地可执行文件的多行字符串（例如 git 提交消息）：

```powershell
git commit -m @'
Commit message here.
Second line with $literal dollar signs.
'@
```

- 使用单引号 here-string `@'...'@` 以防止变量扩展
- 闭合 `'@` 必须在第 0 列（无前导空白）
- 使用 `--%` stop-parsing 令牌处理带有 `-`、`@` 等的参数

## 输出处理

- Stderr **不**合并到 stdout（与 BashTool 不同）——它们保持分离
- 从 stdout 剥离前导空行
- 修剪尾随空白
- 大输出（> getMaxOutputLength()）持久化到磁盘
- 通过数据 URI 前缀检测图像输出，调整大小以限制尺寸
- 从输出中提取并剥离 Claude Code 提示

## Git 操作跟踪

通过 `trackGitOperations()` 跟踪 Git 操作——PowerShell 调用 git/gh/glab/curl 作为外部二进制文件，语法与 bash 相同，因此 shell-agnostic 正则检测照常工作。

## 依赖

| 模块 | 用途 |
|--------|---------|
| `utils/Shell.ts` | Shell 执行包装器 |
| `utils/shell/powershellDetection.ts` | PowerShell 路径和版本检测 |
| `utils/sandbox/sandbox-adapter.ts` | 沙箱运行时适配器 |
| `utils/task/TaskOutput.ts` | 任务输出流 |
| `utils/task/diskOutput.ts` | 磁盘输出路径管理 |
| `utils/toolResultStorage.ts` | 大结果持久化 |
| `utils/claudeCodeHints.ts` | 提示提取和剥离 |
| `utils/stringUtils.ts` | EndTruncatingAccumulator |
| `shared/gitOperationTracking.ts` | Git 操作跟踪 |
| `tasks/LocalShellTask/` | 后台任务管理 |
| `BashTool/` | 共享实用工具（图像处理、CWD 重置、实用工具） |
| `services/analytics/` | 事件日志 |
