# 设置同步服务

## 目的

通过后端 API 跨 Claude Code 环境同步用户设置和记忆文件（settings.json、CLAUDE.md）。支持双向同步：从交互式 CLI 上传，为 CCR（Claude Code Remote）模式下载。

## 位置

`restored-src/src/services/settingsSync/`

### 关键文件

| 文件 | 描述 |
|------|-------------|
| `index.ts` | 主服务实现 — 上传/下载逻辑、API 调用、文件应用 |
| `types.ts` | 同步 API 合同的 Zod 模式和 TypeScript 类型 |

## 关键导出

### 函数 (index.ts)
- `uploadUserSettingsInBackground()`：将本地设置上传到远程（仅交互式 CLI，即发即忘）
- `downloadUserSettings()`：为 CCR 模式从远程下载设置（缓存承诺，可加入）
- `redownloadUserSettings()`：强制新鲜下载，绕过缓存的启动承诺（用于 /reload-plugins）
- `_resetDownloadPromiseForTesting()`：测试专用助手以清除缓存的下载承诺

### 类型 (types.ts)
- `UserSyncData`：来自 GET /api/claude_code/user_settings 的完整响应（userId、version、lastModified、checksum、content）
- `UserSyncContentSchema`：平面键值存储（entries: Record<string, string>）
- `SettingsSyncFetchResult`：获取用户设置的结果（success、data、isEmpty、error、skipRetry）
- `SettingsSyncUploadResult`：上传用户设置的结果（success、checksum、lastModified、error）

### 常量
- `SYNC_KEYS`：同步条目的键映射
  - `USER_SETTINGS`：'~/.claude/settings.json'
  - `USER_MEMORY`：'~/.claude/CLAUDE.md'
  - `projectSettings(projectId)`：'projects/{projectId}/.claude/settings.local.json'
  - `projectMemory(projectId)`：'projects/{projectId}/CLAUDE.local.md'
- `SETTINGS_SYNC_TIMEOUT_MS`：10 秒
- `DEFAULT_MAX_RETRIES`：3
- `MAX_FILE_SIZE_BYTES`：每个文件 500 KB

## 依赖

### 内部依赖
- `auth` — OAuth 令牌管理和刷新
- `git` — 用于项目识别的仓库远程哈希
- `settings/settings` — 读取/写入本地设置文件
- `settings/settingsCache` — 同步后缓存失效
- `claudemd` — 记忆文件缓存管理
- `api/withRetry` — 重试延迟计算
- `analytics` — 同步操作的遥测事件
- `growthbook` — 功能标志评估

### 外部依赖
- `axios` — API 调用的 HTTP 客户端
- `lodash-es/pickBy` — 差异本地与远程条目
- `zod/v4` — 模式验证
- `fs/promises` — 文件 I/O

## 实现细节

### 上传流程（交互式 CLI）

1. 检查资格：功能标志 `UPLOAD_USER_SETTINGS`、growthbook `tengu_enable_settings_sync_push`、交互模式、OAuth
2. 获取当前远程设置
3. 从本地文件构建条目（用户设置、用户记忆、项目设置、项目记忆）
4. 差异本地与远程 — 仅上传更改的条目
5. PUT 更改的条目到 API
6. 记录遥测事件

### 下载流程（CCR 模式）

1. 检查资格：功能标志 `DOWNLOAD_USER_SETTINGS`、growthbook `tengu_strap_foyer`、OAuth
2. 带重试获取远程设置（最多 DEFAULT_MAX_RETRIES）
3. 将远程条目应用到本地文件：
   - 写入 settings.json → resetSettingsCache()
   - 写入 CLAUDE.md → clearMemoryFileCaches()
4. 使用 `markInternalWrite()` 来防止虚假更改检测
5. 缓存承诺：第一次调用开始获取，后续调用加入

### 文件大小限制
- 客户端：每个文件 500 KB（与后端限制匹配）
- 超出限制或为空/仅空格的文件被跳过

### 项目识别
- 使用 git 远程哈希作为项目 ID
- 仅在项目 ID 可用时同步项目特定文件

## 数据流

### 上传

```
交互式 CLI 启动
    ↓
uploadUserSettingsInBackground() (即发即忘)
    ↓
fetchUserSettings() — 获取当前远程状态
    ↓
buildEntriesFromLocalFiles() — 读取本地文件
    ↓
pickBy diff — 查找更改的条目
    ↓
uploadUserSettings() — PUT 更改的条目
    ↓
遥测：tengu_settings_sync_upload_success/failed
```

### 下载

```
CCR 启动 (print.ts runHeadless)
    ↓
downloadUserSettings() — 缓存承诺，可加入
    ↓
fetchUserSettings() — 带重试
    ↓
applyRemoteEntriesToLocal() — 写入文件，使缓存失效
    ↓
markInternalWrite() — 抑制更改检测
    ↓
遥测：tengu_settings_sync_download_success/empty/error
```

## 集成点

- **主 CLI**：`uploadUserSettingsInBackground()` 从 main.tsx preAction 调用
- **CCR 模式**：`downloadUserSettings()` 在 print.ts 顶部触发，在插件安装中等待
- **重载插件**：`redownloadUserSettings()` 由 /reload-plugins 调用，用于会话中期更改
- **设置缓存**：`resetSettingsCache()` 和 `clearMemoryFileCaches()` 在文件写入后
- **更改检测**：`markInternalWrite()` 防止同步写入触发热重载循环

## 配置

- 功能标志：`UPLOAD_USER_SETTINGS`、`DOWNLOAD_USER_SETTINGS`
- Growthbook 标志：`tengu_enable_settings_sync_push`、`tengu_strap_foyer`
- OAuth 要求：带 `user:inference` 范围的第一方 OAuth
- 超时：每个请求 10 秒
- 重试：3 次，带指数退避

## 错误处理

- **失败开放**：上传和下载都记录错误但不阻止启动
- **重试逻辑**：瞬态错误（网络、超时）最多重试 3 次；auth 错误跳过重试
- **缓存共享**：下载使用缓存承诺以便多个调用者共享一次获取
- **文件写入失败**：单个文件写入失败记录但不中止整个同步

## 测试

- `_resetDownloadPromiseForTesting()` 清除测试之间的缓存承诺
- 测试可以模拟功能标志和 OAuth 状态

## 相关模块

- [远程托管设置](./remote-managed-settings.md) — 服务器管理的设置（不同系统）
- [团队记忆同步](./team-memory-sync-service.md) — 团队记忆文件同步

## 备注

- 后端 API 合同：anthropic/anthropic#218817
- 上传是增量的 — 仅发送更改的条目
- 下载覆盖本地文件（服务器获胜）
- 项目特定同步需要用于识别的 git 远程哈希
- 下载承诺被缓存以便即发即忘调用和等待调用共享一次获取
