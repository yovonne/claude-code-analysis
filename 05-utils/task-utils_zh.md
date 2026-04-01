# 任务工具

## 目的

任务工具模块提供 Claude Code 中后台任务的管理框架，包括任务状态管理、输出处理（内存和磁盘）、进度跟踪、SDK 事件发射和输出格式化。它支持两种模式：bash 命令输出（文件模式）和 hook 输出（管道模式）。

## 位置

- `restored-src/src/utils/task/framework.ts` — 任务状态管理、轮询、通知
- `restored-src/src/utils/task/TaskOutput.ts` — 任务输出管理（内存缓冲区 + 磁盘溢出）
- `restored-src/src/utils/task/diskOutput.ts` — 任务输出文件的异步磁盘写入队列
- `restored-src/src/utils/task/outputFormatting.ts` — 输出截断和格式化
- `restored-src/src/utils/task/sdkProgress.ts` — SDK 进度事件发射

## 主要导出

### 框架（`framework.ts`）

#### 常量
- `POLL_INTERVAL_MS`（1000）：所有任务的標準轮询间隔
- `STOPPED_DISPLAY_MS`（3000）：显示被终止任务然后驱逐的持续时间
- `PANEL_GRACE_MS`（30000）：协调器面板中终端 local_agent 任务的宽限期

#### 类型
- `TaskAttachment`: 带 taskId、status、description 和增量摘要的任务状态更新的附件类型

#### 函数
- `updateTaskState<T>(taskId, setAppState, updater)`: AppState 中通用类型安全的任务状态更新
- `registerTask(task, setAppState)`: 注册新任务并发出 `task_started` SDK 事件
- `evictTerminalTask(taskId, setAppState)`: 从 AppState 移除终端任务（完成/失败/终止），带面板宽限期支持
- `getRunningTasks(state)`: 从 AppState 获取所有运行中的任务
- `generateTaskAttachments(state)`: 为有新输出或状态更改的任务生成附件，从磁盘读取增量
- `applyTaskOffsetsAndEvictions(setAppState, updatedTaskOffsets, evictedTaskIds)`: 针对新状态应用输出偏移补丁和驱逐（避免 TOCTOU 竞态）
- `pollTasks(getAppState, setAppState)`: 主要轮询循环 — 生成附件、应用偏移、发送通知

### TaskOutput（`TaskOutput.ts`）

#### 类：`TaskOutput`
Shell 命令输出的单一真实来源，有两种模式：

**文件模式**（bash 命令）：stdout/stderr 通过 stdio fd 直接写入文件。进度通过轮询文件尾部提取。

**管道模式**（hooks）：数据通过 `writeStdout()`/`writeStderr()` 流动，并在内存中缓冲，如果超过限制（默认 8MB）则溢出到磁盘。

#### 关键方法
- `constructor(taskId, onProgress, stdoutToFile, maxMemory?)`: 创建输出处理程序，注册轮询
- `static startPolling(taskId)`、`static stopPolling(taskId)`: 启动/停止文件轮询（从 React 调用）
- `writeStdout(data)`、`writeStderr(data)`: 写入数据（仅管道模式）
- `getStdout()`: 获取 stdout 内容（在文件模式下从文件，在管道模式下从缓冲区）
- `getStderr()`: 获取 stderr 内容（同步，仅从缓冲区）
- `spillToDisk()`: 强制所有缓冲内容到磁盘
- `flush()`: 等待所有待处理的磁盘写入
- `deleteOutputFile()`: 删除输出文件
- `clear()`: 清除所有状态并停止轮询

#### 属性
- `isOverflowed`: 输出已溢出到磁盘时为 true
- `totalLines`、`totalBytes`: 输出统计
- `outputFileRedundant`: 在 `getStdout()` 后为 true（当文件被完全读取时，可以删除）
- `outputFileSize`: 总文件大小（字节）

#### 静态轮询
- 共享的 `#registry` 和 `#activePolling` 映射管理所有文件模式实例
- 单一 `setInterval` 为所有活动任务打勾，读取文件尾部（4096 字节）并推断行数

