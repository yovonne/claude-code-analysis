# BriefTool

## 目的

BriefTool（用户面向名称：`SendUserMessage`）是 Claude 向用户发送消息的主要输出通道。它是 Claude 传达答案、状态更新、检查点和主动通知的机制。此工具外的文本在详情视图中可见，但大多数用户不会打开它 — 答案在这里。该工具支持 markdown 格式的消息、文件附件（图像、截图、diffs、日志），并区分正常回复和主动通知以供下游路由。

## 位置

- `restored-src/src/tools/BriefTool/BriefTool.ts`
- `restored-src/src/tools/BriefTool/prompt.ts`
- `restored-src/src/tools/BriefTool/attachments.ts`
- `restored-src/src/tools/BriefTool/upload.ts`
- `restored-src/src/tools/BriefTool/UI.tsx`

## 关键导出

### 函数

- `BriefTool`：通过 `buildTool()` 构建的主要工具定义 — 处理消息传递、附件解析和状态路由
- `isBriefEntitled()`：权限检查 — 用户是否被允许使用 Brief（结合功能标志、GrowthBook 门控和环境绕过）
- `isBriefEnabled()`：统一激活门控 — Brief 在当前会话中是否实际激活（需要权限 + 选择加入）
- `validateAttachmentPaths()`：验证附件文件路径存在、是常规文件且可访问
- `resolveAttachments()`：将附件路径解析为元数据（大小、isImage）并可选上传到云以供 web 查看器使用
- `uploadBriefAttachment()`：将单个附件上传到 REPL bridge 的私有 API 以进行 web 预览
- `renderToolUseMessage()`、`renderToolResultMessage()`：用于终端显示的 UI 渲染函数

### 类型

- `Output`：`{ message: string, attachments?: ResolvedAttachment[], sentAt?: string }`
- `ResolvedAttachment`：`{ path: string, size: number, isImage: boolean, file_uuid?: string }`
- `BriefUploadContext`：`{ replBridgeEnabled: boolean, signal?: AbortSignal }`

### 常量

- `BRIEF_TOOL_NAME`：`'SendUserMessage'`
- `LEGACY_BRIEF_TOOL_NAME`：`'Brief'` — 向后兼容的旧有线名称
- `DESCRIPTION`：`'Send a message to the user'`
- `BRIEF_TOOL_PROMPT`：LLM 如何使用工具的说明
- `BRIEF_PROACTIVE_SECTION`：关于与用户对话的系统提示部分
- `KAIROS_BRIEF_REFRESH_MS`：`300,000`（5 分钟）— GrowthBook 功能刷新间隔

## 依赖

### 内部依赖

- `prompt.ts` — 工具名称、描述、提示和主动部分文本
- `attachments.ts` — 附件验证、解析和上传编排
- `upload.ts` — 到 REPL bridge 私有 API 的文件上传（基于 axios 的多部分上传）
- `UI.tsx` — 用于在终端中渲染消息和附件列表的 React 组件
- `bridge/bridgeConfig.js` — 用于 bridge 上传的 OAuth 令牌和基础 URL
- `imageResizer.js` — 用于粘贴图像的图像调整大小和下采样
- `imageStore.js` — 粘贴内容的图像缓存和存储
- `formatBriefTimestamp.js` — 仅 brief 视图的时间戳格式
- `Markdown.js` — 消息内容的 Markdown 渲染

### 外部依赖

- `zod/v4` — 带严格对象的 Schema 验证
- `react` — 使用 React Compiler 的 UI 组件渲染
- `bun:bundle` — 功能标志门控（`KAIROS`、`KAIROS_BRIEF`、`BRIDGE_MODE`）
- `axios` — 用于 bridge 上传的 HTTP 客户端
- `crypto` — 用于多部分表单边界的 UUID 生成
- `fs/promises` — 文件 stat 和读取操作

## 实现细节

### 输入 Schema

```typescript
z.strictObject({
  message: z.string()
    .describe('The message for the user. Supports markdown formatting.'),
  attachments: z.array(z.string())
    .optional()
    .describe('Optional file paths (absolute or relative to cwd) to attach.'),
  status: z.enum(['normal', 'proactive'])
    .describe("Use 'proactive' when surfacing something the user hasn't asked for."),
})
```

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `message` | `string` | 消息文本（支持 markdown） |
| `attachments` | `string[]`（可选） | 图像、截图、diffs、日志的文件路径 |
| `status` | `'normal' \| 'proactive'` | 路由意图 — 回复 vs. 主动通知 |

