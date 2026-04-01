# 配置工具

**文件：** `restored-src/src/utils/config.ts`

## 目的

管理全局和项目级配置存储，包括用户偏好、项目元数据、API 密钥响应和特性标志。提供基于文件的持久化、锁定、缓存和备份机制。

## 主要类型

### GlobalConfig
持久化在 `~/.claude.json`。包含：
- 用户偏好（主题、编辑器模式、自动更新）
- 通知设置
- API 密钥响应（批准/拒绝）
- OAuth 账户信息
- 特性标志和实验跟踪
- 使用统计（内存计数、提示队列等）
- 模型和订阅缓存
- 终端和 IDE 配置

### ProjectConfig
按规范化路径键控的每项目设置：
- `allowedTools` - 此项目允许的工具
- `mcpServers` - MCP 服务器配置
- 信任对话框接受（`hasTrustDialogAccepted`）
- 会话指标（成本、持续时间、令牌）
- Worktree 会话管理
- 远程控制生成模式

## 主要函数

### 全局配置

- `getGlobalConfig()` - 返回带新鲜度观察者的缓存全局配置
- `saveGlobalConfig(updater)` - 通过锁定文件进行线程安全更新，带备份
- `getCustomApiKeyStatus(truncatedKey)` - 返回批准/拒绝/新建状态
- `checkHasTrustDialogAccepted()` - 检查 cwd 和父目录的信任接受
- `isPathTrusted(dir)` - 检查任意目录的信任
- `enableConfigs()` - 在引导后启用配置读取
- `getGlobalConfigWriteCount()` - 返回会话写入计数用于诊断

### 信任管理

- `checkHasTrustDialogAccepted()` - 锁定一次 true 检查，重新评估为 false
- `computeTrustDialogAccepted()` - 遍历 cwd 父目录和项目路径以获取信任
- `isPathTrusted(dir)` - 从任意目录向上遍历以获取信任

### 文件操作

- `saveConfigWithLock()` - 获取文件锁定、读取当前、合并、写带备份
- `saveConfig()` - 写过滤后的配置（移除默认值），权限 0o600
- `getConfig()` - 读取并解析配置文件，带默认值回退
- `writeThroughGlobalConfigCache()` - 写后更新缓存以跳过重新读取

## 缓存策略

- **内存缓存**：`globalConfigCache` 带 mtime 跟踪
- **新鲜度观察者**：`fs.watchFile` 每 1s 轮询外部更改
- **写穿透**：自己的写操作立即更新缓存，带超调的 mtime
- **认证丢失防护**：拒绝将带认证状态的缓存配置覆盖为默认配置

## 安全注意事项

- 配置文件以 0o600 权限写入（仅所有者读写）
- 锁定文件防止并发写入损坏
- 带时间戳的备份（最多 5 个）在 `~/.claude/backups/`，间隔至少 60 秒
- 认证丢失预防：检测重新读取是否返回会擦除 OAuth/入职状态的默认值
- 信任对话框：项目设置不能自动接受信任（仅用户/本地/标志/策略来源）

## 常量

- `GLOBAL_CONFIG_KEYS` - 所有顶级全局配置键的列表
- `PROJECT_CONFIG_KEYS` - 可编辑项目配置键的列表
- `CONFIG_WRITE_DISPLAY_THRESHOLD` - 20 次写入触发警告
- `CONFIG_FRESHNESS_POLL_MS` - 文件观察者的 1000ms 轮询间隔

## 依赖

- `lockfile.ts` - 并发访问的文件锁定
- `envUtils.ts` - 配置目录路径
- `fsOperations.ts` - 文件系统抽象
- `git.ts` - 项目路径的 Git 根检测
- `settings/settings.ts` - 设置集成
