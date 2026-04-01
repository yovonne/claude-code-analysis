# 功能命令

## 目的

记录管理 Claude Code 可扩展功能（agent、技能、记忆、后台任务和钩子）的 CLI 命令。这些命令提供用于配置和监控扩展 Claude Code 能力的组件的交互式 UI。

---

## /agents（Agent 管理）

### 目的
打开用于管理 agent 配置（自定义 agent 定义、权限和行为设置）的交互式菜单。

### 位置
`restored-src/src/commands/agents/index.ts`
`restored-src/src/commands/agents/agents.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `agents` |
| 类型 | `local-jsx` |
| 描述 | Manage agent configurations |

### 关键导出

#### 函数
- `call(onDone, context)`：入口点；使用可用工具渲染 `AgentsMenu`

### 实现细节

#### 核心逻辑
1. 检索当前 app 状态并提取 `toolPermissionContext`
2. 通过 `getTools(permissionContext)` 获取可用工具列表
3. 使用工具和退出回调渲染 `AgentsMenu` 组件
4. 菜单允许用户查看、配置和管理 agent 定义

### 数据流
```
User runs /agents
  → context.getAppState() → toolPermissionContext
  → getTools(permissionContext) → tool list
  → Render AgentsMenu(tools, onExit)
  → User interacts with agent management UI
  → User exits → onDone() closes
```

---

## /skills（技能列表）

### 目的
显示可在会话期间调用的可用技能（命令技能）菜单。

### 位置
`restored-src/src/commands/skills/index.ts`
`restored-src/src/commands/skills/skills.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `skills` |
| 类型 | `local-jsx` |
| 描述 | List available skills |

### 关键导出

#### 函数
- `call(onDone, context)`：入口点；使用可用命令渲染 `SkillsMenu`

### 实现细节

#### 核心逻辑
1. 渲染 `SkillsMenu` 组件，传递：
   - `onExit`：关闭菜单的回调
   - `commands`：来自 `context.options.commands` 的可用命令
2. 菜单显示注册给用户的所有可用技能/命令插件

### 数据流
```
User runs /skills
  → Render SkillsMenu(onExit, commands=context.options.commands)
  → User browses available skills
  → User exits → onDone() closes
```

---

## /memory（记忆文件编辑）

### 目的
打开用于选择和编辑 Claude 记忆文件（CLAUDE.md 文件）的对话框。记忆文件在会话之间提供持久上下文。

### 位置
`restored-src/src/commands/memory/index.ts`
`restored-src/src/commands/memory/memory.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `memory` |
| 类型 | `local-jsx` |
| 描述 | Edit Claude memory files |

### 关键导出

#### 函数
- `call(onDone)`：入口点；清除缓存，填充记忆文件，渲染 `MemoryCommand`

#### 组件
- `MemoryCommand`：呈现记忆文件选择器并处理编辑的 React 组件

### 实现细节

#### 核心逻辑
1. **准备**：清除记忆文件缓存并等待 `getMemoryFiles()` 填充数据后再渲染
2. **文件选择**：渲染列出可用记忆文件的 `MemoryFileSelector` 对话框
3. **文件处理**（选择时）：
   - 如果不存在，创建 `claude` 配置目录（使用 `recursive: true` 的幂等操作）
   - 如果文件不存在则创建（使用 `wx` 标志避免覆盖现有内容）
   - 在用户配置的外部编辑器（`$VISUAL` 或 `$EDITOR`）中打开文件
4. **编辑器检测**：首先检查 `$VISUAL`，然后是 `$EDITOR`，报告正在使用的编辑器
5. **完成**：显示相对路径和编辑器提示，或失败时的错误消息

#### 边缘情况
- 文件已存在：`wx` 标志抛出 `EEXIST`，被忽略以保留内容
- 目录不存在：使用 `mkdir({ recursive: true })` 自动创建
- 未配置编辑器：回退到系统默认编辑器
- 打开时出错：通过 `logError` 记录并报告给用户

### 数据流
```
User runs /memory
  → clearMemoryFileCaches() → await getMemoryFiles()
  → Render MemoryCommand dialog
  → User selects a memory file
  → Ensure directory exists (mkdir recursive)
  → Create file if needed (wx flag, ignore EEXIST)
  → editFileInEditor(memoryPath)
  → Detect editor ($VISUAL or $EDITOR)
  → Display: "Opened memory file at <relative-path>"
