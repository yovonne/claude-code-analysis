# Skills 工具

## 目的

Skills 工具模块为 skill 和命令目录提供文件监视和更改检测。它监视用户和项目 skill/命令目录的更改（添加、修改、删除），并在文件更改时触发缓存失效和重新加载，确保 Claude Code 始终使用最新的 skill 定义。

## 位置

- `restored-src/src/utils/skills/skillChangeDetector.ts` — 用于 skills/commands 的文件观察者和更改检测

## 主要导出

### 模块对象
- `skillChangeDetector`: 暴露 `initialize`、`dispose`、`subscribe` 和 `resetForTesting` 方法的单例对象

### 函数

#### `initialize(): Promise<void>`
初始化 skill 和命令目录的文件监视。监视用户 skills（`~/.claude/skills`）、用户命令（`~/.claude/commands`）、项目 skills（`.claude/skills`）、项目命令（`.claude/commands`）以及来自 `--add-dir` 标志的附加目录。使用 chokidar 进行文件监视，带 Bun 特定轮询工作区以避免 PathWatcherManager 死锁。为动态 skills 加载注册回调。

#### `dispose(): Promise<void>`
清理文件观察者、清除待处理的重新加载计时器并重置内部状态。

#### `subscribe`: 用于监听 skill 更改事件的信号订阅函数。

#### `resetForTesting(overrides?): Promise<void>`
重置内部状态用于测试。接受可选的定时覆盖，用于稳定性阈值、轮询间隔、重新加载防抖和 chokidar 间隔。

### 内部常量
- `FILE_STABILITY_THRESHOLD_MS`（1000）：等待文件写入稳定的时间
- `FILE_STABILITY_POLL_INTERVAL_MS`（500）：检查文件稳定性的轮询间隔
- `RELOAD_DEBOUNCE_MS`（300）：将快速 skill 更改事件防抖为单个重新加载
- `POLLING_INTERVAL_MS`（2000）：当 `usePolling` 启用时 chokidar 的轮询间隔
- `USE_POLLING`: 在 Bun 下运行时启用以避免 FSWatcher 死锁

## 依赖

- `chokidar` — 文件监视库
- `../../skills/loadSkillsDir.js` — Skill 缓存管理（`clearSkillCaches`、`getSkillsPath`、`onDynamicSkillsLoaded`）
- `../../commands.js` — 命令缓存管理
- `../attachments.js` — 重置发送的 skill 名称
- `../cleanupRegistry.js` — 清理注册
- `../hooks.js` — 配置更改 hook 执行
- `../signal.js` — 事件信号

## 设计注意事项

- **Bun 死锁解决**：在 Bun 下，chokidar 使用 stat() 轮询而不是原生 FSWatchers，以避免在关闭观察者时发生 PathWatcherManager 死锁（oven-sh/bun#27469）。
- **防抖**：快速文件更改（例如在 git 操作或自动更新期间）被防抖为单个重新加载，以防止可能使 Bun 事件循环死锁的级联缓存清除。
- **批量 hooks**：ConfigChange hooks 一次为一批更改的路径触发，而不是每个文件，避免冗余的 hook matcher 调用。
- **动态 skills**：当加载动态 skills 时，记忆化缓存被清除（但不是完整命令缓存，这会清除刚加载的动态 skills）。
