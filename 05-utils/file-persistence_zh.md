# 文件持久化

## 目的

在每个回合结束时协调将会话输出文件持久化到 Files API。支持两种模式：BYOC（自带云）扫描本地文件系统并上传修改的文件，以及云（1P）模式从扩展属性读取文件 ID（rclone 处理同步）。仅对具有适当环境配置的远程会话用户激活。

## 位置

- `restored-src/src/utils/filePersistence/filePersistence.ts` — 主要编排逻辑
- `restored-src/src/utils/filePersistence/outputsScanner.ts` — 输出目录扫描器、修改文件检测
- `restored-src/src/utils/filePersistence/types.ts` — 共享类型（两个模块引用）

## 主要导出

### 编排（`filePersistence.ts`）

#### 函数

- `runFilePersistence(turnStartTime, signal?)`: 主要入口点。检查环境种类（仅在 BYOC 模式下运行）、收集配置（会话访问令牌、会话 ID）、扫描修改的文件、上传到 Files API，并返回带有成功/失败列表的事件数据。
- `executeFilePersistence(turnStartTime, signal, onResult)`: 运行持久化并通过回调发出结果的包装器。内部处理错误。
- `isFilePersistenceEnabled()`: 检查特性标志、环境种类、会话访问令牌和 `CLAUDE_CODE_REMOTE_SESSION_ID` 以确定持久化是否激活。

### 扫描器（`outputsScanner.ts`）

#### 函数

- `findModifiedFiles(turnStartTime, outputsDir)`: 递归扫描输出目录中自回合开始以来修改的文件。为提高效率使用并行 `stat()` 调用。为安全跳过符号链接。
- `getEnvironmentKind()`: 读取 `CLAUDE_CODE_ENVIRONMENT_KIND` 环境变量；返回 `'byoc'`、``'anthropic_cloud'` 或 `null`。
- `logDebug(message)`: 文件持久化模块的共享调试日志记录器。

## 执行流程（BYOC 模式）

1. 检查环境种类 — 如果不是 `'byoc'` 则返回 `null`
2. 验证会话访问令牌和 `CLAUDE_CODE_REMOTE_SESSION_ID`
3. 扫描 `{cwd}/{sessionId}/outputs` 目录中 `mtime >= turnStartTime` 的文件
4. 强制执行文件计数限制（跳过解析到输出目录外部的文件以确保安全）
5. 通过 `uploadSessionFiles()` 并行上传文件，带可配置并发
6. 将成功上传（带文件 ID）与失败分离
7. 为开始/完成/错误发出分析事件

## 设计注意事项

- 文件持久化受特性标志 `FILE_PERSISTENCE` 门控，需要远程会话上下文 — 普通 CLI 用户从不触发它
- 安全：扫描期间跳过符号链接；拒绝解析到输出目录外部的文件
- 云模式（1P）当前是空操作 — 基于 xattr 的文件 ID 读取已计划但尚未实现
- 文件计数限制防止用大批量文件压垮 Files API
