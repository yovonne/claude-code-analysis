# AskUserQuestionTool

## 目的

AskUserQuestionTool 是 Claude 在任务执行期间向用户呈现多选题的主要机制。它支持交互式对话框，模型可以在其中收集用户偏好、澄清模糊指令、关于实现选择做出决策，或提供方向性选择。该工具支持每次调用 1-4 个问题，每个问题有 2-4 个选项、带可选预览内容以进行视觉比较的多选模式，以及自由文本"其他"输入。它是 TUI 中交互式用户提示体验的核心。

## 位置

- `restored-src/src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx`
- `restored-src/src/tools/AskUserQuestionTool/prompt.ts`
- `restored-src/src/components/permissions/AskUserQuestionPermissionRequest/`（UI 组件）

## 关键导出

### 函数

- `AskUserQuestionTool`：通过 `buildTool()` 构建的主要工具定义 — 处理问题验证、权限检查和响应收集
- `validateHtmlPreview()`：验证 HTML 预览片段的安全性（无 `<script>`、`<style>`、完整文档）

### 类型

- `Question`：带 `question`、`header`、`options` 和 `multiSelect` 字段的单个问题
- `QuestionOption`：问题中的选项，带 `label`、`description` 和可选的 `preview`
- `Output`：工具输出 schema，包含 `questions`、`answers`（问题文本 → 答案字符串）和可选的 `annotations`

### 常量

- `ASK_USER_QUESTION_TOOL_NAME`：`'AskUserQuestion'`
- `ASK_USER_QUESTION_TOOL_CHIP_WIDTH`：`12` — 标题 chip/tag 的最大字符数
- `DESCRIPTION`：总结其用途的工具描述
- `ASK_USER_QUESTION_TOOL_PROMPT`：LLM 何时以及如何使用工具的指令
- `PREVIEW_FEATURE_PROMPT`：`markdown` 和 `html` 预览模式的格式特定指导

## 依赖

### 内部依赖

- `prompt.ts` — 工具提示、描述和预览功能指导
- `PermissionRequest.tsx` — 基础权限请求组件
- `AskUserQuestionPermissionRequest/` — 完整交互式对话框 UI（7 个组件）
- `use-multiple-choice-state.ts` — 多问题导航的状态管理 hook
- `CustomSelect/index.js` — `Select` 和 `SelectMulti` 组件用于选项选择
- `PreviewBox.tsx` — 用于渲染预览内容的带边框等宽框
- `QuestionView.tsx` — 标准问题视图（选项列表 + 文本输入）
- `PreviewQuestionView.tsx` — 并排视图（左侧选项，右侧预览）
- `QuestionNavigationBar.tsx` — 用于在问题之间导航的标签栏
- `SubmitQuestionsView.tsx` — 最终答案确认的审核提交屏幕

### 外部依赖

- `zod/v4` — 带延迟 schema 的 Schema 验证
- `react` — 使用 React Compiler 的 UI 组件渲染
- `bun:bundle` — 功能标志门控（`KAIROS`、`KAIROS_CHANNELS`）

## 实现细节

### 输入 Schema

工具使用 Zod 严格对象验证，结构如下：

```typescript
{
  questions: [
    {
      question: string,       // 完整问题（必须以 ? 结尾）
      header: string,         // chip/tag 的短标签（最多 12 个字符）
      options: [              // 每个问题 2-4 个选项
        {
          label: string,       // 显示文本（1-5 个词）
          description: string, // 权衡/含义的解释
          preview?: string     // 可选：markdown 或 HTML 预览内容
        }
      ],
      multiSelect: boolean     // 允许多选（默认：false）
    }
  ],                       // 每次调用 1-4 个问题
  answers?: Record<string, string>,       // 由权限组件填充
  annotations?: Record<string, {          // 每问题注释
    preview?: string,       // 所选选项的预览内容
    notes?: string          // 用户添加的自由文本备注
  }>,
  metadata?: {
    source?: string         // 例如 "remember" 用于 /remember 命令
  }
}
```

**验证约束：**
- 问题必须唯一（无重复问题文本）
- 每个问题中的选项标签必须唯一
- 每个问题必须有 2-4 个选项
- 不允许"其他"选项（UI 自动提供）
- HTML 预览模式：仅限片段 — 无 `<html>`、`<body>`、`<script>`、`<style>` 或 `<!DOCTYPE>`

### 输出 Schema

```typescript
{
  questions: Question[],                    // 被问的问题
  answers: Record<string, string>,          // question text → answer string
  annotations?: Record<string, {            // 可选的每问题注释
    preview?: string,
    notes?: string
  }>
}
```

多选答案是逗号分隔的字符串（例如 `"Option A, Option B"`）。

### 权限行为

