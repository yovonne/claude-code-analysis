# WebFetchTool

## 目的

WebFetchTool 从指定 URL 获取并提取内容，将 HTML 转换为 markdown，并通过快速的辅助 AI 模型（Haiku）处理内容以回答用户指定的提示。它包括 15 分钟的 LRU 缓存、域名预检黑名单检查、重定向安全验证和二进制内容持久化。该工具设计用于只读网页内容检索，具有多层安全防护以防止数据泄露和访问内部资源。

## 位置

- `restored-src/src/tools/WebFetchTool/WebFetchTool.ts` — 主工具定义和执行逻辑（319 行）
- `restored-src/src/tools/WebFetchTool/utils.ts` — URL 获取、缓存、域名检查、HTML 到 markdown 转换（531 行）
- `restored-src/src/tools/WebFetchTool/prompt.ts` — 工具描述和辅助模型提示生成（47 行）
- `restored-src/src/tools/WebFetchTool/preapproved.ts` — 预批准主机/域名列表（167 行）
- `restored-src/src/tools/WebFetchTool/UI.tsx` — 终端 UI 渲染（72 行）

## 关键导出

### 来自 WebFetchTool.ts

| 导出 | 描述 |
|--------|-------------|
| `WebFetchTool` | 通过 `buildTool()` 构建的完整工具定义 |
| `Output` | Zod 推断的输出类型：`{ bytes, code, codeText, result, durationMs, url }` |

### 来自 utils.ts

| 导出 | 描述 |
|--------|-------------|
| `getURLMarkdownContent(url, abortController)` | 获取 URL 内容，将 HTML 转换为 markdown，返回缓存或新内容 |
| `applyPromptToMarkdown(prompt, content, signal, ...)` | 将内容和提示发送到 Haiku 模型进行处理 |
| `validateURL(url)` | 验证 URL 格式、长度和安全性（无凭证、公共主机名） |
| `checkDomainBlocklist(domain)` | 通过 Anthropic 的黑名单 API 检查域名 |
| `isPermittedRedirect(originalUrl, redirectUrl)` | 验证重定向安全性（相同来源、仅 www. 变化） |
| `getWithPermittedRedirects(url, signal, redirectChecker)` | 带受控重定向跟随的 HTTP GET |
| `isPreapprovedUrl(url)` | 检查 URL 是否在预批准域名列表上 |
| `clearWebFetchCache()` | 清除 URL 和域名检查缓存 |
| `FetchedContent` | 类型：`{ content, bytes, code, codeText, contentType, persistedPath?, persistedSize? }` |

### 来自 preapproved.ts

| 导出 | 描述 |
|--------|-------------|
| `PREAPPROVED_HOSTS` | ~100+ 个代码相关内容的预批准域名集 |
| `isPreapprovedHost(hostname, pathname)` | 检查主机/路径是否预批准（支持路径作用域条目） |

## 输入/输出模式

### 输入模式

```typescript
{
  url: string,     // 必需：要获取内容的 URL
  prompt: string,  // 必需：在获取内容上运行的提示
}
```

### 输出模式

```typescript
{
  bytes: number,       // 获取内容的大小（字节）
  code: number,        // HTTP 响应代码
  codeText: string,    // HTTP 响应代码文本
  result: string,      // 将提示应用于内容后的处理结果
  durationMs: number,  // 获取和处理所用时间
  url: string,         // 被获取的 URL
}
```

## 内容获取管道

### 执行流程

```
1. 接收输入
   - Model 返回 WebFetch tool_use，附带 { url, prompt }

2. 验证
   - validateInput()：检查 URL 是否可解析
   - URL 验证：长度 ≤ 2000，无用户名/密码，公共主机名（≥ 2 部分）

3. 权限检查
   - checkPermissions()：
     a. 预批准主机检查 → 自动允许
     b. 拒绝规则查找 → 拒绝
     c. 询问规则查找 → 询问（带建议）
     d. 允许规则查找 → 自动允许
     e. 无规则 → 询问（默认）

4. 内容获取（getURLMarkdownContent）
   a. 缓存查找（LRU，15 分钟 TTL，50MB 最大）
   b. URL 验证（validateURL）
   c. HTTP→HTTPS 升级（如需要）
   d. 域名黑名单预检（api.anthropic.com）
   e. 带受控重定向的 HTTP GET（最多 10 跳）
   f. 内容类型检测（HTML vs 文本 vs 二进制）
   g. 通过 Turndown 进行 HTML→Markdown 转换
   h. 二进制内容持久化到磁盘
   i. 缓存存储

5. 提示应用
   a. 预批准 + markdown + <100K 字符 → 返回原始内容
   b. 否则 → 使用提示发送到 Haiku 模型
   c. 在模型调用前将内容截断为 100K 字符

6. 结果处理
   - 组装输出，包含 bytes、code、result、duration
   - 如果持久化则附加二进制内容路径
   - 返回结果给模型
```