### 输出 Schema

```typescript
z.object({
  message: z.string().describe('The message'),
  attachments: z.array(z.object({
    path: z.string(),
    size: z.number(),
    isImage: z.boolean(),
    file_uuid: z.string().optional(),
  })).optional().describe('Resolved attachment metadata'),
  sentAt: z.string().optional()
    .describe('ISO timestamp at tool execution time'),
})
```

**注意：** `attachments` 和 `sentAt` 必须保持可选 — 恢复的会话原样重放预附件/预 sentAt 输出，所需字段会在恢复时使 UI 渲染器崩溃。

### 功能门控

BriefTool 有两层门控系统：

#### 第 1 层：权限（`isBriefEntitled()`）

确定用户是否**被允许**使用 Brief：

```
feature('KAIROS') || feature('KAIROS_BRIEF')
    ? getKairosActive() || isEnvTruthy(CLAUDE_CODE_BRIEF) || getFeatureValue('tengu_kairos_brief')
    : false
```

| 条件 | 效果 |
|-----------|--------|
| `KAIROS` 或 `KAIROS_BRIEF` 功能关闭 | 始终为 false（外部构建中死代码消除） |
| `getKairosActive()` | 助手模式处于活动状态 |
| `CLAUDE_CODE_BRIEF` 环境变量 | 强制授予权限用于开发/测试 |
| `tengu_kairos_brief` GrowthBook 标志 | 远程功能标志（每 5 分钟刷新） |

#### 第 2 层：激活（`isBriefEnabled()`）

确定 Brief 在当前会话中是否**实际激活**：

```
(feature('KAIROS') || feature('KAIROS_BRIEF'))
    ? (getKairosActive() || getUserMsgOptIn()) && isBriefEntitled()
    : false
```

激活需要通过以下方式明确选择加入：
- `--brief` CLI 标志
- 设置中的 `defaultView: 'chat'`
- `/brief` 斜杠命令
- `/config` defaultView 选择器
- `--tools` / SDK `tools` 选项中的 `SendUserMessage`
- `CLAUDE_CODE_BRIEF` 环境变量（开发/测试绕过）

助手模式（`kairosActive`）绕过选择加入要求，因为其系统提示硬编码"你必须使用 SendUserMessage"。

**Kill-switch 行为：** GrowthBook 门控在每次调用时重新检查。在会话中途翻转 `tengu_kairos_brief` 会在 5 分钟刷新后禁用该工具，即使对于选择加入的会话也是如此。

### 工具属性

| 属性 | 值 | 理由 |
|----------|-------|-----------|
| `isReadOnly()` | `true` | 不修改文件系统 |
| `isConcurrencySafe()` | `true` | 可以安全并发调用 |
| `userFacingName()` | `''`（空） | 从 UI 中删除工具 chrome — 消息显示为纯文本 |
| `shouldDefer` | 未设置 | 不延迟执行 |
| `maxResultSizeChars` | `100,000` | 最大结果大小 |

### `call()` 方法

```typescript
async call({ message, attachments, status }, context) {
  const sentAt = new Date().toISOString()
  logEvent('tengu_brief_send', {
    proactive: status === 'proactive',
    attachment_count: attachments?.length ?? 0,
  })
  if (!attachments || attachments.length === 0) {
    return { data: { message, sentAt } }
  }
  const appState = context.getAppState()
  const resolved = await resolveAttachments(attachments, {
    replBridgeEnabled: appState.replBridgeEnabled,
    signal: context.abortController.signal,
  })
  return { data: { message, attachments: resolved, sentAt } }
}
```

**流程：**
1. 捕获 `sentAt` 时间戳
2. 记录带主动标志和附件计数的分析事件
3. 如果没有附件，立即返回消息 + 时间戳
4. 如果有附件，解析它们（stat 文件，可选上传）
5. 返回消息 + 解析的附件 + 时间戳

### 附件系统

#### 验证（`validateAttachmentPaths()`）

验证每个附件路径：
1. 相对于 cwd 展开相对路径
2. Stat 文件以验证存在且是常规文件
3. 处理错误：
   - `ENOENT`：文件不存在（在错误消息中包含 cwd）
   - `EACCES`/`EPERM`：权限被拒绝
   - 其他错误：重新抛出