### 磁盘输出（`diskOutput.ts`）

#### 常量
- `MAX_TASK_OUTPUT_BYTES`（5GB）：任务输出文件的磁盘上限
- `MAX_TASK_OUTPUT_BYTES_DISPLAY`（`'5GB'`）：人类可读的显示字符串

#### 类：`DiskTaskOutput`
使用平面数组写入队列封装异步磁盘写入，由单个排空循环处理。每个块在写入完成后可立即被 GC 回收（避免链式 `.then()` 闭包的内存保留）。

#### 关键方法
- `append(content)`: 排队等待写入的内容；自动刷新
- `flush()`: 等待所有待处理的写入
- `cancel()`: 清除写入队列

#### 模块函数
- `getTaskOutputDir()`: 获取会话特定的输出目录（项目 temp + 会话 ID）
- `getTaskOutputPath(taskId)`: 获取任务的输出文件路径
- `appendTaskOutput(taskId, content)`: 追加内容到任务的磁盘文件
- `flushTaskOutput(taskId)`: 等待待处理的写入
- `evictTaskOutput(taskId)`: 刷新并从内存映射中移除（不删除文件）
- `getTaskOutputDelta(taskId, fromOffset, maxBytes?)`: 读取自上次偏移以来的新内容
- `getTaskOutput(taskId, maxBytes?)`: 获取输出文件的尾部
- `getTaskOutputSize(taskId)`: 获取当前文件大小
- `cleanupTaskOutput(taskId)`: 取消写入并删除输出文件
- `initTaskOutput(taskId)`: 创建空输出文件（带 O_NOFOLLOW 以确保安全）
- `initTaskOutputAsSymlink(taskId, targetPath)`: 将输出文件创建为符号链接

#### 安全性
- `O_NOFOLLOW` 标志防止符号链接跟随攻击
- `O_EXCL` 确保新文件创建在路径已存在时失败

### 输出格式化（`outputFormatting.ts`）

#### 常量
- `TASK_MAX_OUTPUT_UPPER_LIMIT`（160,000）：最大允许输出长度
- `TASK_MAX_OUTPUT_DEFAULT`（32,000）：默认输出长度限制

#### 函数
- `getMaxTaskOutputLength()`: 从 `TASK_MAX_OUTPUT_LENGTH` 环境变量获取配置的限制（有界限）
- `formatTaskOutput(output, taskId)`: 如果太大则截断输出，包括带文件路径的头部

### SDK 进度（`sdkProgress.ts`）

#### 函数
- `emitTaskProgress(params)`: 发出带 taskId、令牌使用、工具使用、持续时间和可选工作流进度的 `task_progress` SDK 事件

## 依赖

- `../CircularBuffer.js` — 最近输出行的循环缓冲区
- `../fsOperations.js` — 文件范围读取和尾部读取
- `../sdkEventQueue.js` — SDK 事件发射
- `../messageQueueManager.js` — 待处理通知队列
- `../shell/outputLimits.js` — 输出长度限制
- `../permissions/filesystem.js` — 项目 temp 目录

## 设计注意事项

- **内存高效的磁盘写入**：`DiskTaskOutput` 使用平面数组队列，带 `splice(0, length)` 就地清除数组，通知 GC 字符串引用可以立即释放。这避免了链式 `.then()` 闭包的内存膨胀问题。
- **TOCTOU 安全**：`applyTaskOffsetsAndEvictions` 针对新鲜的 `prev.tasks`（不是过时的预等待快照）合并补丁，因此并发状态转换不会被覆盖。
- **会话隔离**：输出目录包含会话 ID，以便并发会话不会相互覆盖文件。会话 ID 在首次调用时捕获，不重新读取，因此后台任务在 `/clear` 后继续存在。
- **共享轮询**：所有文件模式 `TaskOutput` 实例共享单一 `setInterval` 打勾，减少多个任务运行时的开销。
- **两种驱逐路径**：终端任务通过 `evictTerminalTask`（在通知时）积极驱逐，并通过 `generateTaskAttachments`（安全网）懒散驱逐。
