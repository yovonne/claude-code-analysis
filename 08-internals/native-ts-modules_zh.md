# Native-TS 模块

## 目的

原生 Rust/NAPI 模块的纯 TypeScript 移植。这些模块重新实现了与原生模块相同的 API 表面，允许应用程序在无原生依赖的情况下运行，同时保持 API 兼容性。

## 位置

`restored-src/src/native-ts/`

## 模块

### `color-diff/` — 语法高亮 Diff 渲染

**目的:** 为终端显示渲染带语法高亮的彩色 diff 块和源文件。

**原始:** `vendor/color-diff-src`（Rust，使用 syntect + bat + similar crate）
**移植:** 使用 `highlight.js`（通过 cli-highlight 已有的依赖）和 `diff` npm 包

**关键导出:**
- `ColorDiff`: 渲染带语法高亮、单词级 diff、行号和 ANSI 颜色代码的统一 diff 块
- `ColorFile`: 渲染带语法高亮和行号的完整源文件
- `getSyntaxTheme(themeName)`: 返回给定 Claude 主题的语法主题名称
- `getNativeModule()`: 返回与原生调用者兼容的模块 API

**颜色系统:**
- 支持 `truecolor`、`color256` 和 `ansi` 颜色模式
- `ansi256FromRgb()`: `ansi_colours::ansi256_from_rgb` 的移植——使用 6x6x6 立方体 + 24 灰度近似 RGB 到 xterm-256 调色板
- 三个预测量主题调色板：Monokai Extended（暗）、GitHub（亮）、ANSI

**语法高亮:**
- 延迟加载 `highlight.js` 以避免启动时约 50MB 堆成本
- 通过文件名、扩展名和 shebang 检测语言
- 处理 hljs 10.x（`kind`）和 11.x（`scope`）节点格式
- 存储关键字重新分割：`const`、`function`、`class` 等关键字获得青色存储颜色而非粉色关键字颜色

**单词 diff 算法:**
- 标记化为单词运行、空格运行和标点符号
- 使用 `diff` 包的 `diffArrays`
- CHANGE_THRESHOLD (0.4): 当 >40% 文本更改时跳过单词 diff（太嘈杂）
- 找到相邻的 +/- 行对并计算字符级更改范围

**转换管道:**
1. 解析标记（+、-、）和行号
2. 为相邻更改行计算单词-diff 范围
3. 用语法颜色高亮每行
4. 应用背景颜色（行 + 单词级）
5. 包装到终端宽度
6. 添加行号和标记
7. 转换为 ANSI 转义序列

### `file-index/` — 模糊文件搜索

**目的:** 高性能模糊文件搜索，移植自 nucleo 库。

**原始:** `vendor/file-index-src`（Rust，包装 nucleo）
**移植:** 带基于位图的过滤和 top-k 选择的纯 TypeScript

**关键导出:**
- `FileIndex` 类:
  - `loadFromFileList(fileList)`: 从路径数组同步构建索引
  - `loadFromFileListAsync(fileList)`: 带渐进式查询的异步构建
  - `search(query, limit)`: 模糊搜索返回前 N 个结果

**搜索算法:**
1. **位图拒绝**: 每个路径有 26 位位图表示存在的字母。如果路径不包含所有 needle 字母则 O(1) 拒绝（"test" 约 89% 存活率，稀有字符 >90% 拒绝）
2. **融合 indexOf 扫描**: 使用 `String.indexOf`（V8/JSC 中的 SIMD 加速）找到位置，并内联累积间隙/连续项
3. **边界/camelCase 评分**: 在单词边界（`/`、`-`、`_`、`.`）和 camelCase 转换处匹配加分
4. **Top-k 选择**: 维护最佳 `limit` 匹配的排序数组，避免 O(n log n) 完全排序
5. **分数归一化**: 最终分数 = 结果中的位置 / 结果数量（越低越好）
6. **测试惩罚**: 包含 "test" 的路径获得 1.05x 惩罚（上限为 1.0）

**评分常量（nucleo 风格）:**
- `SCORE_MATCH = 16`、`BONUS_BOUNDARY = 8`、`BONUS_CAMEL = 6`、`BONUS_CONSECUTIVE = 4`、`BONUS_FIRST_CHAR = 8`
- `PENALTY_GAP_START = 3`、`PENALTY_GAP_EXTENSION = 1`

