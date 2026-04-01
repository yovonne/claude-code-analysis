# 远程会话工具

## 目的

Claude Code 远程会话（teleport）特性的工具 — 管理云/BYOC 环境、通过 Sessions API 获取和交互远程会话、创建用于引导远程环境的 git bundle，以及选择要使用的环境。

## 位置

- `restored-src/src/utils/teleport/api.ts` — 带重试逻辑、会话 CRUD、事件发送的 API 客户端
- `restored-src/src/utils/teleport/environments.ts` — 环境列表和创建
- `restored-src/src/utils/teleport/environmentSelection.ts` — 从设置中选择环境的逻辑
- `restored-src/src/utils/teleport/gitBundle.ts` — 用于 CCR seed-bundle 引导的 Git bundle 创建和上传

## 主要导出

### API 客户端（`api.ts`）

#### 类型
- `SessionStatus`: `'requires_action' | 'running' | 'idle' | 'archived'`
- `SessionResource`: 带 id、title、status、environment_id、session_context 的完整会话对象
- `SessionContext`: 来源（git/knowledge_base）、cwd、outcomes、model、seed_bundle_file_id、github_pr
- `CodeSession`: 用于 UI 显示的简化会话形状（Zod 验证）
- `ListSessionsResponse`: 分页会话列表
- `RemoteMessageContent`: 远程消息的纯字符串或内容块数组

#### 函数
- `prepareApiRequest()`: 验证 OAuth 访问令牌和组织 UUID；如果未认证则抛出
- `getOAuthHeaders(accessToken)`: 创建标准 OAuth 头（Authorization、Content-Type、anthropic-version）
- `axiosGetWithRetry(url, config?)`: GET 请求，带指数退避重试（2s、4s、8s、16s — 共 5 次尝试）用于瞬态错误（网络错误、5xx）
- `isTransientNetworkError(error)`: 检查 axios 错误是否可重试（无响应或 5xx）
- `fetchCodeSessionsFromSessionsAPI()`: 从 `/v1/sessions` 获取代码会话，转换为 CodeSession 格式
- `fetchSession(sessionId)`: 按 ID 获取单个会话。专门处理 404（未找到）和 401（过期）
- `getBranchFromSession(session)`: 从会话的 git 仓库 outcomes 提取第一个分支名称
- `sendEventToRemoteSession(sessionId, messageContent, opts?)`: 向现有远程会话发送用户消息事件。支持 UUID 用于回声去重。冷启动容器 30 秒超时
- `updateSessionTitle(sessionId, title)`: 通过 PATCH 更新现有远程会话的标题

#### 常量
- `CCR_BYOC_BETA`: `'ccr-byoc-2025-07-29'` — CCR BYOC 端点的 beta 头

### 环境（`environments.ts`）

#### 类型
- `EnvironmentKind`: `'anthropic_cloud' | 'byoc' | 'bridge'`
- `EnvironmentResource`: `{ kind, environment_id, name, created_at, state }`
- `EnvironmentListResponse`: 分页环境列表

#### 函数
- `fetchEnvironments()`: 从 `/v1/environment_providers` 获取可用环境。需要 OAuth 认证
- `createDefaultCloudEnvironment(name)`: 使用 Python 3.11 和 Node 20 创建默认 anthropic_cloud 环境

### 环境选择（`environmentSelection.ts`）

#### 类型
- `EnvironmentSelectionInfo`: `{ availableEnvironments, selectedEnvironment, selectedEnvironmentSource }`

#### 函数
- `getEnvironmentSelectionInfo()`: 获取环境并确定将使用哪个。偏好非 bridge 环境；遵循 `remote.defaultEnvironmentId` 设置，带源跟踪

### Git Bundle（`gitBundle.ts`）

#### 类型
- `BundleUploadResult`: `{ success, fileId?, bundleSizeBytes?, scope?, hasWip? }` 或 `{ success: false, error, failReason? }`
- `BundleScope`: `'all' | 'head' | 'squashed'` — bundle 范围层级
- `BundleFailReason`: `'git_error' | 'too_large' | 'empty_repo'`

#### 函数
- `createAndUploadGitBundle(config, opts?)`: 创建 git bundle 并上传到 Files API。返回用于 SessionContext 上 `seed_bundle_file_id` 的 file_id。

#### Bundle 创建流程（回退链）
1. **`--all`**: Bundle 所有 refs（包括 `refs/seed/stash` 用于 WIP）。最完整但最大
2. **`HEAD`**: 仅 Bundle 当前分支。放弃侧分支/标签但保留完整历史
3. **`squashed-root`**: HEAD 树的单个无父提交（或者 WIP 的 stash 树）。无历史，只有快照

#### WIP 处理
- 使用 `git stash create`（悬挂提交，不触及工作树）→ 存储为 `refs/seed/stash`
- 在 squashed 模式下，WIP 直接烘焙到树中（无法单独 bundle stash ref 否则会拖入历史）
- 未跟踪文件故意不捕获
- 清理：`refs/seed/stash` 和 `refs/seed/root` 在上传后始终删除（甚至崩溃恢复）

#### 大小限制
- 默认：100MB（可通过 `tengu_ccr_bundle_max_bytes` GrowthBook 标志配置）
- 当 bundle 超过限制时通过范围层级

## 设计注意事项

- 所有 API 调用都需要 OAuth 认证（API 密钥不足）— 用户必须运行 `/login`
- API 请求使用指数退避重试用于瞬态错误，但不用于客户端错误（4xx）
- Git bundle 创建在开始前清理崩溃先前运行留下的过时 refs
- 早期检测空仓库（无提交）并返回特定的 `empty_repo` 失败原因
- Bundle 上传使用固定 `relativePath`（`_source_seed.bundle`），以便 CCR 可以找到它
- 环境选择默认偏好非 bridge 环境，并跟踪配置选择的设置源
