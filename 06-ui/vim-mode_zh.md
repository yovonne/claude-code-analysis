# Vim 模式

## 目的

Vim 模式在 Claude Code 的文本输入字段中提供 vim 风格的键盘导航和编辑。它实现了普通/插入模式的状态机、操作符-动作语法（delete/change/yank + 动作）、文本对象、查找动作、点重复，以及基于寄存器的 yank/粘贴——所有这些都作为纯函数，没有副作用。

## 位置

`restored-src/src/vim/`

## 关键导出

### 类型 (`types.ts`)

| 类型 | 描述 |
|------|-------------|
| `Operator` | `'delete' | 'change' | 'yank'` |
| `FindType` | `'f' | 'F' | 't' | 'T'` |
| `TextObjScope` | `'inner' | 'around'` |
| `VimState` | INSERT 模式（跟踪 `insertedText`）和 NORMAL 模式（跟踪 `CommandState`）的联合 |
| `CommandState` | 状态机: `idle`、`count`、`operator`、`operatorCount`、`operatorFind`、`operatorTextObj`、`find`、`g`、`operatorG`、`replace`、`indent` |
| `PersistentState` | 跨命令内存: `lastChange`、`lastFind`、`register`、`registerIsLinewise` |
| `RecordedChange` | 所有变更类型的点重复数据（插入、操作符、文本对象、查找、替换、x、切换大小写、缩进、打开行、连接） |

### 常量 (`types.ts`)

| 常量 | 描述 |
|----------|-------------|
| `OPERATORS` | `{d: 'delete', c: 'change', y: 'yank'}` |
| `SIMPLE_MOTIONS` | 基本动作键集合（h、l、j、k、w、b、e、W、B、E、0、^、$） |
| `FIND_KEYS` | 集合 `{f, F, t, T}` |
| `TEXT_OBJ_SCOPES` | `{i: 'inner', a: 'around'}` |
| `TEXT_OBJ_TYPES` | 文本对象键集合（w、W、引号、括号、方括号、花括号、尖括号） |
| `MAX_VIM_COUNT` | `10000` — 最大计数值 |

### 工厂函数

| 函数 | 描述 |
|----------|-------------|
| `createInitialVimState()` | 返回 `{mode: 'INSERT', insertedText: ''}` |
| `createInitialPersistentState()` | 返回空的持久状态 |

### 动作 (`motions.ts`)

| 函数 | 描述 |
|----------|-------------|
| `resolveMotion(key, cursor, count)` | 将动作键解析为目标光标位置（应用 count 次） |
| `isInclusiveMotion(key)` | 对于包含目标字符的动作返回 true（e、E、$） |
| `isLinewiseMotion(key)` | 对于整行动作返回 true（G） |

支持的动作：
- **字符**: `h`（左）、`l`（右）
- **行**: `j`/`gj`（下逻辑/视觉）、`k`/`gk`（上逻辑/视觉）
- **单词**: `w`/`b`/`e`（vim 单词）、`W`/`B`/`E`（WORD — 空白分隔）
- **行位置**: `0`（开始）、`^`（第一个非空白）、`$`（结束）
- **文件**: `G`（最后一行开始）

### 操作符 (`operators.ts`)

| 函数 | 描述 |
|----------|-------------|
| `executeOperatorMotion(op, motion, count, ctx)` | 使用简单动作执行操作符 |
| `executeOperatorFind(op, findType, char, count, ctx)` | 使用查找动作执行操作符 |
| `executeOperatorTextObj(op, objType, scope, count, ctx)` | 使用文本对象执行操作符 |
| `executeOperatorG(op, count, ctx)` | 执行操作符 + G（到第 N 行） |
| `executeOperatorGg(op, count, ctx)` | 执行操作符 + gg（从第 N 行） |
| `executeX(count, ctx)` | 删除光标下的字符 |
| `executeReplace(char, count, ctx)` | 替换光标下的字符 |
| `executeToggleCase(count, ctx)` | 切换字符大小写 |
| `executeIndent(dir, count, ctx)` | 缩进/取消缩进行 |
| `executeOpenLine(direction, ctx)` | 在上方/下方打开行并进入插入模式 |
| `executeJoin(count, ctx)` | 连接行 |
| `executePaste(after, count, ctx)` | 从寄存器粘贴（光标前/后） |

**OperatorContext** 提供: `cursor`、`text`、`setText`、`setOffset`、`enterInsert`、`getRegister`、`setRegister`、`getLastFind`、`setLastFind`、`recordChange`

### 文本对象 (`textObjects.ts`)

| 函数 | 描述 |
|----------|-------------|
| `findTextObject(text, offset, objectType, isInner)` | 查找文本对象边界 |