该工具始终使用 `'ask'` 行为 — 它需要显式的用户交互：

```typescript
async checkPermissions(input) {
  return {
    behavior: 'ask',
    message: 'Answer questions?',
    updatedInput: input
  };
}
```

该工具标记为 `requiresUserInteraction() => true` 和 `shouldDefer: true`，意味着它阻塞执行直到用户响应。它也是 `isReadOnly() => true` 和 `isConcurrencySafe() => true`。

### 通道可用性

当在通道模式（Telegram/Discord）下运行时，该工具被禁用，因为没有用户在键盘前：

```typescript
isEnabled() {
  if ((feature('KAIROS') || feature('KAIROS_CHANNELS')) && getAllowedChannels().length > 0) {
    return false;
  }
  return true;
}
```

### 预览功能

选项可以包含 `preview` 字段，渲染具体制品以进行视觉比较。支持两种格式：

| 格式 | 渲染 | 用例 |
|--------|-----------|-----------|
| `markdown` | 带 markdown 渲染的等宽框 | ASCII 模拟、代码片段、配置示例 |
| `html` | 带内联样式的 HTML 片段 | UI 模拟、格式比较、图表 |

当任何选项有预览时，UI 切换到并排布局：
- **左侧面板**：垂直选项列表（30 个字符宽）
- **右侧面板**：预览内容 + 备注字段

预览仅支持单选问题（不是 `multiSelect`）。

### 响应处理

当用户提交答案时，工具：

1. 将答案收集到 `Record<string, string>` 中，以问题文本为键
2. 从所选选项预览和用户备注构建注释
3. 将更新后的输入通过 `toolUseConfirm.onAllow()` 传递
4. `call()` 方法返回问题、答案和注释
5. `mapToolResultToToolResultBlockParam()` 格式化结果供 LLM 使用：

```
User has answered your questions: "Question 1"="Answer 1" ..., You can now continue with the user's answers in mind.
```

## UI 组件

### AskUserQuestionPermissionRequest

协调交互式对话框的顶级组件。它：

1. 通过 `inputSchema.safeParse()` 解析和验证工具输入
2. 根据终端大小计算最佳内容尺寸
3. 通过 `useMultipleChoiceState()` hook 管理多问题状态
4. 处理每问题的图像粘贴附件
5. 根据当前问题是否有预览选项在 `QuestionView`（标准）和 `PreviewQuestionView`（并排）之间路由
6. 当所有问题都已回答时显示 `SubmitQuestionsView`（或导航到最后一题之后）

**关键处理程序：**
- `handleQuestionAnswer` — 记录答案，对单题单选自动前进
- `handleFinalResponse` — 从审核屏幕提交或取消
- `handleRespondToClaude` — 发送答案并返回"澄清这些问题"反馈消息
- `handleFinishPlanInterview` — 在计划模式中，信号面试完成
- `handleCancel` — 完全拒绝问题

### QuestionView

不带预览的标准问题视图。显示：
- 问题导航栏（带已回答复选框的标签）
- 问题标题
- 带选项 + "其他" 文本输入的 `Select` 或 `SelectMulti` 组件
- 页脚操作："Chat about this"和（在计划模式中）"Skip interview and plan immediately"
- 带键盘快捷键的帮助文本

支持：
- 键盘导航（上/下，ctrl+p/ctrl+n）
- 带外部编辑器的文本输入模式（ctrl+g）
- 图像粘贴附件
- 单选自动前进

### PreviewQuestionView

用于带预览内容的问题的并排布局：
- **左面板**（30 个字符）：带焦点指针和选择复选框的编号选项
- **右面板**：`PreviewBox` 渲染焦点选项的预览 + 备注字段

导航：
- 上/下箭头导航选项
- Enter 选择焦点选项
- `n` 键聚焦备注输入
- Escape 退出备注输入或取消
- 数字键（1-9）跳转到特定选项

### PreviewBox

用于渲染预览内容的带边框等宽框组件：
- 带语法高亮的 markdown 渲染
- 截断超过 `maxLines` 的内容并显示截断指示符
- 根据可用终端空间计算框尺寸
- 使用 box-drawing 字符（`┌`、`┐`、`└`、`┘`、`─`、`│`）作为边框

### QuestionNavigationBar

显示在每个问题上方的标签栏：
- 问题标题作为带复选框指示器的可点击标签（✓ 表示已回答）
- 导航箭头（← →）用于多问题集
- "Submit" 标签（单问题单选隐藏）
- 动态宽度分配 — 当前问题获得优先宽度，其他共享剩余空间
- 终端太窄时截断

### SubmitQuestionsView

在回答所有问题后显示的审核提交屏幕：
- 列出所有已回答的问题及其选定的答案
- 如果并非所有问题都已回答则发出警告
- 显示权限规则说明
- 提供"Submit answers"或"Cancel"选项