### 缓存策略

两级 LRU 缓存：

| 缓存 | 键 | TTL | 最大大小 | 目的 |
|-------|-----|-----|----------|---------|
| `URL_CACHE` | 完整 URL | 15 分钟 | 50 MB | 获取的内容 |
| `DOMAIN_CHECK_CACHE` | 主机名 | 5 分钟 | 128 条目 | 黑名单检查结果 |

域名检查缓存仅存储 `allowed` 状态 — 被阻止/失败的条目在每次尝试时重新检查。这在获取同一域名的多个路径时防止冗余的 HTTP 往返。

### 重定向处理

重定向采用严格的安全控制处理：

- **最大跳数**：10（防止无限循环）
- **允许的重定向**：仅同源重定向（允许 www. 变化）
- **协议必须匹配**：不允许 http→https 或 https→http 重定向
- **端口必须匹配**：不允许端口更改
- **无凭证**：拒绝带有用户名/密码的重定向 URL
- **跨主机重定向**：作为 `REDIRECT DETECTED` 消息返回给模型，并附带重新获取的说明

## 安全控制

### URL 验证

| 检查 | 规则 |
|-------|------|
| 长度 | ≤ 2000 个字符 |
| 格式 | 必须是有效、可解析的 URL |
| 凭证 | URL 中无用户名或密码 |
| 主机名 | 必须可公开解析（≥ 2 个点分隔部分） |
| 协议 | HTTP 自动升级到 HTTPS |

### 域名黑名单预检

在获取之前，主机名通过 Anthropic 的域名黑名单 API 检查：

```
GET https://api.anthropic.com/api/web/domain_info?domain=<hostname>
```

- **allowed**（`can_fetch: true`）：继续获取，缓存结果
- **blocked**（`can_fetch: false`）：抛出 `DomainBlockedError`
- **check_failed**：抛出 `DomainCheckFailedError`
- **跳过**：如果 `settings.skipWebFetchPreflight` 为 true（企业客户具有限制性安全策略）

### 预批准主机

~100+ 个域名被预批准自动访问，无需权限提示。类别包括：

- **Anthropic**：platform.claude.com、modelcontextprotocol.io 等
- **编程语言**：docs.python.org、developer.mozilla.org、doc.rust-lang.org 等
- **框架**：react.dev、angular.io、django、flask、spring 等
- **云/DevOps**：docs.aws.amazon.com、kubernetes.io、docker.com 等
- **数据库**：postgresql.org、mongodb.com、redis.io 等
- **数据/ML**：huggingface.co、kaggle.com、pytorch.org 等

路径作用域条目（例如 `github.com/anthropics`）强制路径段边界以防止前缀匹配攻击。

### 出口代理检测

通过 `X-Proxy-Error: blocked-by-allowlist` 头检测企业出口代理阻塞，并抛出带有结构化错误信息的 `EgressBlockedError`。

### 内容长度限制

| 限制 | 值 | 目的 |
|-------|-------|-------------|
| `MAX_URL_LENGTH` | 2000 字符 | 防止过大 URL |
| `MAX_HTTP_CONTENT_LENGTH` | 10 MB | 防止资源耗尽 |
| `MAX_MARKDOWN_LENGTH` | 100K 字符 | 在 Haiku 模型调用前截断内容 |
| `FETCH_TIMEOUT_MS` | 60 秒 | 防止在慢速服务器上挂起 |
| `DOMAIN_CHECK_TIMEOUT_MS` | 10 秒 | 防止慢速黑名单检查 |
| `MAX_REDIRECTS` | 10 | 防止重定向循环 |

