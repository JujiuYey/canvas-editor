# 教程 04：Draw 渲染引擎 —— 画布上的指挥官

> **前置知识**：建议先阅读教程 03（Editor 类），理解 Editor 如何创建 Draw 实例。

---

## 目录

1. [Draw 类是什么](#1-draw-类是什么)
2. [DOM 结构：Canvas 页面管理](#2-dom结构canvas-页面管理)
3. [子系统概览：40+ 个组件](#3-子系统概览40-个组件)
4. [核心渲染流程：render()](#4-核心渲染流程-render)
5. [computeRowList：排版的核心算法](#5-computerowlist排版的核心算法)
6. [页面绘制：_drawPage](#6-页面绘制-_drawpage)
7. [粒子系统：内容如何绘制](#7-粒子系统内容如何绘制)
8. [Frame 框架系统：页眉页脚水印](#8-frame-框架系统页眉页脚水印)
9. [位置系统：Position 与坐标映射](#9-位置系统position-与坐标映射)
10. [懒加载渲染：IntersectionObserver](#10-懒加载渲染-intersectionobserver)
11. [核心要点](#11-核心要点)

---

## 1. Draw 类是什么

`Draw` 类（`src/editor/core/draw/Draw.ts`）是编辑器**唯一的渲染引擎**，约 96KB，占整个编辑器代码量的大半。它负责：

- 在 Canvas 上绘制所有内容（文字、图片、表格、公式等）
- 管理文档的排版（分行、分页）
- 维护光标、选区、搜索高亮等视觉元素
- 处理用户输入事件（键盘、鼠标）

**与 Editor 的关系**：

```
Editor（编排层）
  └── Draw（渲染引擎）← 真正的核心
```

Editor 是一个薄的编排层，而 Draw 是真正干活的引擎。Editor 创建 Draw 时，将自己持有的通信基础设施（listener、eventBus、override）传给 Draw，让 Draw 可以在状态变化时通知外部。

---

## 2. DOM 结构：Canvas 页面管理

Draw 在容器中创建了两层 DOM 结构：

### 2.1 容器包装

```typescript
// src/editor/core/draw/Draw.ts, lines 208-222
this.container = this._wrapContainer(rootContainer)  // 包装原始容器
this.pageContainer = this._createPageContainer()     // 创建页面容器
```

创建的结构：

```
rootContainer（用户提供的 div）
  └── .editor-container（Draw 包装层）
        ├── .page-container（页面容器，flex 布局）
        │     ├── .page-canvas（每页一个）
        │     ├── .page-canvas
        │     └── ...
        ├── .cursor（光标层，position: absolute）
        ├── .search-mark（搜索高亮层）
        ├── .group-mark（批注高亮层）
        ├── .table-tool（表格工具层）
        └── .control.Float-control（浮动控件层）
```

### 2.2 多页 Canvas

```typescript
// src/editor/core/draw/Draw.ts, lines 2709-2796
// render 时动态创建/删除页面
for (let i = 0; i < this.pageRowList.length; i++) {
  if (!this.pageList[i]) {
    this._createPage(i)  // 创建新的 Canvas
  }
}
// 移除多余的页面
if (prePageCount > curPageCount) {
  this.pageList.splice(curPageCount, deleteCount).forEach(page => page.remove())
}
```

每页是一个独立的 `<canvas>` 元素，所有页面在 `.page-container` 中垂直排列（分页模式）或共享一个页面（连续模式）。

---

## 3. 子系统概览：40+ 个组件

Draw 的构造器创建了 40+ 个子系统，分为六类：

### 3.1 核心管理器（5 个）

| 组件 | 类 | 职责 |
|------|-----|------|
| `position` | `Position` | 光标/元素位置计算、坐标映射 |
| `range` | `RangeManager` | 选区管理（start/end 索引、样式） |
| `historyManager` | `HistoryManager` | 撤销/重做历史栈 |
| `zone` | `Zone` | 页眉/主体/页脚区域管理 |
| `workerManager` | `WorkerManager` | Web Worker 异步任务调度 |

### 3.2 粒子系统（20+ 个）

每个 Particle 负责绘制一种内容类型：

| 粒子 | 绘制内容 |
|------|---------|
| `textParticle` | 普通文本 |
| `imageParticle` | 图片 |
| `tableParticle` | 表格 |
| `hyperlinkParticle` | 超链接 |
| `laTexParticle` | LaTeX 公式 |
| `listParticle` | 列表标记 |
| `separatorParticle` | 分隔线 |
| `pageBreakParticle` | 分页符 |
| `checkboxParticle` | 复选框 |
| `radioParticle` | 单选框 |
| `dateParticle` | 日期控件 |
| `labelParticle` | 标签 |
| `superscriptParticle` | 上标 |
| `subscriptParticle` | 下标 |
| `blockParticle` | 块级元素 |
| `lineBreakParticle` | 换行符 |
| `whiteSpaceParticle` | 空白字符 |

### 3.3 框架系统（10 个）

| 组件 | 绘制内容 |
|------|---------|
| `header` | 页眉区域 |
| `footer` | 页脚区域 |
| `margin` | 页边距指示器 |
| `background` | 页面背景 |
| `pageNumber` | 页码 |
| `watermark` | 水印 |
| `placeholder` | 空占位符 |
| `lineNumber` | 行号 |
| `pageBorder` | 页面边框 |
| `badge` | 批注标记 |

### 3.4 富文本装饰（3 个）

| 组件 | 装饰内容 |
|------|---------|
| `underline` | 下划线 |
| `strikeout` | 删除线 |
| `highlight` | 高亮背景 |

### 3.5 交互系统（3 个）

| 组件 | 交互内容 |
|------|---------|
| `search` | 搜索与高亮 |
| `group` | 批注分组 |
| `area` | 区域划分 |

### 3.6 观察者（4 个）

| 组件 | 观察内容 |
|------|---------|
| `scrollObserver` | 滚动事件 |
| `selectionObserver` | 选区变化 |
| `imageObserver` | 图片懒加载 |
| `mouseObserver` | 鼠标事件 |

---

## 4. 核心渲染流程：render()

`render()` 是 Draw 的核心方法，编辑器的任何状态变化（输入文字、删除、格式化等）最终都会调用它。

### 4.1 方法签名与参数

```typescript
// src/editor/core/draw/Draw.ts, lines 2709-2720
public render(payload?: IDrawOption) {
  const {
    isSubmitHistory = true,      // 是否提交历史记录
    isSetCursor = true,         // 是否设置光标
    isCompute = true,            // 是否重新计算排版
    isLazy = true,               // 是否懒渲染
    isInit = false,              // 是否首次渲染
    isSourceHistory = false,    // 是否来自历史记录（撤销/重做）
    isFirstRender = false        // 是否首次渲染
  } = payload || {}
}
```

### 4.2 完整渲染流程

```
render()
│
├─ 1. renderCount++              渲染计数++
│
├─ 2. isCompute=true 时：         重新计算排版
│   ├─ position.setFloatPositionList([])   清空浮动元素位置
│   ├─ header.compute()            计算页眉
│   ├─ footer.compute()           计算页脚
│   ├─ this.rowList = computeRowList()     计算行列表【核心】
│   ├─ this.pageRowList = _computePageList()  计算页面列表
│   ├─ position.computePositionList()     计算位置列表
│   ├─ area.compute()              计算区域信息
│   ├─ search.compute()           计算搜索高亮
│   └─ graffiti.compute()         计算涂鸦
│
├─ 3. cursor.recoveryCursor()     恢复光标状态
│
├─ 4. 页面管理
│   ├─ 创建缺失的页面 Canvas
│   └─ 删除多余的页面 Canvas
│
├─ 5. 实际绘制
│   ├─ isLazy && isPagingMode → _lazyRender()   懒渲染
│   └─ 否则 → _immediateRender()                立即渲染
│
├─ 6. 光标重绘
│   └─ isSetCursor → setCursor()     设置光标位置
│
├─ 7. 提交历史记录
│   └─ isSubmitHistory && !isFirstRender → submitHistory()
│
└─ 8. nextTick 后执行回调
    ├─ range.setRangeStyle()              更新选区样式
    ├─ listener.pageSizeChange            页数变化回调
    └─ listener.contentChange              内容变化回调
```

### 4.3 渲染触发的时机

编辑器中有大量操作会触发 `render()`：

```typescript
// 文本输入后
this.render({ isSubmitHistory: true, isSetCursor: true })

// 文字格式化后（加粗、颜色等）
this.render({ isSubmitHistory: true, isSetCursor: false })

// 页面滚动后（懒加载渲染）
this.render({ isCompute: false, isSetCursor: false, isLazy: true })

// 撤销/重做后
this.render({ isSubmitHistory: false, isSetCursor: true, isSourceHistory: true })
```

---

## 5. computeRowList：排版的核心算法

`computeRowList` 是 Draw 最复杂的算法之一，负责将元素数组转换为行数组。

### 5.1 方法签名

```typescript
// src/editor/core/draw/Draw.ts, line 1377
public computeRowList(payload: IComputeRowListPayload): IRow[]
```

### 5.2 关键概念

**IRow 接口**：

```typescript
interface IRow {
  width: number              // 行宽
  height: number             // 行高
  ascent: number             // 基线上方高度
  elementList: IRowElement[] // 行内元素列表
  startIndex: number         // 起始元素索引
  rowIndex: number           // 行号
  rowFlex: RowFlex           // 行对齐方式
  offsetX?: number           // 偏移X（列表缩进）
  offsetY?: number           // 偏移Y（段落间距）
}
```

### 5.3 排版算法核心逻辑

```typescript
// src/editor/core/draw/Draw.ts, lines 1422-1438
for (let i = 0; i < elementList.length; i++) {
  const element = elementList[i]
  const curRow: IRow = rowList[rowList.length - 1]

  // 计算可用宽度
  const availableWidth = innerWidth - offsetX

  // 判断是否需要换行
  if (x + elementWidth > availableWidth) {
    // 换行：创建新行
    x = startX
    y += curRow.height + rowMargin
    rowList.push({ ... })
  }

  // 将元素添加到当前行
  curRow.elementList.push(element)
  x += elementWidth
}
```

### 5.4 元素尺寸计算

不同类型元素的尺寸计算方式不同：

```typescript
// 文本元素：用 Canvas measureText 测量
const metrics = ctx.measureText(text)
element.metrics = {
  width: metrics.width,
  height: fontSize,
  boundingBoxAscent: metrics.actualBoundingBoxAscent,
  boundingBoxDescent: metrics.actualBoundingBoxDescent
}

// 图片元素：直接使用 width/height 属性（已缩放）
metrics.width = element.width * scale
metrics.height = element.height * scale

// 表格元素：递归计算内部行列
this.tableParticle.computeRowColInfo(element)
```

### 5.5 分页处理

当 `isPagingMode=true` 时，`computeRowList` 会在页面放不下时自动分页：

```typescript
// src/editor/core/draw/Draw.ts, lines 1442-1460
if (isPagingMode && y + elementHeight > pageHeight - mainOuterHeight) {
  // 超出页面高度，创建新页
  pageNo++
  x = startX
  y = margins[0] + headerExtraHeight
  rowList.push({ ... }) // 新页的第一行
}
```

---

## 6. 页面绘制：_drawPage

`_drawPage` 负责在单个 Canvas 上绘制一页的内容。

### 6.1 方法签名

```typescript
// src/editor/core/draw/Draw.ts, line 2571
private _drawPage(payload: IDrawPagePayload) {
  const { elementList, positionList, rowList, pageNo } = payload
  const ctx = this.ctxList[pageNo]
  const canvas = this.pageList[pageNo]
}
```

### 6.2 绘制顺序

页面元素按以下顺序绘制（后绘制的在上层）：

```
1. background     页面背景
2. margin        页边距
3. header        页眉（如果跨页）
4. lineNumber    行号
5. watermark     水印（半透明）
6. 逐行绘制：
   ├─ row elementList  每行元素
   │   ├─ textParticle
   │   ├─ imageParticle
   │   ├─ tableParticle
   │   └─ ... 其他粒子
   ├─ underline        下划线
   ├─ strikeout        删除线
   └─ highlight        高亮
7. pageBorder     页面边框
8. footer         页脚（如果跨页）
9. pageNumber     页码
```

### 6.3 逐行绘制实现

```typescript
// src/editor/core/draw/Draw.ts, lines 2571-2707
_drawPage(payload: IDrawPagePayload) {
  const { rowList, pageNo } = payload
  const ctx = this.ctxList[pageNo]

  // 清空画布
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // 1. 绘制背景
  this.background.draw(ctx, pageNo)

  // 2. 逐行绘制内容
  for (let rowIndex = 0; rowIndex < rowList.length; rowIndex++) {
    const row = rowList[rowIndex]
    for (const element of row.elementList) {
      // 调用对应粒子的 draw 方法
      this.getParticleByElement(element).draw(ctx, element, row)
    }
  }

  // 3. 绘制装饰层（下划线、高亮等）
  this.underline.draw(ctx, rowList)
  this.highlight.draw(ctx, rowList)

  // 4. 绘制覆盖层（水印、页码等）
  this.waterMark.draw(ctx, pageNo)
  this.pageNumber.draw(ctx, pageNo)
}
```

---

## 7. 粒子系统：内容如何绘制

每个 Particle 都知道如何绘制特定类型的内容。

### 7.1 Particle 接口

```typescript
interface IParticle {
  draw(ctx: CanvasRenderingContext2D, element: IElement, row: IRow): void
}
```

### 7.2 获取对应粒子

Draw 通过元素类型找到对应的粒子：

```typescript
// src/editor/core/draw/Draw.ts, lines 2449-2501
private getParticleByElement(element: IElement): IParticle {
  switch (element.type) {
    case ElementType.TEXT:
    case ElementType.CONTROL:
      return this.textParticle
    case ElementType.IMAGE:
      return this.imageParticle
    case ElementType.TABLE:
      return this.tableParticle
    case ElementType.HYPERLINK:
      return this.hyperlinkParticle
    case ElementType.LATEX:
      return this.laTexParticle
    case ElementType.LIST:
      return this.listParticle
    case ElementType.SEPARATOR:
      return this.separatorParticle
    case ElementType.PAGE_BREAK:
      return this.pageBreakParticle
    case ElementType.CHECKBOX:
      return this.checkboxParticle
    case ElementType.RADIO:
      return this.radioParticle
    case ElementType.DATE:
      return this.dateParticle
    case ElementType.LABEL:
      return this.labelParticle
    case ElementType.SUPERSCRIPT:
      return this.superscriptParticle
    case ElementType.SUBSCRIPT:
      return this.subscriptParticle
    case ElementType.BLOCK:
      return this.blockParticle
    case ElementType.LINE_BREAK:
      return this.lineBreakParticle
    case ElementType.WHITE_SPACE:
      return this.whiteSpaceParticle
    default:
      return this.textParticle
  }
}
```

### 7.3 典型粒子实现：TextParticle

TextParticle 是最常用的粒子，负责绘制普通文本：

```typescript
// src/editor/core/draw/particle/TextParticle.ts
draw(ctx: CanvasRenderingContext2D, element: IElement, row: IRow) {
  // 1. 设置字体样式
  ctx.font = `${element.bold ? 'bold ' : ''}${element.size * scale}px ${element.font}`

  // 2. 设置颜色
  ctx.fillStyle = element.color || '#000'

  // 3. 计算绘制位置
  const x = element.x + positionContext.x
  const y = row.y + row.ascent

  // 4. 绘制文本
  ctx.fillText(element.value, x, y)

  // 5. 绘制下划线/删除线（如果需要）
  if (element.underline) {
    this.drawUnderline(ctx, element, row)
  }
}
```

---

## 8. Frame 框架系统：页眉页脚水印

Frame 系统负责绘制文档的"框架"元素，与内容无关但属于页面视觉的一部分。

### 8.1 Header 和 Footer

Header 和 Footer 各自维护独立的元素列表和行列表：

```typescript
// Draw 构造器中
this.header = new Header(this, data.header)  // 传入页眉数据
this.footer = new Footer(this, data.footer)  // 传入页脚数据

// Header/Footer 内部也使用 computeRowList 计算自己的排版
// src/editor/core/draw/frame/Header.ts, lines 51-67
private _computeRowList() {
  this.rowList = this.draw.computeRowList({
    startX: margins[3],
    startY: margins[0],
    pageHeight: pageHeight,
    mainOuterHeight: margins[0] + margins[2],
    isPagingMode: true,
    innerWidth: this.draw.getInnerWidth(),
    elementList: this.elementList
  })
}
```

### 8.2 Watermark 水印

水印是一种视觉装饰，通常用于文档背景：

```typescript
// src/editor/core/draw/frame/Watermark.ts
draw(ctx: CanvasRenderingContext2D, pageNo: number) {
  const { data, size, color, opacity } = this.options.watermark
  ctx.save()
  ctx.globalAlpha = opacity
  ctx.font = `${size}px Arial`
  ctx.fillStyle = color
  ctx.rotate(-45 * Math.PI / 180)  // 旋转 45 度

  // 在页面上平铺水印
  for (let x = -pageWidth; x < pageWidth; x += spacing) {
    for (let y = -pageHeight; y < pageHeight; y += spacing) {
      ctx.fillText(data, x, y)
    }
  }
  ctx.restore()
}
```

### 8.3 PageNumber 页码

页码显示在页面底部中央：

```typescript
// src/editor/core/draw/frame/PageNumber.ts
draw(ctx: CanvasRenderingContext2D, pageNo: number) {
  const format = this.options.pageNumber.format
  const text = format
    .replace('{pageNo}', pageNo + 1)
    .replace('{pageCount}', totalPages)

  ctx.font = '12px Arial'
  ctx.fillStyle = '#000'
  ctx.textAlign = 'center'
  ctx.fillText(text, pageWidth / 2, pageHeight - 50)
}
```

---

## 9. 位置系统：Position 与坐标映射

Position 负责维护元素和光标的精确位置信息。

### 9.1 Position 的职责

```typescript
// src/editor/core/position/Position.ts
export class Position {
  // 从元素索引到坐标的转换
  indexToPosition(index: number): IPosition

  // 从坐标到元素索引的转换
  positionToIndex(x: number, y: number): number

  // 计算所有元素的位置列表
  computePositionList(): void

  // 获取当前光标位置上下文
  getPositionContext(): IPositionContext
}
```

### 9.2 坐标映射流程

```
用户点击 Canvas
    ↓
GlobalEvent/MouseObserver 捕获坐标 (x, y)
    ↓
Position.positionToIndex(x, y)
    ↓
二分查找找到最近的元素索引
    ↓
RangeManager 更新选区
    ↓
Cursor 移动到新位置
```

### 9.3 位置列表

`computePositionList()` 为每个元素计算精确的绘制坐标：

```typescript
// position.computePositionList() 遍历所有行和元素
// 为每个元素设置 x, y, width, height
for (const row of rowList) {
  for (const element of row.elementList) {
    element.x = x
    element.y = row.y
    element.width = metrics.width
    element.height = metrics.height
  }
}
```

---

## 10. 懒加载渲染：IntersectionObserver

编辑器使用 IntersectionObserver 实现懒加载渲染，只绘制可见页。

### 10.1 懒加载策略

```typescript
// src/editor/core/draw/Draw.ts, lines 2700-2706
if (isLazy && isPagingMode) {
  this._lazyRender()
} else {
  this._immediateRender()
}
```

### 10.2 懒渲染实现

```typescript
// src/editor/core/observer/ScrollObserver.ts
constructor(draw: Draw) {
  // 创建 IntersectionObserver
  this.intersectionObserver = new IntersectionObserver(
    (entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          // 可见时触发绘制
          this.draw.render({
            isCompute: false,
            isSetCursor: false,
            isLazy: true
          })
        }
      })
    },
    { root: this.scrollElement }
  )

  // 观察所有页面
  this.pageList.forEach(page => {
    this.intersectionObserver.observe(page)
  })
}
```

### 10.3 可见性判断

每页 Canvas 只在该页进入视口时才渲染内容：

```
滚动 → ScrollObserver 检测到页面可见性变化
    ↓
_isVisible(pageIndex) 返回 true/false
    ↓
_immediateRender() 只绘制可见页
```

---

## 11. 核心要点

1. **Draw 是渲染引擎**——它不持有命令接口，而是直接管理 Canvas、元素和排版。

2. **render() 是核心方法**——几乎所有状态变化最终都会调用它，它协调 computeRowList、_drawPage 和粒子系统。

3. **40+ 子系统各司其职**——粒子负责内容、Frame 负责框架、交互系统负责用户功能、观察者负责事件。

4. **computeRowList 是排版核心**——它将一维元素数组转换为二维行数组，是最复杂的算法之一。

5. **粒子系统是绘制基础**——每种内容类型对应一个 Particle，通过简单接口 `draw(ctx, element, row)` 实现。

6. **多 Canvas 分层**——每页一个 Canvas，光标/搜索/批注等覆盖层使用 DOM 或独立 Canvas。

7. **懒加载优化性能**——IntersectionObserver 只渲染可见页，避免一次性绘制大量内容。

---

## 下一步

现在你已经理解了渲染引擎如何工作，下一步可以深入 **Command 命令系统**（教程 05）了解操作如何实现，或者深入 **粒子系统**（教程 06）了解每种内容类型如何绘制。

**值得思考的问题**：

- `computeRowList` 如何处理表格跨页拆分？
- 搜索高亮是如何在已渲染的页面上叠加的？
- 撤销/重做如何与 render 流程配合工作？
- 控件（如日期选择器）是如何浮在 Canvas 之上的？

这些问题将在后续教程中一一解答。
