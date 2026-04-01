# 团队记忆同步服务

## 目的

在本地文件系统和服务器 API 之间同步团队记忆文件。团队记忆按仓库作用域（由 git 远程标识），并在所有认证的组织成员之间共享。支持拉取（服务器获胜）、推送（冲突时本地获胜）和带乐观锁的双向同步。

## 位置

`restored-src/src/services/teamMemorySync/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `index.ts` | 主同步服务 — 拉取、推送、批量、冲突解决、本地文件操作 |
| `types.ts` | 团队记忆同步 API 的 Zod 模式和类型 |
| `watcher.ts` | 带防抖推送和抑制逻辑的文件系统监视器 |
| `secretScanner.ts` | 使用 gitleaks 正则表达式模式的客户端秘密扫描器 |
| `teamMemSecretGuard.ts` | FileWriteTool/FileEditTool 的预写入守卫 |

## 关键导出

### 函数 (index.ts)
- `pullTeamMemory(state, options)`：从服务器拉取，写入本地目录，返回写入的文件
- `pushTeamMemory(state)`：将本地文件推送到服务器，带增量上传和冲突解决
- `syncTeamMemory(state)`：双向同步 — 先拉取后推送
- `isTeamMemorySyncAvailable()`：检查 OAuth 是否可用于同步
- `createSyncState()`：为隔离创建新的 SyncState 对象
- `hashContent(content)`：计算用于内容比较的 sha256 校验和
- `batchDeltaByBytes(delta)`：将增量分割为 PUT 大小的批次，在网关主体限制下

### 函数 (watcher.ts)
- `startTeamMemoryWatcher()`：开始监视团队记忆目录，初始拉取，然后 fs.watch
- `stopTeamMemoryWatcher()`：停止监视器，刷新待处理更改
- `notifyTeamMemoryWrite()`：来自 PostToolUse 钩子的显式推送触发器
- `isPermanentFailure(result)`：确定推送失败是否应抑制重试
- `_resetWatcherStateForTesting(opts)`：测试专用状态重置
- `_startFileWatcherForTesting(dir)`：测试专用真实 fs.watch 启动器

### 函数 (secretScanner.ts)
- `scanForSecrets(content)`：扫描凭证内容，返回匹配项（ruleId、label）
- `redactSecrets(content)`：就地用 [REDACTED] 替换匹配的凭证
- `getSecretLabel(ruleId)`：将 gitleaks 规则 ID 转换为人类可读标签

### 函数 (teamMemSecretGuard.ts)
- `checkTeamMemSecrets(filePath, content)`：预写入守卫 — 如果检测到凭证则返回错误消息

### 类型 (types.ts)
- `TeamMemoryData`：完整 API 响应（organizationId、repo、version、lastModified、checksum、content）
- `TeamMemoryContentSchema`：entries + 可选的 entryChecksums（每键 SHA-256）
- `TeamMemorySyncFetchResult`：带 notModified、isEmpty、checksum 支持的拉取结果
- `TeamMemorySyncPushResult`：带 filesUploaded、conflict、skippedSecrets 的推送结果
- `TeamMemoryHashesResult`：轻量级仅元数据探测结果（?view=hashes）
- `SkippedSecretFile`：由于检测到凭证而跳过的文件（path、ruleId、label）
- `TeamMemoryTooManyEntriesSchema`：带 max_entries、received_entries 的结构化 413 错误

### 类型 (index.ts)
- `SyncState`：可变状态 — lastKnownChecksum、serverChecksums (Map)、serverMaxEntries

### 常量
- `TEAM_MEMORY_SYNC_TIMEOUT_MS`：30 秒
- `MAX_FILE_SIZE_BYTES`：每条目 250 KB
- `MAX_PUT_BODY_BYTES`：200 KB（网关主体大小限制）
- `MAX_RETRIES`：获取 3 次
- `MAX_CONFLICT_RETRIES`：412 冲突解决 2 次
- `DEBOUNCE_MS`：监视器推送防抖 2000ms

## 依赖

### 内部依赖
- `auth` — OAuth 令牌管理
- `git` — GitHub repo slug 标识
- `teamMemPaths` — 团队记忆目录路径验证
- `analytics` — 推送/拉取操作的遥测事件
- `api/withRetry` — 重试延迟计算

### 外部依赖
- `axios` — HTTP 客户端
- `crypto` — SHA-256 哈希
- `fs/promises` — 文件 I/O
- `path` — 路径操作
- `zod/v4` — 模式验证

## 实现细节

### 拉取（服务器获胜）

1. 使用 ETag 缓存从 API 获取（If-None-Match → 304 Not Modified）
2. 404 时：服务器没有数据，清除陈旧的 serverChecksums
3. 200 时：将远程条目写入本地目录
4. 跳过其磁盘内容已匹配的条目（保留 mtime）
5. 根据目录边界验证路径（防止遍历）
6. 并行文件写入以提高性能

### 推送（本地获胜，增量上传）

1. 读取本地文件，扫描凭证（跳过检测到凭证的文件）
2. 对每个本地条目哈希一次
3. 计算增量：仅其本地哈希与 serverChecksums 不同的键
4. 将增量分割为小于 MAX_PUT_BODY_BYTES 的批次
5. 使用 If-Match 头 PUT 批次（乐观锁）
6. 412 冲突时：探测 GET ?view=hashes，刷新 serverChecksums，重新计算增量，重试
7. 每个成功批次后更新 serverChecksums

### 冲突解决

- 使用基于 ETag 的乐观锁（If-Match 头）
- 412 时：廉价探测（?view=hashes）以刷新 serverChecksums 而不下载主体
- 重新计算的增量自然排除并发推送中的服务器来源内容
- 冲突时本地获胜：用户活动编辑不会被静默丢弃
- 最多 2 次冲突重试然后失败

### 文件监视器

- 使用 `fs.watch({recursive: true})` — macOS 上 O(1) fds（APFS/FSEvents），Linux 上 O(子目录)（inotify）
- 防抖推送：最后一次更改后 2s
- 永久失败时抑制推送（4xx 除了 409/429、no_oauth、no_repo）
- 文件删除时清除抑制（too-many-entries 的恢复操作）
- 初始拉取在监视器开始之前运行（磁盘写入不会触发 schedulePush）

### 秘密扫描 (PSR M22174)

- 精心策划的 gitleaks 规则子集，近零假阳性率
- 覆盖：AWS、GCP、Azure、GitHub、GitLab、Slack、Stripe、OpenAI、Anthropic 等
- Anthropic API 密钥前缀在运行时组装以避免在 bundle 中出现字面量
- 带凭证的文件被跳过（不上传），从不记录凭证值
- FileWriteTool/FileEditTool 中的预写入守卫在写入前验证内容

### 批量分割

- 按排序键的贪婪装箱（确定性批次）
- 超出限制的单个条目放入其自己的独立批次
- 每个批次是独立的 PUT — 如果批次 N 失败，1..N-1 已提交

## 数据流

```
文件编辑 → teamMemSecretGuard.ts (预写入检查)
                ↓
          scanForSecrets() — 如果发现凭证则阻止
                ↓
          写入磁盘 → fs.watch 事件 → schedulePush()
                ↓
          防抖 (2s) → executePush()
                ↓
          readLocalTeamMemory() → scanForSecrets() → hashContent()
                ↓
          计算增量 → batchDeltaByBytes()
                ↓
          PUT 批次带 If-Match → 412? → 探测 ?view=hashes → 重试
