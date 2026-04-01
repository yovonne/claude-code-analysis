# Vendor 模块

## 目的

vendor 目录将包含提供核心功能高性能实现的原生 Rust/NAPI 模块。在此源代码映射重建中，vendor 目录不存在作为源代码目录——相反，其功能已被 `native-ts/` 中的纯 TypeScript 移植取代。

## 位置

`restored-src/src/vendor/` — **在重建源代码中不存在**

## 原始原生模块（已被 native-ts 移植取代）

以下原生模块存在于原始代码库中，已移植到 TypeScript：

### color-diff-src (Rust)

**原始:** Rust 模块，使用 `syntect`（语法高亮）、`bat`（终端输出）和 `similar` crate（单词 diffing）

**取代自:** `native-ts/color-diff/`
- 使用 `highlight.js` 而非 syntect
- 使用 `diff` npm 包而非 similar crate
- API 与原始完全匹配，因此调用者不变

**已知差异:**
- highlight.js 有语法差距：纯标识符和操作符（`=`、`:`）没有作用域，默认为前景色而非白色/粉色渲染
- `BAT_THEME` 环境支持是存根（highlight.js 没有 bat 主题选择）
- 输出结构（行号、标记、背景、单词-diff）相同

### file-index-src (Rust)

**原始:** Rust NAPI 模块，包装 `nucleo`（https://github.com/helix-editor/nucleo）用于高性能模糊文件搜索

**取代自:** `native-ts/file-index/`
- 使用位图过滤和 top-k 选择重新实现 nucleo 风格评分
- 相同 API: `new FileIndex()`、`.loadFromFileList()`、`.search()`

### yoga-layout (C++)

**原始:** Meta 的 Yoga 布局引擎，编译为 WebAssembly/NAPI

**取代自:** `native-ts/yoga-layout/`
- 简化的单遍 flexbox 实现
- 覆盖 Ink 实际使用的功能子集
- CalculateLayout.cpp 中约 2500 行 C++ → 约 3000 行 TypeScript

## 为什么原生模块被取代

1. **可移植性**: 原生模块需要为每个平台（macOS、Windows、Linux）编译
2. **包大小**: 原生模块为 npm 包增加显著的二进制大小
3. **构建复杂性**: NAPI 模块需要 Rust/C++ 工具链
4. **CI 可靠性**: 原生模块在 Windows 上导致 CI 超时（highlight.js 延迟加载将测试推入 GC 暂停领域）

## 延迟加载模式

color-diff 移植使用延迟加载 highlight.js 以避免在启动时支付约 50MB 堆成本：

```typescript
let cachedHljs: HLJSApi | null = null
function hljs(): HLJSApi {
  if (cachedHljs) return cachedHljs
  const mod = require('highlight.js')
  cachedHljs = 'default' in mod && mod.default ? mod.default : mod
  return cachedHljs!
}
```

这反映了 NAPI 包装器的延迟 `dlopen` 模式——完整的 highlight.js 包在 require 时注册 190+ 语言语法（约 100-200ms 在 macOS 上，Windows 上数倍于此）。

## 相关模块

- [Native-TS 模块](./native-ts-modules.md) — 取代这些原生模块的 TypeScript 移植
- [UI 组件](../06-ui/ink-framework.md) — Yoga 布局为 Ink 的布局系统提供支持
