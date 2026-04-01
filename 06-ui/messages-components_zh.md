# 消息组件

## 目的

消息组件系统渲染 Claude Code 聊天界面中的各种消息类型。每种消息类型都有专门的组件处理其特定的内容格式、样式和交互——从用户提示和助手响应到工具使用、系统通知和团队协作消息。

## 位置

`restored-src/src/components/messages/`

## 关键组件

### 用户消息类型

| 组件 | 描述 |
|-----------|-------------|
| `UserPromptMessage` | 用户文本输入/提示消息 |
| `UserTextMessage` | 纯文本用户消息 |
| `UserCommandMessage` | 斜杠命令消息 |
| `UserChannelMessage` | 来自用户的渠道/系统消息 |
| `UserBashInputMessage` | 用户 bash/shell 输入 |
| `UserBashOutputMessage` | Bash 命令输出 |
| `UserImageMessage` | 用户提交的图像消息 |
| `UserMemoryInputMessage` | 用户内存输入 |
| `UserPlanMessage` | 用户计划相关消息 |
| `UserResourceUpdateMessage` | 资源更新通知 |
| `UserToolResultMessage/` | 工具结果显示（子目录） |
| `UserTeammateMessage` | 来自队友代理的消息 |
| `UserLocalCommandOutputMessage` | 本地命令输出显示 |

### 助手消息类型

| 组件 | 描述 |
|-----------|-------------|
| `AssistantTextMessage` | 助手文本响应 |
| `AssistantThinkingMessage` | 扩展思考显示 |
| `AssistantRedactedThinkingMessage` | 已编辑的思考内容 |
| `AssistantToolUseMessage` | 带状态的工具使用显示 |

### 系统和特殊消息

| 组件 | 描述 |
|-----------|-------------|
| `SystemTextMessage` | 系统通知文本 |
| `SystemAPIErrorMessage` | API 错误显示 |
| `ShutdownMessage` | 会话关闭通知 |
| `RateLimitMessage` | 速率限制警告 |
| `PlanApprovalMessage` | 计划批准请求 |
| `TaskAssignmentMessage` | 任务分配通知 |
| `HookProgressMessage` | Hook 执行进度 |
| `AdvisorMessage` | 顾问建议 |

### 内容组件

| 组件 | 描述 |
|-----------|-------------|
| `GroupedToolUseContent` | 将多个工具使用条目分组 |
| `HighlightedThinkingText` | 语法高亮的思考文本 |
| `CollapsedReadSearchContent` | 折叠的搜索结果内容 |
| `CompactBoundaryMessage` | 紧凑模式边界标记 |
| `AttachmentMessage` | 文件附件显示 |

### 团队协作

| 组件 | 描述 |
|-----------|-------------|
| `teamMemCollapsed` | 折叠的队友消息 |
| `teamMemSaved` | 已保存的队友状态指示器 |

### 工具函数

| 导出 | 描述 |
|--------|-------------|
| `nullRenderingAttachments` | 附件渲染工具 |

## 依赖

### 内部依赖

- `../ink.js` — Ink 框架（Box、Text 等）
- `../design-system/` — 主题感知原语
- `../Spinner/` — 工具消息的加载指示器
- `../HighlightedCode/` — 语法高亮代码块
- `../StructuredDiff/` — 文件编辑的 Diff 显示
- `../messageActions.tsx` — 消息操作按钮
- `../Markdown.tsx` / `../MarkdownTable.tsx` — Markdown 渲染
- `../FilePathLink.tsx` — 文件路径链接
- `../FallbackToolUseErrorMessage.tsx` — 工具错误回退
- `../FallbackToolUseRejectedMessage.tsx` — 工具拒绝回退
- `../FileEditToolDiff.tsx` — 文件编辑 diff 显示
- `../FileEditToolUpdatedMessage.tsx` — 文件编辑更新通知
- `../FileEditToolUseRejectedMessage.tsx` — 文件编辑拒绝
- `../NotebookEditToolUseRejectedMessage.tsx` — Notebook 编辑拒绝
- `../SandboxViolationExpandedView.tsx` — 沙箱违规详情
- `../ToolUseLoader.tsx` — 工具加载指示器
- `../ThinkingToggle.tsx` — 思考切换
- `../../utils/` — 格式化、主题和工具函数
- `../../hooks/` — 共享 React hooks

### 外部依赖

- `react` — React 19
- `figures` — Unicode 符号

## 消息架构

### 消息流

```
消息 (容器)
    ↓
MessageResponse (响应包装器)
    ↓
MessageRow (行布局)
    ↓
[按类型的特定消息组件]
    ↓
内容渲染器 (Markdown、Code、Diff 等)
```

### 消息类型路由

消息根据以下内容路由到适当的组件：
- **消息角色**: user、assistant、system
- **消息子类型**: text、command、tool-use、image 等
- **内容格式**: 纯文本、markdown、结构化数据

### 工具消息处理

工具消息有专门的处理：
- **加载状态**: 执行期间显示 `ToolUseLoader` 旋转器
- **成功**: 显示格式化输出（diff、文本、结构化数据）
- **错误**: 显示带错误详情的 `FallbackToolUseErrorMessage`
- **拒绝**: 显示带拒绝原因的 `FallbackToolUseRejectedMessage`
- **文件编辑**: 使用 `StructuredDiff` 进行可视化 diff 显示
- **分组工具**: `GroupedToolUseContent` 折叠多个工具调用

### 思考显示

扩展思考消息：
- **活跃思考**: 带闪烁动画的 `AssistantThinkingMessage`
- **完成的思考**: 显示持续时间和折叠内容
- **已编辑的思考**: 用于隐藏内容的 `AssistantRedactedThinkingMessage`
- **切换**: `ThinkingToggle` 允许展开/折叠

### 团队消息

队友代理消息：
- **折叠**: `teamMemCollapsed` 显示紧凑视图
- **展开**: 带代理身份和状态的完整消息
- **保存状态**: `teamMemSaved` 表示持久化的工作

## 关键设计决策

1. **类型特定组件**: 每种消息类型都有自己的组件，以清晰分离关注点
2. **渐进式渲染**: 工具消息显示加载状态，然后转换到结果
3. **Markdown 支持**: 通过 Markdown 组件进行富文本渲染，带代码高亮
4. **Diff 集成**: 文件编辑使用结构化 diff 显示以提高视觉清晰度
5. **紧凑模式**: 边界消息和折叠视图支持紧凑显示
6. **团队感知**: 队友与领导者代理消息的独特渲染
7. **错误弹性**: 用于格式错误或失败工具消息的回退组件
8. **附件支持**: 图像和文件附件以内联预览方式渲染