### useMultipleChoiceState Hook

用于多问题对话框的 `useReducer` 状态管理器：

```typescript
type State = {
  currentQuestionIndex: number;
  answers: Record<string, AnswerValue>;
  questionStates: Record<string, QuestionState>;
  isInTextInput: boolean;
}
```

**操作：**
- `next-question` — 前进到下一题
- `prev-question` — 返回上一题（夹紧到 0）
- `update-question-state` — 更新问题的内部状态（选定的值、文本输入）
- `set-answer` — 记录答案并可选自动前进
- `set-text-input-mode` — 切换文本输入焦点状态

**公开 API：**
- `currentQuestionIndex`、`answers`、`questionStates`、`isInTextInput`
- `nextQuestion()`、`prevQuestion()`
- `updateQuestionState(questionText, updates, isMultiSelect)`
- `setAnswer(questionText, answer, shouldAdvance?)`
- `setTextInputMode(isInInput)`

## 计划模式集成

在计划模式中，该工具在面试阶段用于澄清需求：

- 问题应该澄清需求或在最终确定计划之前选择方法**在最终确定计划之前**
- 该工具不能用于计划批准 — `ExitPlanModeTool` 处理那个
- 问题不能引用"计划"，因为用户直到调用 `ExitPlanModeTool` 才能看到它
- 页脚提供"Chat about this"（将澄清发送回 Claude）和"Skip interview and plan immediately"选项
- 当 `metadata.source === "remember"` 时，分析事件跟踪 `/remember` 命令流程

## 键盘快捷键

| 键 | 动作 |
|-----|--------|
| `↑` / `Ctrl+P` | 向上导航（选项或页脚） |
| `↓` / `Ctrl+N` | 向下导航（选项或页脚） |
| `Enter` | 选择焦点选项 / 执行页脚操作 |
| `Escape` | 取消 / 退出备注输入 |
| `Tab` | 切换问题（多问题） |
| `1`–`9` | 跳转到编号选项（预览模式） |
| `n` | 聚焦备注输入（预览模式） |
| `Ctrl+G` | 在外部编辑器中打开备注 |

## 错误处理

### 输入验证错误

| 错误代码 | 消息 |
|------------|---------|
| 1 | HTML 预览验证失败（无效片段、不允许的标签） |

### Schema 验证

- 重复的问题文本 → 改进错误
- 问题中重复的选项标签 → 改进错误
- 问题超出 1-4 范围 → Zod 错误
- 选项超出 2-4 范围 → Zod 错误

## 数据流

```
Model calls AskUserQuestion tool with questions
    │
    ▼
checkPermissions() → behavior: 'ask'
    │
    ▼
AskUserQuestionPermissionRequest renders
    │
    ├── Parses input via inputSchema.safeParse()
    ├── useMultipleChoiceState() initializes state
    │
    ├── For each question:
    │   ├── has preview? → PreviewQuestionView (side-by-side)
    │   └── no preview?  → QuestionView (standard)
    │
    ├── User navigates, selects options, adds notes
    │
    ├── All answered → SubmitQuestionsView (review)
    │
    └── User action:
        ├── Submit → onAllow(updatedInput with answers + annotations)
        ├── Cancel → onReject()
        ├── Chat about this → onReject(feedback, imageBlocks)
        └── Finish interview → onReject(feedback, imageBlocks)
    │
    ▼
call() returns { questions, answers, annotations }
    │
    ▼
mapToolResultToToolResultBlockParam() formats for LLM
```

## 集成点

- **Permission System**：使用 `requiresUserInteraction()` 阻塞执行直到用户响应
- **Plan Mode**：与计划面试阶段集成（`isPlanModeInterviewPhaseEnabled`）
- **Image Paste**：支持每问题粘贴图像，转换为 `ImageBlockParam`
- **External Editor**：备注可以通过 `editPromptInEditor()` 在用户配置的 IDE 中编辑
- **Analytics**：为接受、拒绝、响应和完成计划操作记录事件
- **Channel Permissions**：通过 `isEnabled()` 在通道模式（Telegram/Discord）中禁用
- **SDK**：公开 schemas 导出为 `_sdkInputSchema` 和 `_sdkOutputSchema` 供 SDK 消费者使用

## 相关模块

- [ExitPlanModeTool](./ExitPlanModeTool.md) — 计划批准（不要与提问混淆）
- [PermissionRequest](../components/permissions/PermissionRequest.md) — 基础权限请求组件
- [PermissionDialog](../components/permissions/PermissionDialog.md) — 权限对话框编排
- [SkillTool](./SkillTool.md) — 在执行期间可能触发问题的技能
