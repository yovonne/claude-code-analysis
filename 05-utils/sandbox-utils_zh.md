# 沙箱工具

## 目的

沙箱工具模块在 Claude Code 和 `@anthropic-ai/sandbox-runtime` 包之间提供适配器层。它将 Claude Code 设置转换为沙箱运行时配置，管理沙箱生命周期，处理权限规则的路径解析，并提供用于沙箱违规显示的 UI 工具。

## 位置

- `restored-src/src/utils/sandbox/sandbox-adapter.ts` — 主要沙箱管理器适配器
- `restored-src/src/utils/sandbox/sandbox-ui-utils.ts` — 用于违规标签移除的 UI 辅助函数

## 主要导出

### 沙箱适配器（`sandbox-adapter.ts`）

#### 路径解析
- `resolvePathPatternForSandbox(pattern, source)`: 为 sandbox-runtime 解析 Claude Code 特定的路径模式：
  - `//path` → 文件系统根目录的绝对路径（`/path`）
  - `/path` → 相对于设置文件目录
  - `~/path`、`./path`、`path` → 通过 sandbox-runtime 处理
- `resolveSandboxFilesystemPath(pattern, source)`: 从 `sandbox.filesystem.*` 设置解析路径，使用标准路径语义（`/path` = 绝对路径，不是设置相对）

#### 设置转换
- `convertToSandboxRuntimeConfig(settings)`: 将 Claude Code 设置转换为 `SandboxRuntimeConfig` 格式：
  - 从 WebFetch 权限规则提取网络域
  - 从 Edit/Read 规则和 `sandbox.filesystem.*` 设置构建文件系统允许/拒绝列表
  - 始终将当前目录和 Claude 临时目录包含为可写
  - 拒绝写入 settings.json 文件、`.claude/skills` 和裸 git 仓库文件
  - 在 allowWrite 中包含 `--add-dir` 目录
  - 从设置或捆绑二进制文件配置 ripgrep

#### 策略检查
- `shouldAllowManagedSandboxDomainsOnly()`: 检查是否仅应使用托管域
- `areSandboxSettingsLockedByPolicy()`: 检查沙箱设置是否被策略/标志设置覆盖

#### 平台支持
- `isSupportedPlatform()`: 平台支持（macOS、Linux、WSL2+）的缓存检查
- `isPlatformInEnabledList()`: 检查当前平台是否在 `enabledPlatforms` 列表中（未记录的企业设置）
- `getLinuxGlobPatternWarnings()`: 返回在 Linux/WSL 上不起作用的 glob 模式警告（bubblewrap 限制）

#### 沙箱管理器（`SandboxManager` 对象）
实现 `ISandboxManager` 接口：

**生命周期**
- `initialize(sandboxAskCallback?)`: 使用配置初始化沙箱，订阅设置更改，检测 git worktree
- `reset()`: 清除状态、记忆化缓存并重置基础管理器
- `refreshConfig()`: 立即从当前设置更新沙箱配置

**状态检查**
- `isSandboxingEnabled()`: 检查平台支持、依赖项、enabledPlatforms 和 enabled 设置
- `isSandboxEnabledInSettings()`: 直接检查 `sandbox.enabled` 设置
- `isSandboxRequired()`: 检查沙箱是否启用且 `failIfUnavailable` 为 true
- `getSandboxUnavailableReason()`: 返回沙箱无法运行的人类可读原因（明确启用时）
- `checkDependencies()`: 记忆化依赖项检查
- `isAutoAllowBashIfSandboxedEnabled()`: 检查 `autoAllowBashIfSandboxed` 设置
- `areUnsandboxedCommandsAllowed()`: 检查 `allowUnsandboxedCommands` 设置

**配置**
- `setSandboxSettings(options)`: 更新本地沙箱设置（enabled、autoAllowBashIfSandboxed、allowUnsandboxedCommands）
- `getExcludedCommands()`: 获取从沙箱中排除的命令
- `addToExcludedCommands(command, permissionUpdates?)`: 将命令添加到排除列表，从权限建议中提取模式

**执行**
- `wrapWithSandbox(command, binShell?, customConfig?, abortSignal?)`: 用沙箱强制包装命令

**委托给基础管理器**
- `getFsReadConfig()`、`getFsWriteConfig()`、`getNetworkRestrictionConfig()`、`getIgnoreViolations()`、`getAllowUnixSockets()`、`getAllowLocalBinding()`、`getEnableWeakerNestedSandbox()`、`getProxyPort()`、`getSocksProxyPort()`、`getLinuxHttpSocketPath()`、`getLinuxSocksSocketPath()`、`waitForNetworkInitialization()`、`getSandboxViolationStore()`、`annotateStderrWithSandboxFailures()`、`cleanupAfterCommand()`

#### 安全：裸 Git 仓库清理
- `scrubBareGitRepoFiles()`: 在沙箱命令期间在 cwd 植入的裸仓库文件（HEAD、objects、refs、hooks、config），在非沙箱 git 运行之前删除。防止通过 git 的 `is_git_directory()` 检测进行沙箱逃逸。

#### Worktree 检测
- `detectWorktreeMainRepoPath(cwd)`: 检测 cwd 是否为 git worktree 并解析主仓库路径。为会话缓存。

### UI 工具（`sandbox-ui-utils.ts`）

#### 函数
- `removeSandboxViolationTags(text)`: 从文本中移除 `<sandbox_violations>...</sandbox_violations>` XML 块以获得干净的 UI 显示

## 依赖

- `@anthropic-ai/sandbox-runtime` — 核心沙箱运行时包
- `../../bootstrap/state.js` — CWD、原始 CWD、附加目录
- `../settings/` — 设置系统（读取、更新、更改检测）
- `../permissions/` — 权限规则
- `../platform.js` — 平台检测
- `../path.js` — 路径展开
- `../ripgrep.js` — Ripgrep 命令解析
- `../errors.js`、`../debug.js` — 错误处理和日志记录

## 设计注意事项

- **设置到配置转换**：`convertToSandboxRuntimeConfig` 函数是中心转换层，将 Claude Code 的权限规则、网络设置和文件系统策略映射到 sandbox-runtime 的配置格式。它遍历所有设置源（策略、项目、用户、本地）以正确解析路径。
- **安全加固**：多层防御防止沙箱逃逸：
  - 设置文件始终被写拒绝
  - `.claude/skills` 目录被写拒绝（与命令/agent 相同的权限）
  - 命令后清理裸 git 仓库文件以防止基于 git 的逃逸
  - 允许 git worktree 主仓库路径的写访问（需要用于 index.lock）
- **动态配置更新**：设置更改通过 `settingsChangeDetector.subscribe()` 触发自动沙箱配置更新。`refreshConfig()` 函数在权限更改后提供同步即时更新。
- **平台特定行为**：Linux/WSL 不支持 bubblewrap 中的 glob 模式，因此 `getLinuxGlobPatternWarnings()` 向用户公开此限制。`enabledPlatforms` 设置允许企业在推广期间将沙箱限制到特定平台。
- **记忆化策略**：`checkDependencies()` 和 `isSupportedPlatform()` 使用设置对象作为缓存键进行记忆化 — 新设置对象 = 缓存未命中，确保配置更改时自动失效。
- **网络策略执行**：`initialize()` 中的 `wrappedCallback` 在询问级别强制执行 `allowManagedDomainsOnly`，确保所有代码路径（REPL、print/SDK）都遵循策略。