支持的文本对象：
- **单词**: `w`（vim 单词）、`W`（WORD — 空白分隔）
- **引号**: `"`、`'`、`` ` ``（inner/around）
- **括号**: `(`、`)`、`b`（圆括号）、`[`、`]`（方括号）、`{`、`}`、`B`（花括号）、`<`、`>`（尖括号）

### 状态转换 (`transitions.ts`)

| 函数 | 描述 |
|----------|-------------|
| `transition(state, input, ctx)` | 主转换函数 — 根据当前状态类型分发 |

转换处理程序（内部）：
- **`fromIdle(input, ctx)`**: 处理初始按键 — 操作符、计数、查找、g、替换、缩进、动作、x、粘贴、切换大小写、打开行、连接
- **`fromCount(state, input, ctx)`**: 累积数字计数或转换到操作符/查找/g
- **`fromOperator(state, input, ctx)`**: 处理操作符后的动作 — 简单动作、查找、文本对象、G、gg
- **`fromOperatorCount(state, input, ctx)`**: 在操作符后累积计数
- **`fromOperatorFind(state, input, ctx)`**: 处理操作符+查找后的字符
- **`fromOperatorTextObj(state, input, ctx)`**: 处理操作符+范围后的文本对象类型
- **`fromFind(state, input, ctx)`**: 处理查找动作后的字符（f/F/t/T）
- **`fromG(state, input, ctx)`**: 处理 `g` 后的第二个键（gg、ge 等）
- **`fromOperatorG(state, input, ctx)`**: 处理操作符+g 后的第二个键
- **`fromReplace(state, input, ctx)`**: 处理要替换的字符
- **`fromIndent(state, input, ctx)`**: 处理第二个缩进键（>>、<<）

**TransitionResult**: `{next?: CommandState, execute?: () => void}` — 下一个状态和可选的副作用函数

## 依赖

### 内部依赖

- `../utils/Cursor.js` — 带字素感知导航的 Cursor 类
- `../utils/intl.js` — 字素分段器（`getGraphemeSegmenter`、`firstGrapheme`、`lastGrapheme`）
- `../utils/stringUtils.js` — 字符串工具（`countCharInString`）

### 外部依赖

- 无 — 所有纯函数

## 状态机

```
VimState
├── INSERT (跟踪用于点重复的 insertedText)
└── NORMAL
    └── CommandState
        ├── idle ──┬─[d/c/y]──► operator
        │          ├─[1-9]────► count
        │          ├─[fFtT]───► find
        │          ├─[g]──────► g
        │          ├─[r]──────► replace
        │          └─[><]─────► indent
        │
        ├── operator ─┬─[motion]──► execute (返回 idle)
        │            ├─[0-9]────► operatorCount
        │            ├─[ia]─────► operatorTextObj
        │            └─[fFtT]───► operatorFind
        │
        ├── count ────┬─[d/c/y]──► operator (带计数)
        │            ├─[fFtT]───► find (带计数)
        │            ├─[g]──────► g (带计数)
        │            └─[0-9]────► 累积数字
        │
        └── ... (其他状态类似)
```

## 操作符-动作语法

vim 命令结构遵循模式: `[count] operator [count] motion`

示例：
- `d3w` — 删除 3 个单词（operator=d，count=3，motion=w）
- `ci"` — 修改引号内（operator=c，text object=i"）
- `y$` — 拉到行尾（operator=y，motion=$）
- `3dd` — 删除 3 行（count=3，operator=d，motion=d [整行]）
- `2f,` — 查找第二个逗号（count=2，find=f，char=,）

## 点重复

变更记录在 `PersistentState.lastChange` 中用于点重复（`.`）：

- **插入变更**: `{type: 'insert', text}`
- **操作符变更**: `{type: 'operator', op, motion, count}`
- **文本对象变更**: `{type: 'operatorTextObj', op, objType, scope, count}`
- **查找变更**: `{type: 'operatorFind', op, find, char, count}`
- **其他变更**: 替换、x、切换大小写、缩进、打开行、连接

## 寄存器系统

- **单一寄存器**: 存储 yanked/deleted 内容
- **整行标志**: 跟踪内容是否基于行（用于粘贴定位）
- **获取/设置**: 通过 OperatorContext 中的 `getRegister()` 和 `setRegister(content, linewise)`

## 关键设计决策

1. **纯函数**: 所有动作解析和操作符执行都是纯的 — 无副作用，只有返回值和上下文变化
2. **通过区分联合的状态机**: TypeScript 确保穷尽处理所有状态
3. **字素感知**: 使用 Intl.Segmenter 进行单词/文本对象检测中的字素集群边界
4. **转换表模式**: 每个状态有自己的转换函数 — 可扫描的真相来源
5. **OperatorContext 抽象**: 所有副作用（文本修改、寄存器访问、模式变化）通过上下文接口
6. **计数累积**: 数字作为字符串累积，在执行时解析为数字（支持任意计数直到 MAX_VIM_COUNT）
7. **查找持久化**: 最后查找（f/F/t/T）存储以使用 `;` 和 `,` 重复
8. **无外部依赖**: 整个 vim 模式是自包含的纯 TypeScript