#### 解析（`resolveAttachments()`）

两阶段解析：
1. **Stat 阶段**（串行、本地、快速）：Stat 每个文件以通过扩展名检测图像类型获取大小
2. **上传阶段**（并行、网络、慢）：如果启用则上传到 bridge

```typescript
// 阶段 1：Stat 所有文件
const stated: ResolvedAttachment[] = []
for (const rawPath of rawPaths) {
  const fullPath = expandPath(rawPath)
  const stats = await stat(fullPath)
  stated.push({
    path: fullPath,
    size: stats.size,
    isImage: IMAGE_EXTENSION_REGEX.test(fullPath),
  })
}

// 阶段 2：如果 bridge 启用则上传
if (feature('BRIDGE_MODE')) {
  const shouldUpload = uploadCtx.replBridgeEnabled || isEnvTruthy(process.env.CLAUDE_CODE_BRIEF_UPLOAD)
  const { uploadBriefAttachment } = await import('./upload.js')
  const uuids = await Promise.all(stated.map(a => uploadBriefAttachment(a.path, a.size, { ... })))
  return stated.map((a, i) => uuids[i] === undefined ? a : { ...a, file_uuid: uuids[i] })
}
return stated
```

上传模块在 `BRIDGE_MODE` 功能守卫内动态导入，以便 `upload.ts`（及其 axios、crypto、zod、auth utils、MIME map 依赖）从非 bridge 构建中完全 tree-shaken。

#### 上传（`uploadBriefAttachment()`）

将单个文件上传到 REPL bridge 的私有 API：

**端点：** `POST {baseUrl}/api/oauth/file_upload`

**流程：**
1. 检查 `BRIDGE_MODE` 功能标志
2. 验证 bridge 已启用
3. 检查文件大小是否超过 `MAX_UPLOAD_BYTES`（30 MB）
4. 获取 OAuth 访问令牌
5. 读取文件内容
6. 构建带正确 MIME 类型的多部分表单
7. 使用 Bearer auth POST 到上传端点
8. 解析响应的 `file_uuid`

**MIME 类型检测：**
```typescript
const MIME_BY_EXT: Record<string, string> = {
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.gif': 'image/gif',
  '.webp': 'image/webp',
}
// 默认：'application/octet-stream'
```

**优雅降级：** 每个失败返回 `undefined` 而非抛出。附件仍携带 `{path, size, isImage}`，因此本地渲染器不受影响。

**尽力失败：**
- 无 OAuth 令牌 → 跳过
- 文件太大 → 跳过（日志记录大小和限制）
- 读取错误 → 跳过
- 网络错误 → 跳过
- 非 201 响应 → 跳过
- 无效响应形状 → 跳过

### 结果格式化

`mapToolResultToToolResultBlockParam()` 为 LLM 格式化结果：

```
Message delivered to user. (N attachment(s) included)
```

附件计数后缀仅在存在附件时添加。

## UI 渲染

### renderToolUseMessage

返回空字符串 — 该工具没有可见的"use"chrome，因为 `userFacingName()` 为空。

### renderToolResultMessage

以三种不同的视图模式渲染传递的消息：

#### Transcript 模式（Ctrl+O）

模型文本不过滤 — 保留黑圆标记，以便 `SendUserMessage` 在周围文本块中在视觉上突出：

```
⏺ <markdown content>
    [attachment list]
```

#### Brief-Only 模式（聊天视图）

带 2 列缩进的"Claude"标签，匹配应用于用户输入的"You"标签：

```
Claude [timestamp]
  <markdown content>
  [attachment list]
```

#### 默认视图

无 gutter 标记 — 读取为纯文本。`userFacingName() === ''` 导致 `AssistantToolUseMessage` 渲染 null（无工具 chrome），而 `dropTextInBriefTurns` 隐藏会出现在此内容之前的冗余 assistant 文本：

```
<markdown content>
[attachment list]
```

### AttachmentList 组件

渲染附加文件列表：

```
· [image] /path/to/screenshot.png (1.2 MB)
· [file]  /path/to/log.txt (45 KB)
```

每个附件显示：
- 类型指示器：`[image]` 或 `[file]`
- 显示路径（尽可能相对）
- 格式化的文件大小

## 通信指南（来自 BRIEF_PROACTIVE_SECTION）

系统提示部分 `BRIEF_PROACTIVE_SECTION` 提供了 Claude 如何通信的指导：

