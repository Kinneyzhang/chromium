# Blink 渲染引擎核心原理分析

本文档详细分析 Blink 渲染引擎的核心架构、数据结构和关键 API，按模块清晰划分，适用于用其他编程语言(如 Emacs Lisp)重新实现。

## 目录

1. [整体架构概述](#整体架构概述)
2. [核心模块](#核心模块)
3. [数据结构模型](#数据结构模型)
4. [关键 API](#关键-api)
5. [渲染管道流程](#渲染管道流程)
6. [模块依赖关系](#模块依赖关系)

---

## 整体架构概述

### Blink 的层次结构

Blink 渲染引擎位于 `third_party/blink/renderer/` 目录，采用分层架构：

```
┌─────────────────────────────────────────┐
│         controller/                     │  系统基础设施层
├─────────────────────────────────────────┤
│         extensions/                     │  嵌入器特定 API
├─────────────────────────────────────────┤
│         modules/                        │  Web 平台特性模块
├─────────────────────────────────────────┤
│   core/          bindings/core          │  核心渲染实现 + V8 绑定
├─────────────────────────────────────────┤
│         platform/                       │  底层平台抽象
├─────────────────────────────────────────┤
│    public/platform, public/common       │  公共 API
└─────────────────────────────────────────┘
```

### 渲染管道的四个核心阶段

Blink 的渲染过程可以分为四个关键阶段：

1. **DOM 构建** - 解析 HTML/XML 生成 DOM 树
2. **样式计算** - 解析 CSS 并应用到 DOM 节点
3. **布局计算** - 计算元素的几何位置和大小
4. **绘制渲染** - 生成显示列表并合成最终输出

---

## 核心模块

### 1. Platform 模块 (`platform/`)

**职责**: 提供底层平台抽象，不依赖 `core/` 或 `modules/`

**主要子模块**:

#### 1.1 WTF (Web Template Framework) - `platform/wtf/`

核心容器和数据类型：

- **容器类型**:
  - `Vector<T>` - 动态数组
  - `HashMap<K, V>` - 哈希表
  - `HashSet<T>` - 哈希集合
  - `LinkedHashSet<T>` - 保持插入顺序的哈希集合
  - `Deque<T>` - 双端队列

- **字符串类型**:
  - `String` - 标准字符串
  - `AtomicString` - 原子化字符串（共享存储，快速比较）
  - `StringBuilder` - 字符串构建器

- **智能指针**:
  - `scoped_refptr<T>` - 引用计数指针
  - `std::unique_ptr<T>` - 独占指针

#### 1.2 Graphics - `platform/graphics/`

图形和绘制基础设施：

- **Paint 子系统** (`platform/graphics/paint/`):
  - `PaintController` - 管理绘制操作
  - `DisplayItem` - 显示项基类
  - `DrawingDisplayItem` - 绘制显示项
  - `PaintChunk` - 共享属性树状态的显示项序列
  - `PaintArtifact` - 绘制输出结果

- **属性树** (Property Trees):
  - `TransformPaintPropertyNode` - 变换属性节点
  - `ClipPaintPropertyNode` - 裁剪属性节点
  - `EffectPaintPropertyNode` - 效果属性节点（透明度、混合等）
  - `ScrollPaintPropertyNode` - 滚动属性节点

#### 1.3 Scheduler - `platform/scheduler/`

任务调度系统：

- `TaskRunner` - 任务执行器
- `WebThreadScheduler` - Web 线程调度器
- `IdleTaskRunner` - 空闲任务执行器

#### 1.4 Heap - `platform/heap/`

垃圾回收内存管理（Oilpan）：

- `GarbageCollected<T>` - 垃圾回收对象基类
- `Member<T>` - 强引用指针
- `WeakMember<T>` - 弱引用指针
- `Persistent<T>` - 持久引用指针

### 2. Core 模块 (`core/`)

**职责**: 实现 Web 平台的核心功能

#### 2.1 DOM - `core/dom/`

**核心数据结构**:

```cpp
class Node : public EventTarget {
  // 节点类型
  enum NodeType {
    ELEMENT_NODE = 1,
    ATTRIBUTE_NODE = 2,
    TEXT_NODE = 3,
    CDATA_SECTION_NODE = 4,
    COMMENT_NODE = 8,
    DOCUMENT_NODE = 9,
    DOCUMENT_FRAGMENT_NODE = 11
  };
  
  // 树结构
  Node* parent_or_shadow_host_node_;
  Node* previous_sibling_;
  Node* next_sibling_;
  
  // 子节点
  virtual Node* FirstChild() const;
  virtual Node* LastChild() const;
  
  // 文档引用
  Document* GetDocument() const;
  
  // 生命周期
  virtual void Trace(Visitor*) const;
};
```

**Element 数据结构**:

```cpp
class Element : public ContainerNode {
  // 属性存储
  ElementData* element_data_;
  
  // 样式相关
  const ComputedStyle* GetComputedStyle() const;
  void SetComputedStyle(scoped_refptr<const ComputedStyle>);
  
  // 标签和属性
  const QualifiedName& TagQName() const;
  const AtomicString& GetIdAttribute() const;
  bool HasAttribute(const QualifiedName&) const;
  const AtomicString& getAttribute(const QualifiedName&) const;
  
  // 布局对象
  LayoutObject* GetLayoutObject() const;
};
```

**Document 数据结构**:

```cpp
class Document : public ContainerNode {
  // 文档元素
  Element* documentElement() const;
  
  // 样式引擎
  StyleEngine& GetStyleEngine();
  
  // 生命周期
  DocumentLifecycle lifecycle_;
  
  // 帧视图
  LocalFrameView* View() const;
  
  // 选择器查询
  Element* getElementById(const AtomicString& id);
  NodeList* querySelectorAll(const String& selectors);
};
```

**关键 API**:

- `Node::appendChild(Node*)` - 添加子节点
- `Node::removeChild(Node*)` - 移除子节点
- `Node::insertBefore(Node*, Node*)` - 在指定节点前插入
- `Element::setAttribute(name, value)` - 设置属性
- `Element::classList()` - 类列表操作
- `Document::createElement(tagName)` - 创建元素

#### 2.2 CSS - `core/css/`

**样式表数据结构**:

```cpp
class StyleSheet {
  // 样式表类型
  virtual String type() const = 0;
  virtual bool disabled() const = 0;
};

class CSSStyleSheet : public StyleSheet {
  // 规则集合
  CSSRuleList* cssRules();
  
  // 插入/删除规则
  unsigned insertRule(const String& rule, unsigned index);
  void deleteRule(unsigned index);
  
  // 规则存储
  HeapVector<Member<StyleSheetContents>> contents_;
};
```

**CSS 规则数据结构**:

```cpp
class CSSRule {
  enum Type {
    kStyleRule = 1,
    kImportRule = 3,
    kMediaRule = 4,
    kFontFaceRule = 5,
    kKeyframesRule = 7,
    // ...
  };
  
  virtual Type GetType() const = 0;
  virtual String cssText() const = 0;
};

class CSSStyleRule : public CSSRule {
  // 选择器
  String selectorText() const;
  void setSelectorText(const String&);
  
  // 样式声明
  CSSStyleDeclaration* style() const;
  
  // 内部表示
  StyleRule* style_rule_;
};
```

**样式解析器**:

```cpp
class StyleResolver {
  // 解析元素样式
  scoped_refptr<ComputedStyle> StyleForElement(
      Element*,
      const ComputedStyle* parent_style);
  
  // 匹配选择器
  void MatchAllRules(ElementResolveContext&,
                     StyleResolverState&);
  
  // 应用样式
  void ApplyMatchedProperties(StyleResolverState&,
                              const MatchResult&);
};
```

**计算样式**:

```cpp
class ComputedStyle {
  // 盒模型
  const Length& Width() const;
  const Length& Height() const;
  const LengthBox& Padding() const;
  const LengthBox& Margin() const;
  const BorderValue& BorderLeft() const;
  
  // 定位
  EPosition GetPosition() const;
  const Length& Top() const;
  const Length& Left() const;
  
  // 显示
  EDisplay Display() const;
  EVisibility Visibility() const;
  
  // 颜色和背景
  const Color& Color() const;
  const Color& BackgroundColor() const;
  
  // 变换
  const TransformOperations& Transform() const;
  
  // 字体
  const Font& GetFont() const;
};
```

**关键 API**:

- `StyleEngine::UpdateStyle()` - 更新样式
- `StyleResolver::ResolveStyle()` - 解析样式
- `CSSParser::ParseSheet()` - 解析样式表
- `CSSParser::ParseRule()` - 解析规则
- `CSSSelector::Match()` - 匹配选择器

#### 2.3 Layout - `core/layout/`

**布局对象基类**:

```cpp
class LayoutObject {
  // 节点引用
  Node* GetNode() const { return node_; }
  Document& GetDocument() const;
  
  // 样式
  const ComputedStyle& StyleRef() const;
  
  // 父子关系
  LayoutObject* Parent() const;
  LayoutObject* NextSibling() const;
  LayoutObject* PreviousSibling() const;
  LayoutObject* SlowFirstChild() const;
  
  // 几何信息
  virtual LayoutRect LocalVisualRect() const;
  virtual PhysicalRect PhysicalBorderBoxRect() const;
  
  // 布局标记
  bool NeedsLayout() const;
  void SetNeedsLayout(LayoutInvalidationReasonForTracing);
  
  // 布局计算
  virtual void UpdateLayout();
  virtual void ComputeIntrinsicLogicalWidths(
      LayoutUnit& min_logical_width,
      LayoutUnit& max_logical_width) const;
};
```

**盒模型布局对象**:

```cpp
class LayoutBox : public LayoutBoxModelObject {
  // 尺寸
  LayoutUnit Width() const;
  LayoutUnit Height() const;
  LayoutSize Size() const;
  
  // 位置
  LayoutPoint Location() const;
  PhysicalOffset PhysicalLocation() const;
  
  // 边框盒
  PhysicalRect PhysicalBorderBoxRect() const;
  LayoutRect BorderBoxRect() const;
  
  // 内容盒
  PhysicalRect PhysicalContentBoxRect() const;
  LayoutSize ContentSize() const;
  
  // 内边距
  LayoutRectOutsets BorderOutsets() const;
  LayoutRectOutsets PaddingOutsets() const;
  
  // 滚动
  PaintLayerScrollableArea* GetScrollableArea() const;
};
```

**布局算法 (LayoutNG)**:

```cpp
class NGBlockNode {
  // 布局入口
  scoped_refptr<const NGLayoutResult> Layout(
      const NGConstraintSpace& constraint_space,
      const NGBreakToken* break_token = nullptr);
  
  // 内在尺寸
  MinMaxSizesResult ComputeMinMaxSizes(
      WritingMode writing_mode,
      const NGConstraintSpace& constraint_space);
};

class NGConstraintSpace {
  // 可用空间
  LogicalSize AvailableSize() const;
  
  // 书写模式
  WritingMode GetWritingMode() const;
  
  // 百分比分辨率大小
  LayoutUnit PercentageResolutionSize() const;
};

class NGLayoutResult {
  // 物理片段
  const NGPhysicalFragment* PhysicalFragment() const;
  
  // 边界盒
  PhysicalRect PhysicalBorderBoxSize() const;
};
```

**关键 API**:

- `LayoutObject::UpdateLayout()` - 执行布局
- `LayoutBox::SetLogicalWidth()` - 设置宽度
- `LayoutBox::SetLogicalHeight()` - 设置高度
- `LayoutBox::ComputeLogicalHeight()` - 计算高度
- `NGBlockNode::Layout()` - LayoutNG 布局

#### 2.4 Paint - `core/paint/`

**绘制上下文**:

```cpp
class PaintInfo {
  // 绘制阶段
  enum Phase {
    kBlockBackground,
    kFloat,
    kForeground,
    kOutline,
    kSelfBlockBackground,
    kDescendantBlockBackgrounds,
    kSelectionDragImage,
    kTextClip,
    kMask
  };
  
  // 图形上下文
  GraphicsContext& context;
  
  // 裁剪矩形
  const gfx::Rect& GetCullRect() const;
  
  // 绘制阶段
  Phase phase;
};
```

**绘制器类**:

```cpp
class BoxPainter {
  // 绘制背景
  void PaintBackground(const PaintInfo&,
                       const PhysicalRect& paint_rect,
                       const Color& background_color);
  
  // 绘制边框
  void PaintBorder(const PaintInfo&,
                   const PhysicalRect& paint_rect,
                   const ComputedStyle&);
  
  // 绘制阴影
  void PaintBoxShadow(const PaintInfo&,
                      const ComputedStyle&);
};

class TextPainter {
  // 绘制文本
  void Paint(const TextFragmentPaintInfo&,
             const TextPaintStyle&,
             DOMNodeId node_id);
  
  // 绘制装饰线
  void PaintDecorations(const TextDecorationInfo&);
};
```

**绘制层**:

```cpp
class PaintLayer {
  // 层类型判断
  bool IsSelfPaintingLayer() const;
  bool IsRootLayer() const;
  
  // 绘制入口
  void Paint(GraphicsContext&,
             const GlobalPaintFlags);
  
  // 合成
  bool NeedsCompositedScrolling() const;
  CompositedLayerMapping* GetCompositedLayerMapping() const;
  
  // 栈上下文
  PaintLayerStackingNode* StackingNode();
};
```

**关键 API**:

- `PaintLayer::Paint()` - 绘制层
- `BoxPainter::PaintBoxDecorationBackground()` - 绘制盒装饰
- `TextPainter::Paint()` - 绘制文本
- `PaintController::CommitNewDisplayItems()` - 提交显示项

#### 2.5 Events - `core/events/`

**事件基类**:

```cpp
class Event {
  // 事件类型
  const AtomicString& type() const;
  
  // 事件目标
  EventTarget* target() const;
  EventTarget* currentTarget() const;
  
  // 事件阶段
  enum PhaseType {
    kNone = 0,
    kCapturingPhase = 1,
    kAtTarget = 2,
    kBubblingPhase = 3
  };
  PhaseType eventPhase() const;
  
  // 传播控制
  void stopPropagation();
  void stopImmediatePropagation();
  void preventDefault();
  
  // 时间戳
  DOMHighResTimeStamp timeStamp() const;
};
```

**鼠标事件**:

```cpp
class MouseEvent : public UIEvent {
  // 坐标
  int screenX() const;
  int screenY() const;
  int clientX() const;
  int clientY() const;
  int pageX() const;
  int pageY() const;
  
  // 按钮
  short button() const;
  unsigned short buttons() const;
  
  // 修饰键
  bool ctrlKey() const;
  bool shiftKey() const;
  bool altKey() const;
  bool metaKey() const;
};
```

**关键 API**:

- `EventTarget::addEventListener()` - 添加事件监听器
- `EventTarget::removeEventListener()` - 移除事件监听器
- `EventTarget::dispatchEvent()` - 分发事件
- `Event::stopPropagation()` - 停止传播

#### 2.6 Frame - `core/frame/`

**Frame 数据结构**:

```cpp
class LocalFrame : public Frame {
  // 文档
  Document* GetDocument() const;
  
  // 视图
  LocalFrameView* View() const;
  
  // 加载器
  FrameLoader& Loader() const;
  
  // 脚本控制器
  ScriptController& GetScriptController();
  
  // 选择
  FrameSelection& Selection();
  
  // 编辑
  Editor& GetEditor();
};
```

**FrameView 数据结构**:

```cpp
class LocalFrameView : public FrameView {
  // 生命周期更新
  void UpdateLifecycleToLayoutClean();
  void UpdateLifecyclePhasesInternal(
      DocumentLifecycle::LifecycleState target_state);
  
  // 布局
  void PerformLayout();
  void SetNeedsLayout();
  
  // 绘制
  void PaintTree();
  void UpdateLayerPositionsAfterLayout();
  
  // 尺寸
  gfx::Size GetLayoutSize() const;
  gfx::Size FrameRect() const;
};
```

### 3. Bindings 模块 (`bindings/`)

**职责**: 提供 JavaScript (V8) 和 C++ 之间的绑定

**核心组件**:

```cpp
// V8 包装器
class ScriptWrappable {
  v8::Local<v8::Object> Wrap(v8::Isolate*,
                             v8::Local<v8::Object> creation_context);
};

// 类型转换
class V8HTMLElement {
  static v8::Local<v8::Value> ToV8(HTMLElement*,
                                   v8::Local<v8::Object> creation_context,
                                   v8::Isolate*);
  static HTMLElement* ToImpl(v8::Local<v8::Object>);
};
```

### 4. Modules 模块 (`modules/`)

**职责**: 实现独立的 Web 平台特性

主要模块包括：

- `modules/accessibility/` - 无障碍访问
- `modules/webgl/` - WebGL
- `modules/canvas/` - Canvas 2D
- `modules/websockets/` - WebSocket
- `modules/fetch/` - Fetch API
- `modules/indexeddb/` - IndexedDB
- `modules/serviceworkers/` - Service Workers
- `modules/webrtc/` - WebRTC

---

## 数据结构模型

### 1. 树形数据结构

#### DOM 树结构

```
Document
  └── DocumentElement (html)
      ├── Head
      │   ├── Title (text node)
      │   ├── Style (text node with CSS)
      │   └── Script (text node with JS)
      └── Body
          ├── Div (element)
          │   ├── Text Node
          │   └── Span (element)
          │       └── Text Node
          └── Img (element)
```

**实现要点**:
- 每个节点维护父子兄弟关系
- 节点类型枚举 (元素、文本、注释等)
- 节点生命周期管理 (attached/detached)
- 事件冒泡和捕获路径

#### 布局树结构

```
LayoutView (document)
  └── LayoutBlockFlow (body)
      ├── LayoutBlockFlow (div)
      │   ├── LayoutText
      │   └── LayoutInline (span)
      │       └── LayoutText
      └── LayoutImage (img)
```

**实现要点**:
- 布局对象与 DOM 节点对应关系
- 匿名盒模型处理
- 浮动和定位布局对象
- Flex/Grid 容器

#### 绘制层树结构

```
PaintLayer (root)
  ├── PaintLayer (positioned div, z-index: 10)
  │   └── PaintLayer (child with opacity)
  └── PaintLayer (fixed element, z-index: 100)
```

**实现要点**:
- 层叠上下文 (stacking context)
- z-index 排序
- 合成层判断
- 层裁剪和遮罩

### 2. 样式系统数据结构

#### 选择器数据结构

```cpp
struct CSSSelector {
  enum Match {
    kTag,           // div
    kId,            // #id
    kClass,         // .class
    kPseudoClass,   // :hover
    kPseudoElement, // ::before
    kAttribute      // [attr=value]
  };
  
  Match match_;
  AtomicString value_;
  CSSSelector* next_;  // 组合器连接
};
```

#### 样式规则数据结构

```cpp
struct StyleRule {
  // 选择器列表
  Vector<CSSSelector> selectors_;
  
  // 属性声明
  CSSPropertyValueSet* properties_;
  
  // 优先级
  unsigned specificity_;
};
```

#### 计算样式存储

使用 Copy-on-Write 策略共享不可变样式：

```cpp
class ComputedStyle {
  // 非继承属性
  struct NonInheritedData {
    Color background_color_;
    Length width_;
    Length height_;
    // ...
  };
  
  // 继承属性
  struct InheritedData {
    Color color_;
    Font font_;
    Length line_height_;
    // ...
  };
  
  scoped_refptr<NonInheritedData> non_inherited_;
  scoped_refptr<InheritedData> inherited_;
};
```

### 3. 布局数据结构

#### 盒模型

```
┌─────────────────────────────────────┐
│           Margin                    │
│  ┌───────────────────────────────┐  │
│  │        Border                 │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │      Padding            │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │   Content         │  │  │  │
│  │  │  │                   │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**数据表示**:

```cpp
struct LayoutUnit {
  int value_;  // 以 1/64 像素为单位
};

struct PhysicalRect {
  PhysicalOffset offset;  // (x, y)
  PhysicalSize size;      // (width, height)
};

struct BoxStrut {
  LayoutUnit top;
  LayoutUnit right;
  LayoutUnit bottom;
  LayoutUnit left;
};
```

#### 约束空间 (NG Layout)

```cpp
struct NGConstraintSpace {
  // 可用空间
  LogicalSize available_size_;
  
  // 百分比分辨率大小
  LayoutUnit percentage_resolution_size_;
  
  // 书写模式
  WritingMode writing_mode_;
  
  // 文本方向
  TextDirection direction_;
  
  // 是否新格式化上下文
  bool is_new_formatting_context_;
};
```

### 4. 绘制数据结构

#### 显示项列表

```cpp
class DisplayItem {
  enum Type {
    kDrawing,
    kBoxDecorationBackground,
    kCaret,
    kClipPath,
    kScrollbar,
    // ...
  };
  
  DisplayItemClient& client_;
  Type type_;
  gfx::Rect visual_rect_;
};

class DrawingDisplayItem : public DisplayItem {
  sk_sp<PaintRecord> record_;
};
```

#### 绘制块 (Paint Chunk)

```cpp
struct PaintChunk {
  // 显示项范围
  size_t begin_index;
  size_t end_index;
  
  // 属性树状态
  PropertyTreeState properties;
  
  // 边界
  gfx::Rect bounds;
  
  // 命中测试数据
  HitTestData hit_test_data;
};
```

#### 属性树状态

```cpp
struct PropertyTreeState {
  const TransformPaintPropertyNode* transform;
  const ClipPaintPropertyNode* clip;
  const EffectPaintPropertyNode* effect;
};
```

---

## 关键 API

### 1. DOM 操作 API

```cpp
// 节点操作
Node* appendChild(Node* new_child);
Node* removeChild(Node* old_child);
Node* insertBefore(Node* new_child, Node* ref_child);
Node* replaceChild(Node* new_child, Node* old_child);
Node* cloneNode(bool deep);

// 元素查询
Element* getElementById(const AtomicString& id);
Element* querySelector(const String& selectors);
NodeList* querySelectorAll(const String& selectors);
HTMLCollection* getElementsByTagName(const String& name);
HTMLCollection* getElementsByClassName(const String& names);

// 属性操作
void setAttribute(const AtomicString& name, const AtomicString& value);
void removeAttribute(const AtomicString& name);
bool hasAttribute(const AtomicString& name) const;
const AtomicString& getAttribute(const AtomicString& name) const;

// 类操作
DOMTokenList* classList();
void className(const AtomicString& value);
```

### 2. 样式操作 API

```cpp
// 计算样式
CSSStyleDeclaration* getComputedStyle(Element*);

// 内联样式
CSSStyleDeclaration* style();

// 样式表操作
CSSStyleSheet* styleSheets.item(unsigned index);
unsigned insertRule(const String& rule, unsigned index);
void deleteRule(unsigned index);

// 样式引擎
void StyleEngine::UpdateStyle();
void StyleEngine::UpdateActiveStyle();
scoped_refptr<ComputedStyle> StyleResolver::ResolveStyle(Element*);
```

### 3. 布局 API

```cpp
// 布局触发
void LayoutObject::SetNeedsLayout(LayoutInvalidationReasonForTracing);
void LayoutObject::SetNeedsLayoutAndFullPaintInvalidation();

// 布局执行
void LocalFrameView::UpdateLayout();
void LayoutObject::UpdateLayout();

// 几何查询
PhysicalRect LayoutBox::PhysicalBorderBoxRect() const;
PhysicalOffset LayoutBox::PhysicalLocation() const;
LayoutSize LayoutBox::Size() const;

// LayoutNG API
scoped_refptr<const NGLayoutResult> NGBlockNode::Layout(
    const NGConstraintSpace&);
MinMaxSizesResult NGBlockNode::ComputeMinMaxSizes(
    WritingMode,
    const NGConstraintSpace&);
```

### 4. 绘制 API

```cpp
// 绘制入口
void LocalFrameView::PaintTree();
void PaintLayer::Paint(GraphicsContext&, const GlobalPaintFlags);

// 绘制器
void BoxPainter::Paint(const PaintInfo&);
void BoxPainter::PaintBoxDecorationBackground(const PaintInfo&);
void TextPainter::Paint(const TextFragmentPaintInfo&);

// 显示项
void PaintController::CreateAndAppend(DisplayItemClientId,
                                      DisplayItem::Type);

// 绘制块提交
void PaintController::CommitNewDisplayItems();
```

### 5. 事件 API

```cpp
// 事件监听
void addEventListener(const AtomicString& event_type,
                     EventListener*,
                     const AddEventListenerOptions*);
void removeEventListener(const AtomicString& event_type,
                         EventListener*,
                         const EventListenerOptions*);

// 事件分发
bool dispatchEvent(Event*);
void EventDispatcher::Dispatch();

// 事件创建
MouseEvent* MouseEvent::Create(const AtomicString& type,
                                const MouseEventInit*);
KeyboardEvent* KeyboardEvent::Create(const AtomicString& type,
                                      const KeyboardEventInit*);
```

### 6. 生命周期管理 API

```cpp
// 文档生命周期
enum LifecycleState {
  kUninitialized,
  kInactive,
  kVisualUpdatePending,
  kInStyleRecalc,
  kStyleClean,
  kInLayoutSubtreeChange,
  kLayoutSubtreeChangeClean,
  kInPreLayout,
  kInPerformLayout,
  kAfterPerformLayout,
  kLayoutClean,
  kInPrePaint,
  kPrePaintClean,
  kInPaint,
  kPaintClean,
  kStopping,
  kStopped
};

void DocumentLifecycle::AdvanceTo(LifecycleState);
bool DocumentLifecycle::StateAllowsTreeMutations() const;

// 生命周期更新
void LocalFrameView::UpdateLifecycleToLayoutClean();
void LocalFrameView::UpdateAllLifecyclePhasesExceptPaint();
void LocalFrameView::UpdateAllLifecyclePhases();
```

---

## 渲染管道流程

### 完整渲染流程

```
1. HTML 解析
   └→ HTMLDocumentParser::PumpTokenizer()
      └→ HTMLTreeBuilder::ProcessToken()
         └→ 构建 DOM 树

2. CSS 解析
   └→ CSSParser::ParseSheet()
      └→ 构建 CSSOM 树

3. 样式计算 (Style Recalc)
   └→ StyleEngine::UpdateStyle()
      └→ StyleResolver::ResolveStyle()
         ├→ 选择器匹配
         ├→ 层叠和继承
         └→ 计算最终样式

4. 布局 (Layout)
   └→ LocalFrameView::PerformLayout()
      └→ LayoutView::UpdateLayout()
         └→ LayoutObject::UpdateLayout()
            ├→ 计算尺寸
            ├→ 确定位置
            └→ 处理浮动和定位

5. 预绘制 (PrePaint)
   └→ PrePaintTreeWalk::Walk()
      ├→ PaintPropertyTreeBuilder::Update()
      │  └→ 构建属性树
      └→ PaintInvalidator::InvalidatePaint()
         └→ 标记需要重绘的区域

6. 绘制 (Paint)
   └→ LocalFrameView::PaintTree()
      └→ PaintLayer::Paint()
         └→ ObjectPainter::Paint()
            ├→ 生成显示项
            └→ 记录绘制操作

7. 合成 (Composite)
   └→ PaintArtifactCompositor::Update()
      ├→ 分层
      ├→ 转换属性树
      └→ 生成 cc::Layer 列表

8. 光栅化和显示
   └→ Compositor 线程处理
      └→ 最终渲染到屏幕
```

### 关键阶段详解

#### 阶段 1: 样式计算

```cpp
void StyleEngine::UpdateStyle() {
  // 1. 收集需要样式更新的节点
  CollectInvalidatedElementsForUpdate();
  
  // 2. 重新计算样式
  for (Element* element : invalidated_elements) {
    ResolveStyle(element);
  }
  
  // 3. 传播样式更改
  PropagateStyleChangesToDescendants();
}

scoped_refptr<ComputedStyle> StyleResolver::ResolveStyle(
    Element* element,
    const ComputedStyle* parent_style) {
  // 1. 创建样式解析状态
  StyleResolverState state(document, *element);
  
  // 2. 匹配规则
  MatchAllRules(state);
  
  // 3. 应用层叠
  ApplyCascade(state);
  
  // 4. 应用继承
  ApplyInheritance(state, parent_style);
  
  // 5. 调整样式
  AdjustStyle(state);
  
  return state.TakeStyle();
}
```

#### 阶段 2: 布局计算

```cpp
void LayoutObject::UpdateLayout() {
  if (!NeedsLayout())
    return;
  
  // 1. 布局子元素
  LayoutChildren();
  
  // 2. 计算自身尺寸
  ComputeSize();
  
  // 3. 定位子元素
  PositionChildren();
  
  // 4. 清除布局标记
  ClearNeedsLayout();
}

// LayoutNG 示例
scoped_refptr<const NGLayoutResult> NGBlockNode::Layout(
    const NGConstraintSpace& constraint_space) {
  // 1. 创建布局算法
  NGBlockLayoutAlgorithm algorithm(this, constraint_space);
  
  // 2. 布局子元素
  algorithm.LayoutChildren();
  
  // 3. 生成片段
  return algorithm.Layout();
}
```

#### 阶段 3: 绘制

```cpp
void PaintLayer::Paint(GraphicsContext& context,
                       const GlobalPaintFlags flags) {
  // 1. 设置裁剪和变换
  ScopedPaintChunkProperties scoped_properties(
      context, properties_);
  
  // 2. 绘制背景和边框
  if (ShouldPaint(kPaintPhaseBlockBackground))
    PaintBackgroundAndBorder();
  
  // 3. 绘制浮动元素
  if (ShouldPaint(kPaintPhaseFloat))
    PaintFloats();
  
  // 4. 绘制前景（内容）
  if (ShouldPaint(kPaintPhaseForeground))
    PaintForeground();
  
  // 5. 绘制轮廓
  if (ShouldPaint(kPaintPhaseOutline))
    PaintOutline();
}

void BoxPainter::PaintBoxDecorationBackground(const PaintInfo& paint_info) {
  // 1. 绘制背景色
  if (background_color.Alpha() > 0)
    PaintBackground(background_color);
  
  // 2. 绘制背景图
  if (HasBackgroundImage())
    PaintBackgroundImages();
  
  // 3. 绘制边框
  if (HasVisibleBorder())
    PaintBorder();
  
  // 4. 绘制盒阴影
  if (HasBoxShadow())
    PaintBoxShadow();
}
```

### 优化策略

#### 1. 样式共享

```cpp
// 使用共享计算样式避免重复计算
if (CanShareComputedStyle(element, parent_style)) {
  return GetSharedStyle(element);
}
```

#### 2. 增量布局

```cpp
// 只布局标记为需要布局的子树
if (!child->NeedsLayout() && !SelfNeedsLayout())
  continue;
```

#### 3. 绘制缓存

```cpp
// 重用上次绘制的显示项
if (DisplayItemClient->IsValid() && !NeedsRepaint()) {
  UseCachedDisplayItems();
}
```

#### 4. 子序列缓存

```cpp
// 缓存整个 PaintLayer 的绘制结果
if (ShouldCacheSubsequence()) {
  BeginSubsequence();
  PaintContents();
  EndSubsequence();
}
```

---

## 模块依赖关系

### 依赖层次图

```
                 ┌──────────────┐
                 │  controller/ │
                 └──────┬───────┘
                        │
                 ┌──────▼────────┐
                 │ extensions/   │
                 └──────┬────────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
┌───────▼────────┐             ┌───────▼────────┐
│   modules/     │             │ bindings/      │
└───────┬────────┘             │  modules       │
        │                      └───────┬────────┘
        │                              │
        └──────────────┬───────────────┘
                       │
             ┌─────────▼──────────┐
             │  core/ +           │
             │  bindings/core     │
             └─────────┬──────────┘
                       │
                ┌──────▼───────┐
                │  platform/   │
                └──────┬───────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
┌───────▼────────┐          ┌────────▼─────────┐
│ public/platform│          │ public/common    │
└────────────────┘          └──────────────────┘
```

### 模块间通信

#### 1. DOM → Style

```cpp
Element::RecalcStyle() {
  ComputedStyle* new_style = 
      document.GetStyleResolver()
          .ResolveStyle(this, parent_style);
  SetComputedStyle(new_style);
}
```

#### 2. Style → Layout

```cpp
Element::AttachLayoutTree() {
  LayoutObject* layout_object = 
      CreateLayoutObject(computed_style);
  SetLayoutObject(layout_object);
}
```

#### 3. Layout → Paint

```cpp
LayoutObject::Paint() {
  PaintInfo paint_info(...);
  ObjectPainter(*this).Paint(paint_info);
}
```

#### 4. Paint → Compositor

```cpp
LocalFrameView::PushPaintArtifactToCompositor() {
  paint_artifact_compositor_->Update(
      paint_controller_->GetPaintArtifact());
}
```

### 关键接口契约

#### ILayoutObject 接口

```cpp
class ILayoutObject {
  virtual void UpdateLayout() = 0;
  virtual PhysicalRect PhysicalBorderBoxRect() const = 0;
  virtual bool NeedsLayout() const = 0;
  virtual void SetNeedsLayout() = 0;
};
```

#### IPainter 接口

```cpp
class IPainter {
  virtual void Paint(const PaintInfo&) = 0;
  virtual void PaintBoxDecorationBackground(const PaintInfo&) = 0;
};
```

---

## 实现建议 (针对 Emacs Lisp)

### 1. 数据结构建议

使用 Emacs Lisp 的原生数据结构：

```elisp
;; Node 数据结构
(cl-defstruct blink-node
  type
  tag-name
  attributes
  parent
  first-child
  last-child
  previous-sibling
  next-sibling
  computed-style
  layout-object)

;; ComputedStyle 数据结构
(cl-defstruct blink-style
  display
  position
  width
  height
  color
  background-color
  font-family
  font-size)

;; LayoutObject 数据结构
(cl-defstruct blink-layout-object
  node
  x
  y
  width
  height
  children)
```

### 2. 核心算法简化

#### 样式计算

```elisp
(defun blink-compute-style (element parent-style)
  "计算元素的样式"
  (let ((style (make-blink-style)))
    ;; 1. 继承父样式
    (when parent-style
      (blink-inherit-style style parent-style))
    
    ;; 2. 应用匹配的规则
    (dolist (rule (blink-match-rules element))
      (blink-apply-rule style rule))
    
    ;; 3. 应用内联样式
    (blink-apply-inline-style style element)
    
    style))
```

#### 布局计算

```elisp
(defun blink-layout-box (layout-object available-width)
  "计算盒的布局"
  (let* ((node (blink-layout-object-node layout-object))
         (style (blink-node-computed-style node))
         (width (blink-compute-width style available-width))
         (height 0)
         (y-offset 0))
    
    ;; 设置宽度
    (setf (blink-layout-object-width layout-object) width)
    
    ;; 布局子元素
    (dolist (child (blink-layout-object-children layout-object))
      (blink-layout-box child width)
      (setf (blink-layout-object-y child) y-offset)
      (cl-incf y-offset (blink-layout-object-height child)))
    
    ;; 设置高度
    (setf (blink-layout-object-height layout-object) y-offset)))
```

#### 绘制

```elisp
(defun blink-paint-box (layout-object context)
  "绘制盒"
  (let ((style (blink-node-computed-style 
                (blink-layout-object-node layout-object))))
    
    ;; 绘制背景
    (when (blink-style-background-color style)
      (blink-paint-background layout-object context style))
    
    ;; 绘制边框
    (when (blink-has-border-p style)
      (blink-paint-border layout-object context style))
    
    ;; 绘制子元素
    (dolist (child (blink-layout-object-children layout-object))
      (blink-paint-box child context))))
```

### 3. 简化建议

对于 Emacs Lisp 实现，建议：

1. **简化选择器匹配**: 只支持基本选择器（标签、类、ID）
2. **简化布局算法**: 只实现块级布局和内联布局
3. **省略复杂特性**: 
   - 跳过复杂的 CSS 特性（Grid、Flexbox 可选）
   - 跳过动画和过渡
   - 跳过阴影和滤镜效果
4. **使用简化的事件系统**: 只实现冒泡机制
5. **内存管理**: 使用 Emacs 的垃圾回收，无需手动管理

### 4. 最小可行实现

一个最小的渲染引擎应包含：

```elisp
;; 1. HTML 解析器
(defun blink-parse-html (html-string))

;; 2. CSS 解析器
(defun blink-parse-css (css-string))

;; 3. 样式计算
(defun blink-compute-styles (dom-tree css-rules))

;; 4. 布局树构建
(defun blink-build-layout-tree (dom-tree))

;; 5. 布局计算
(defun blink-compute-layout (layout-tree viewport-width))

;; 6. 绘制
(defun blink-paint (layout-tree canvas))

;; 7. 主入口
(defun blink-render (html css viewport-width)
  (let* ((dom (blink-parse-html html))
         (rules (blink-parse-css css))
         (styled-dom (blink-compute-styles dom rules))
         (layout-tree (blink-build-layout-tree styled-dom))
         (positioned-tree (blink-compute-layout layout-tree viewport-width)))
    (blink-paint positioned-tree (current-buffer))))
```

---

## 参考资料

### 官方文档

1. [How Blink Works](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg)
2. [Life of a Pixel](http://bit.ly/lifeofapixel)
3. [LayoutNG Design](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/core/layout/ng/README.md)
4. [Platform Paint README](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/graphics/paint/README.md)

### 源码位置

- DOM: `third_party/blink/renderer/core/dom/`
- CSS: `third_party/blink/renderer/core/css/`
- Layout: `third_party/blink/renderer/core/layout/`
- Paint: `third_party/blink/renderer/core/paint/`
- Platform: `third_party/blink/renderer/platform/`

### Web 标准

- [HTML Living Standard](https://html.spec.whatwg.org/)
- [CSS Specifications](https://www.w3.org/Style/CSS/specs.en.html)
- [DOM Living Standard](https://dom.spec.whatwg.org/)

---

## 总结

Blink 渲染引擎是一个复杂但结构清晰的系统，主要特点：

1. **模块化设计**: 清晰的层次和依赖关系
2. **管道架构**: 解析 → 样式 → 布局 → 绘制 → 合成
3. **优化策略**: 缓存、增量更新、延迟计算
4. **内存管理**: 垃圾回收（Oilpan）和智能指针
5. **并发支持**: 主线程、合成线程、工作线程

对于 Emacs Lisp 实现，建议：
- 专注核心功能
- 简化复杂特性
- 使用原生数据结构
- 采用函数式风格
- 迭代式开发