### 二进制内容处理

二进制内容（PDF、图片等）：
1. 保存到磁盘，带 MIME 衍生的扩展名
2. 仍解码为 UTF-8 并发送到 Haiku（PDF 有足够的 ASCII 结构进行摘要）
3. 持久化文件路径被附加到结果中，以便模型可以检查原始文件

## 权限处理

### 权限决策逻辑

```
1. 主机预批准？ → 允许（原因："Preapproved host"）
2. 有此域名的拒绝规则？ → 拒绝
3. 有此域名的询问规则？ → 询问（带允许建议）
4. 有此域名的允许规则？ → 允许
5. 无匹配规则 → 询问（带允许建议）
```

权限规则按 `domain:<hostname>` 键控，从输入 URL 派生。

### 建议系统

当请求权限时，系统建议添加允许规则：
```json
{
  "type": "addRules",
  "destination": "localSettings",
  "rules": [{ "toolName": "WebFetch", "ruleContent": "domain:<hostname>" }],
  "behavior": "allow"
}
```

## 提示应用

### 辅助模型处理

对于非预批准内容或超过大小限制的内容，获取的 markdown 通过 Haiku 模型处理：

```
Web page content:
---
<markdown content（截断为 100K 字符）>
---

<user prompt>

<基于预批准状态的指南>
```

**预批准域名指南**：基于内容给出简洁响应，包括代码示例和文档摘录。

**非预批准域名指南**：引用最多 125 个字符，引用精确语言，不评论合法性，不引用歌词。

## 配置

### 环境变量

| 变量 | 目的 |
|----------|---------|
| `USER_TYPE` | `'ant'` 记录主机名分析以进行内部监控 |

### 设置

| 设置 | 类型 | 默认 | 描述 |
|---------|------|---------|---------|
| `skipWebFetchPreflight` | boolean | false | 跳过域名黑名单检查（企业使用） |

## 依赖

### 内部

| 模块 | 目的 |
|--------|---------|
| `services/api/claude.js` | Haiku 模型查询 |
| `utils/mcpOutputStorage.js` | 二进制内容持久化 |
| `utils/settings/settings.js` | 设置访问 |
| `utils/http.js` | User-Agent 生成 |
| `utils/format.js` | 文件大小格式化 |
| `utils/permissions/permissions.js` | 权限规则查找 |
| `services/analytics/` | 事件记录 |

### 外部

| 包 | 目的 |
|---------|---------|
| `axios` | 用于获取 URL 的 HTTP 客户端 |
| `lru-cache` | LRU 缓存实现 |
| `turndown` | HTML 到 Markdown 转换 |
| `zod/v4` | 输入/输出模式验证 |
| `react` | UI 渲染 |

## 数据流

```
Model 请求
    |
    v
WebFetchTool 输入 { url, prompt }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 验证                                                               │
│ - validateInput()：URL 可解析                                      │
│ - validateURL()：长度 ≤ 2000，无凭证，公共主机名                  │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 权限检查                                                           │
│  1. 预批准主机 → 允许                                            │
│  2. 拒绝规则 → 拒绝                                               │
│  3. 询问规则 → 询问                                              │
│  4. 允许规则 → 允许                                               │
│  5. 无规则 → 询问                                               │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 内容获取（getURLMarkdownContent）                                  │
│                                                                 │
│  1. 缓存查找（URL_CACHE，15 分钟 TTL，50MB 最大）                │
│  2. HTTP→HTTPS 升级                                              │
│  3. 域名黑名单检查（api.anthropic.com）                           │
│  4. 带重定向控制的 HTTP GET（最多 10 跳）                        │
│  5. 内容类型检测                                                  │
│  6. HTML→Markdown（Turndown）或原始直通                          │
│  7. 二进制持久化到磁盘                                            │
│  8. 缓存存储                                                      │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ 提示应用                                                          │
│                                                                 │
│  预批准 + markdown + <100K → 返回原始                              │
│  否则 → 使用提示的 Haiku 模型                                      │
│  内容截断为 100K 字符                                              │
└─────────────────────────────────────────────────────────────────┘
    |
    v
输出 { bytes, code, codeText, result, durationMs, url }
    |
    v
Model 响应（tool_result 块）
```