**异步构建:**
- 每约 4ms 同步工作向事件循环让步
- `queryable` promise 在第一个块后解析（270k 路径约 5-10ms）
- `done` promise 在整个索引构建完成时解析
- `readyCount` 跟踪已索引的路径数；`search()` 在构建继续时搜索就绪前缀

**顶层缓存:**
- 提取唯一顶层路径段（前 100 个）
- 按长度然后字母顺序排序
- 为空查询返回

### `yoga-layout/` — Flexbox 布局引擎

**目的:** Meta 的 Yoga 布局引擎的纯 TypeScript 移植，用于终端 UI 布局。

**原始:** Yoga C++（仅 CalculateLayout.cpp 就约 2500 行）
**移植:** 简化单遍 flexbox 实现，覆盖 Ink 实际使用的子集

**关键导出:**
- `Node` 类: 带完整样式/布局 API 的 Yoga 节点
- `Config`: 布局配置
- 枚举: `Align`、`FlexDirection`、`Justify`、`Wrap`、`Overflow`、`PositionType`、`Display`、`Edge`、`Gutter`、`MeasureMode`、`Unit`、`Direction`、`BoxSizing`、`Dimension`、`Errata`、`ExperimentalFeature`

**已实现的功能:**
- `flex-direction`（row/column + reverse）
- `flex-grow` / `flex-shrink` / `flex-basis`
- `align-items` / `align-self`（stretch、flex-start、center、flex-end）
- `justify-content`（所有六个值）
- `margin` / `padding` / `border` / `gap`
- `width` / `height` / `min` / `max`（点、百分比、自动）
- `position: relative / absolute`
- `display: flex / none / contents`
- 测量函数（用于文本节点）
- `flex-wrap: wrap / wrap-reverse`（多行）
- `align-content`（定位换行）
- `margin: auto`（主轴 + 交叉轴）
- 多遍 flex 约束夹紧
- 基线对齐

**未实现:**
- `aspect-ratio`
- `box-sizing: content-box`
- RTL 方向（Ink 始终传递 LTR）

**性能优化:**
1. **脏标志缓存**: 当输入匹配缓存值时跳过干净的子树
2. **多条目布局缓存**: 4 槽 LRU 缓存存储（输入 → 计算的 w,h）—— 500 条消息 scrollbox 的 dirty-leaf 重新布局从 76k 减少到 4k（6.86ms → 550µs）
3. **基础缓存**: 带生成跟踪的缓存 `computeFlexBasis` 结果 — 1593 节点树的 105k 访问 → ~10k
4. **快速路径标志**: `_hasAutoMargin`、`_hasPosition`、`_hasPadding`、`_hasBorder`、`_hasMargin` 跳过常见情况的昂贵边解析
5. **`resolveEdges4Into`**: 在一次传递中解析所有 4 个物理边，提升共享回退查找 — 根据 CPU 配置文件曾是 #1 热点
6. **基于生成的缓存有效性**: `_cGen` / `_fbGen` 用当前 `calculateLayout` 生成标记条目，允许干净节点的跨代命中

**布局算法（Yoga STEP 1-9）:**
1. 计算每个子项的 flex-basis，分成行
2. 解析灵活长度（增长/收缩）
3. 测量交叉尺寸
4. 确定容器尺寸
5. 定位子项（主轴，然后交叉轴）
6. 处理绝对定位
7. 应用基线对齐
8. 处理溢出
9. 四舍五入到像素网格

## 依赖

### 内部依赖

- `ink/stringWidth.js` — 用于文本包装的字符串宽度计算（color-diff）
- `utils/log.js` — 错误日志

### 外部依赖

- `diff` — 单词级 diff 的数组差异（color-diff）
- `highlight.js` — 语法高亮（color-diff，延迟加载）
- `path` — 用于语言检测的 `basename`、`extname`（color-diff）

## 相关模块

- [UI 组件](../06-ui/ink-framework.md) — Yoga 布局为 Ink 组件布局提供支持
- [Vendor 模块](./vendor-modules.md) — 这些移植所替换的原始原生实现