```

## 集成点

- **PostToolUse 钩子**：`notifyTeamMemoryWrite()` 在记忆文件写入后调用
- **FileWriteTool/FileEditTool**：`checkTeamMemSecrets()` 在写入前验证内容
- **启动**：`startTeamMemoryWatcher()` 初始化同步系统
- **关闭**：`stopTeamMemoryWatcher()` 刷新待处理更改（尽力而为，2s 预算）

## 配置

- 功能标志：`TEAMMEM` 构建标志
- 资格：团队记忆启用、OAuth 可用、存在 GitHub 远程
- 服务器端点：`/api/claude_code/team_memory?repo={owner/repo}`
- 每条目大小上限：250 KB
- 网关主体上限：~256-512 KB（客户端保持在 200 KB 以下）
- 服务器最大条目数：从结构化 413 响应中学习（GB 可根据 org 调整）

## 错误处理

- **永久失败**：4xx（除了 409/429）、no_oauth、no_repo — 抑制重试直到删除
- **瞬态失败**：409（冲突）、429（速率限制）、5xx — 带退避重试
- **冲突解决**：2 次重试，带廉价哈希探测
- **超大文件**：客户端跳过（250 KB 限制）
- **凭证检测**：文件跳过，用户使用规则标签警告（从不是凭证值）
- **失败开放**：监视器错误记录但不崩溃会话

## 测试

- `_resetWatcherStateForTesting()` 重置模块状态，可选择播种 syncState
- `_startFileWatcherForTesting()` 为 fd-count 回归测试启动真实 fs.watch
- 测试为隔离创建新的 SyncState

## 相关模块

- [设置同步](./settings-sync-service.md) — 用户设置同步（不同系统）
- [提取记忆](./extract-memories-service.md) — 记忆提取代理

## 备注

- API 合同：anthropic/anthropic#250711 + #283027 (entryChecksums) + #293258 (结构化 413)
- 文件删除不传播 — 本地删除不会从服务器移除，下次拉取恢复
- 监视器直接使用 fs.watch（不是 chokidar）以避免 500+ 文件时 fd 耗尽
- 推送是只读磁盘（增量+探测，无合并写入）— 无需事件抑制
- 秘密扫描器使用延迟编译的正则表达式缓存 — 在第一次扫描时编译一次
