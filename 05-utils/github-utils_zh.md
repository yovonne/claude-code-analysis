# GitHub 工具

## 目的

单一工具函数，检测 `gh` CLI 是否已安装并通过认证，用于遥测目的。通过使用 `gh auth token`（仅读取本地配置/密钥链）而不是 `gh auth status`（调用 GitHub API）来避免发出网络请求。

## 位置

- `restored-src/src/utils/github/ghAuthStatus.ts`

## 主要导出

### 类型

- `GhAuthStatus`: 判别联合 — `'authenticated' | 'not_authenticated' | 'not_installed'`

### 函数

- `getGhAuthStatus(): Promise<GhAuthStatus>`: 返回 gh CLI 安装 + 认证状态。使用 `which()`（Bun.which — 无子进程）检测安装，然后检查 `gh auth token` 的退出代码检测认证。生成时带 `stdout: 'ignore'`，因此令牌永远不会进入此进程。

## 执行流程

1. 调用 `which('gh')` — 如果未找到，返回 `'not_installed'`
2. 执行 `gh auth token`，带 `stdout: 'ignore'`、`stderr: 'ignore'`、5 秒超时
3. 退出代码 0 → `'authenticated'`；非零 → `'not_authenticated'`

## 设计注意事项

- 使用 `gh auth token` 而不是 `gh auth status`，因为后者向 GitHub API 发出网络请求，而 `auth token` 仅读取本地配置/密钥链
- 令牌永远不被捕获 — stdout 设置为 `'ignore'`
- 5 秒超时防止在密钥链缓慢时挂起