1. **每个回复都通过 BriefTool** — 即使是"hi"和"thanks"
2. **在工作前确认** — "On it — checking the test output" 防止用户盯着微调器
3. **较长工作的模式**：确认 → 工作 → 结果
4. **检查点** — 当发生有用的决策、惊喜、阶段边界时发送更新
5. **保持消息紧凑** — 决策、文件:行、PR 编号
6. **始终使用第二人称** — "your config"，而非第三人称
7. **无填充** — 跳过"running tests..." — 检查点通过携带信息来赚取其位置

### 状态字段路由

| 状态 | 使用时机 |
|--------|-------------|
| `normal` | 回复用户刚刚问的内容 |
| `proactive` | 呈现主动的内容 — 离开时任务完成、阻碍者、主动状态更新 |

下游路由使用此字段确定如何向用户呈现消息。

## 配置

### 环境变量

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_CODE_BRIEF` | 强制授予 Brief 权限（开发/测试绕过） |
| `CLAUDE_CODE_BRIEF_UPLOAD` | 在作为子进程运行时启用附件上传（例如 cowork desktop bridge） |

### 功能标志

| 标志 | 用途 |
|------|---------|
| `KAIROS` | 启用助手模式（包含 Brief） |
| `KAIROS_BRIEF` | 独立于完整助手模式启用 Brief |
| `BRIDGE_MODE` | 启用 REPL bridge 以进行云附件上传 |

## 错误处理

### 附件验证错误

| 错误代码 | 条件 |
|------------|-----------|
| 1 | 附件不是常规文件（例如目录） |
| 1 | 附件不存在（ENOENT） |
| 1 | 附件不可访问（权限被拒绝） |

### 上传失败

上传失败是静默的 — 它们为 `file_uuid` 返回 `undefined`，附件保留其本地路径。这确保：
- 本地终端渲染不受影响
- 同机器桌面渲染工作
- Web 查看器看到带有预览的惰性附件卡

### 恢复安全性

`attachments` 和 `sentAt` 在输出 schema 中都是可选的，因为：
- 恢复的会话原样重放预附件输出
- 必需字段会在恢复时使 UI 渲染器崩溃
- 预 sentAt 输出没有时间戳

## 数据流

```
Model calls SendUserMessage with { message, attachments?, status? }
    │
    ▼
call()
    ├── Capture sentAt timestamp
    ├── Log analytics event
    │
    ├── [no attachments] ──→ return { message, sentAt }
    │
    └── [has attachments] ──→ resolveAttachments()
        │
        ├── validateAttachmentPaths() — stat each file
        │
        ├── Build ResolvedAttachment[] with path, size, isImage
        │
        └── [BRIDGE_MODE enabled] ──→ uploadBriefAttachment() (parallel)
            │
            ├── Check size limit (30 MB)
            ├── Get OAuth token
            ├── Read file
            ├── Build multipart form
            ├── POST to /api/oauth/file_upload
            └── Extract file_uuid from response
        │
        ▼
    return { message, attachments: resolved[], sentAt }
    │
    ▼
mapToolResultToToolResultBlockParam()
    └── "Message delivered to user. (N attachment(s) included)"
    │
    ▼
renderToolResultMessage()
    ├── Transcript mode: ⏺ + markdown + attachments
    ├── Brief-only mode: "Claude" label + timestamp + markdown + attachments
    └── Default mode: plain markdown + attachments (no chrome)
```

## 集成点

- **REPL Bridge**：通过 `/api/oauth/file_upload` 上传到 web 查看器
- **Image Paste**：AskUserQuestionTool 流程中的粘贴图像通过相同的图像存储
- **Plan Mode**：在计划模式面试期间用于所有用户通信
- **Assistant Mode**：主要输出通道 — 系统提示强制其用于所有用户可见文本
- **SDK**：作为 SDK `tools` 选项中的工具提供，用于编程消息传递
- **Analytics**：`tengu_brief_send` 事件跟踪主动 vs. 正常消息和附件使用

## 相关模块

- [AskUserQuestionTool](./AskUserQuestionTool.md) — 可能先于 BriefTool 消息的交互式问题
- [ExitPlanModeTool](./ExitPlanModeTool.md) — 转换为 BriefTool 通信的计划模式退出
- [permissions/](../components/permissions/) — BriefTool 在计划模式中与之交互的权限系统