```

---

## /tasks（后台任务管理）

### 目的
打开用于查看和管理后台任务（在后运行 bash 进程）的对话框。

### 位置
`restored-src/src/commands/tasks/index.ts`
`restored-src/src/commands/tasks/tasks.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `tasks` |
| 别名 | `bashes` |
| 类型 | `local-jsx` |
| 描述 | List and manage background tasks |

### 关键导出

#### 函数
- `call(onDone, context)`：入口点；渲染 `BackgroundTasksDialog`

### 实现细节

#### 核心逻辑
1. 渲染 `BackgroundTasksDialog` 组件，带有：
   - `toolUseContext`：用于任务操作的完整上下文
   - `onDone`：关闭对话框的回调
2. 对话框显示所有运行中后台任务及其状态、输出和管理选项（查看输出、停止等）

### 数据流
```
User runs /tasks (or /bashes)
  → Render BackgroundTasksDialog(toolUseContext, onDone)
  → User views/manages background tasks
  → User exits → onDone() closes
```

---

## /hooks（钩子配置）

### 目的
打开用于配置钩子行为的菜单 — 在工具事件响应中运行的自定义脚本（pre/post hooks）。

### 位置
`restored-src/src/commands/hooks/index.ts`
`restored-src/src/commands/hooks/hooks.tsx`

### 命令定义
| 属性 | 值 |
|----------|-------|
| 名称 | `hooks` |
| 类型 | `local-jsx` |
| 描述 | View hook configurations for tool events |
| Immediate | `true` |

### 关键导出

#### 函数
- `call(onDone, context)`：入口点；记录分析事件，渲染 `HooksConfigMenu`

### 实现细节

#### 核心逻辑
1. 记录分析事件 `tengu_hooks_command`
2. 检索 app 状态并提取 `toolPermissionContext`
3. 通过 `getTools(permissionContext).map(tool => tool.name)` 获取所有工具名称列表
4. 使用工具名称和退出回调渲染 `HooksConfigMenu`
5. 菜单允许用户为每个工具事件配置 pre/post 钩子

### 数据流
```
User runs /hooks
  → logEvent('tengu_hooks_command')
  → context.getAppState() → toolPermissionContext
  → getTools(permissionContext).map(tool => tool.name) → tool name list
  → Render HooksConfigMenu(toolNames, onExit)
  → User configures hooks for tool events
  → User exits → onDone() closes
```

---

## 功能命令概述

这五个命令提供了管理 Claude Code 可扩展系统的主要界面：

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ /agents  │  │ /skills   │  │ /memory   │  │ /tasks    │  │ /hooks    │
│ (manage  │  │ (list     │  │ (edit     │  │ (manage   │  │ (config   │
│  agents) │  │  skills)  │  │  memory)  │  │  tasks)   │  │  hooks)   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
      │             │             │             │             │
      └─────────────┴─────────────┴─────────────┴─────────────┘
                              │
                    ┌─────────┴─────────┐
                    │  Plugin System   │
                    │  (plugins, MCP,  │
                    │   skills, agents)│
                    └───────────────────┘
```

### 关系
- **Agent** 和 **Skills** 都是通过插件系统管理的插件提供的扩展
- **Memory** 文件（CLAUDE.md）提供 agent 和技能可以引用的持久上下文
- **Tasks** 是 agent 可能生成的后台进程
- **Hooks** 拦截工具事件，可以为任何工具（包括 agent 工具）触发自定义行为
