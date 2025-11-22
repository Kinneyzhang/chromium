# Blink 渲染引擎分析摘要

> 本文档是 `blink_rendering_engine_analysis.md` 的快速参考摘要。

## 核心架构

### 模块层次（从上到下）
1. **controller/** - 系统基础设施
2. **extensions/** - 嵌入器特定 API
3. **modules/** - Web 平台特性
4. **core/** + **bindings/core** - 核心渲染 + V8 绑定
5. **platform/** - 底层平台抽象

### 渲染管道（四大阶段）
1. **DOM 构建** - 解析 HTML/XML → DOM 树
2. **样式计算** - 解析 CSS → 应用样式
3. **布局计算** - 计算几何信息
4. **绘制渲染** - 生成显示列表 → 合成

## 关键数据结构

### DOM 树
```
Node (基类)
  ├─ Element (元素节点)
  ├─ Text (文本节点)
  └─ Document (文档节点)
```

### 布局树
```
LayoutObject (基类)
  ├─ LayoutBox (盒模型)
  ├─ LayoutText (文本)
  └─ LayoutInline (内联)
```

### 样式系统
- **CSSSelector** - 选择器表示
- **StyleRule** - CSS 规则
- **ComputedStyle** - 计算后的样式

### 绘制系统
- **DisplayItem** - 显示项
- **PaintChunk** - 绘制块
- **PaintLayer** - 绘制层

## 核心 API 分类

### DOM 操作
- `appendChild()` / `removeChild()` / `insertBefore()`
- `getElementById()` / `querySelector()`
- `setAttribute()` / `getAttribute()`

### 样式操作
- `getComputedStyle()`
- `StyleEngine::UpdateStyle()`
- `StyleResolver::ResolveStyle()`

### 布局
- `LayoutObject::UpdateLayout()`
- `LayoutBox::ComputeLogicalHeight()`
- `NGBlockNode::Layout()`

### 绘制
- `PaintLayer::Paint()`
- `BoxPainter::PaintBoxDecorationBackground()`
- `PaintController::CommitNewDisplayItems()`

## 渲染流程简图

```
HTML 解析 → DOM 树
    ↓
CSS 解析 → CSSOM
    ↓
样式计算 → 带样式的 DOM
    ↓
布局计算 → 布局树
    ↓
预绘制 → 属性树
    ↓
绘制 → 显示项列表
    ↓
合成 → cc::Layer 列表
    ↓
光栅化 → 屏幕显示
```

## 实现 Emacs Lisp 版本建议

### 最小实现需要：
1. HTML 解析器 (`blink-parse-html`)
2. CSS 解析器 (`blink-parse-css`)
3. 样式计算器 (`blink-compute-styles`)
4. 布局计算器 (`blink-compute-layout`)
5. 绘制器 (`blink-paint`)

### 简化策略：
- 只支持基本选择器（标签、类、ID）
- 只实现块级和内联布局
- 跳过复杂 CSS 特性（Grid、Flexbox、动画）
- 简化事件系统（只实现冒泡）
- 使用 Emacs 内置垃圾回收

### 核心数据结构示例：
```elisp
(cl-defstruct blink-node
  type tag-name attributes
  parent first-child last-child
  previous-sibling next-sibling
  computed-style layout-object)

(cl-defstruct blink-style
  display position width height
  color background-color
  font-family font-size)

(cl-defstruct blink-layout-object
  node x y width height children)
```

## 参考资源

- 完整文档: `docs/blink_rendering_engine_analysis.md`
- 源码位置: `third_party/blink/renderer/`
- 官方文档: [How Blink Works](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg)
