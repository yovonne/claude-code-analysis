# Git 工具

**文件：** `restored-src/src/utils/git/gitignore.ts`、`gitFilesystem.ts`、`gitConfigParser.ts`

## 目的

在不生成 git 子进程的情况下提供 git 相关操作。涵盖 gitignore 管理、基于文件系统的 git 状态读取和 .git/config 解析。

## gitignore.ts

### 函数

- `isPathGitignored(filePath, cwd)` - 通过 `git check-ignore` 检查路径是否被忽略
  - 退出代码 0 = 忽略，1 = 未忽略，128 = 不在 git 仓库中
  - 对于代码 128 返回 `false`（在 git 仓库外 fail open）

- `getGlobalGitignorePath()` - 返回 `~/.config/git/ignore` 的路径

- `addFileGlobRuleToGitignore(filename, cwd)` - 向全局 gitignore 添加模式
  - 如果已被任何 gitignore 源忽略则跳过
  - 需要时创建目录
  - 追加 `**/filename` 模式

## gitFilesystem.ts

### 目的

直接从 `.git/` 文件读取 git 状态，而不是生成 git 子进程。根据 git 源代码验证正确性。

### 核心函数

- `resolveGitDir(startPath)` - 解析实际的 .git 目录，处理 worktree/子模块
  - 缓存每个路径的结果
  - 处理作为文件（worktree `gitdir:` 指针）或目录的 `.git`

- `readGitHead(gitDir)` - 解析 `.git/HEAD` 以确定分支或 detached SHA
  - 返回 `{ type: 'branch', name }` 或 `{ type: 'detached', sha }`
  - 验证 ref 名称和 SHA 的安全性

- `resolveRef(gitDir, ref)` - 将 git ref 解析为提交 SHA
  - 首先检查松散 ref 文件，然后是 packed-refs
  - 跟随 symref
  - 通过 commonDir 处理 worktree

- `getCommonDir(gitDir)` - 读取 worktree 共享目录的 `commondir` 文件

- `readRawSymref(gitDir, refPath, branchPrefix)` - 读取 symref 并提取分支名称

### 缓存的观察者函数

- `getCachedBranch()` - 通过 `GitFileWatcher` 缓存的当前分支名称
- `getCachedHead()` - 通过 `GitFileWatcher` 缓存的 HEAD SHA
- `getCachedRemoteUrl()` - 通过 `GitFileWatcher` 缓存的远程 origin URL
- `getCachedDefaultBranch()` - 通过 `GitFileWatcher` 缓存的默认分支（main/master）

### GitFileWatcher 类

监视 git 文件更改并缓存派生值：
- 监视 `.git/HEAD`、`.git/config` 和当前分支 ref 文件
- 使用 `fs.watchFile` 轮询，间隔 1 秒（测试中 10ms）
- 文件更改时使缓存失效
- 通过更新 ref 观察者处理分支切换
- 延迟文件 I/O 直到滚动空闲以避免事件循环争用

### 工具函数

- `getHeadForDir(cwd)` - 为任意目录读取 HEAD SHA
- `readWorktreeHeadSha(worktreePath)` - 直接为 worktree 读取 HEAD
- `getRemoteUrlForDir(cwd)` - 为任意目录读取远程 URL
- `isShallowClone()` - 通过 `shallow` 文件存在检查浅克隆
- `getWorktreeCountFromFs()` - 通过 `worktrees/` 目录计数 worktree

### 安全验证

- `isSafeRefName(name)` - 针对路径遍历和 shell 注入验证 ref 名称
  - 允许列表：ASCII 字母数字、`/`、`.`、`_`、`+`、`-`、`@`
  - 拒绝 `..`、前导 `-`、空路径组件、shell 元字符

- `isValidGitSha(s)` - 验证 SHA 为 40 个十六进制字符（SHA-1）或 64 个十六进制字符（SHA-256）

## gitConfigParser.ts

### 目的

`.git/config` 文件的轻量级解析器。根据 git 的 `config.c` 验证。

### 函数

- `parseGitConfigValue(gitDir, section, subsection, key)` - 读取单个配置值
  - 不区分大小写的 section 和 key 匹配
  - 区分大小写的 subsection 匹配
  - 处理带反斜杠转义的引号 subsection

- `parseConfigString(config, section, subsection, key)` - 从内存中字符串解析

### 解析器特性

- Section 头：`[section]` 或 `[section "subsection"]`
- 键值对：`key = value`
- 带转义序列的引号值（`\n`、`\t`、`\\`、`\"`）
- 行内注释（`#` 或 `;`）在引号外
- 反斜杠行继续

## 依赖

- `cwd.js` - 当前工作目录
- `execFileNoThrow.js` - Git 子进程执行（仅 gitignore）
- `cleanupRegistry.js` - 退出时观察者清理
- `debug.js` / `log.js` - 错误日志记录
