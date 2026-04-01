# 教程 09：Particle 粒子系统 —— 内容如何绘制

> **前置知识**：建议先阅读教程 04（Draw 渲染引擎），理解 Draw 如何调用粒子系统进行绘制。

---

## 目录

1. [概述：粒子系统的设计](#1-概述粒子系统的设计)
2. [粒子分类总览](#2-粒子分类总览)
3. [粒子与元素的对应关系](#3-粒子与元素的对应关系)
4. [TextParticle：文本绘制的优化](#4-textparticle文本绘制的优化)
5. [ImageParticle：图片加载与缓存](#5-imageparticle图片加载与缓存)
6. [TableParticle：复杂表格的绘制](#6-tableparticle复杂表格的绘制)
7. [LaTeXParticle：数学公式渲染](#7-latexparticle数学公式渲染)
8. [ListParticle：列表标记](#8-listparticle列表标记)
9. [HyperlinkParticle：超链接装饰](#9-hyperlinkparticle超链接装饰)
10. [ControlParticle：表单控件](#10-controlparticle表单控件)
11. [核心要点](#11-核心要点)

---

## 1. 概述：粒子系统的设计

Particle（粒子）是 Draw 渲染引擎的核心组件，每个 Particle 负责绘制特定类型的内容。

### 1.1 设计理念

```
Draw._drawPage()
    ↓
for (row of rowList) {
  for (element of row.elementList) {
    const particle = getParticleByElement(element)
    particle.draw(ctx, element, row)  ← 调用粒子的 draw 方法
  }
}
```

### 1.2 核心接口

所有粒子都实现相同的接口：

```typescript
interface IParticle {
  draw(
    ctx: CanvasRenderingContext2D,  // Canvas 绘图上下文
    element: IElement,              // 要绘制的元素
    row: IRow                      // 所在行信息
  ): void
}
```

### 1.3 粒子文件结构

```
src/editor/core/draw/particle/
├── TextParticle.ts           ← 普通文本（最常用）
├── ImageParticle.ts          ← 图片
├── TableParticle.ts         ← 表格（最复杂）
├── ListParticle.ts          ← 列表标记
├── HyperlinkParticle.ts     ← 超链接
├── LaTeXParticle.ts        ← LaTeX 数学公式
├── SeparatorParticle.ts     ← 分隔线
├── PageBreakParticle.ts     ← 分页符
├── CheckboxParticle.ts      ← 复选框
├── RadioParticle.ts         ← 单选框
├── DateParticle.ts          ← 日期控件
├── LabelParticle.ts         ← 标签
├── SuperscriptParticle.ts   ← 上标
├── SubscriptParticle.ts     ← 下标
├── LineBreakParticle.ts     ← 换行符
├── WhiteSpaceParticle.ts    ← 空白字符
├── block/                   ← 块级元素（iframe、视频等）
│   ├── BlockParticle.ts
│   └── modules/
│       ├── BaseBlock.ts
│       ├── IFrameBlock.ts
│       └── VideoBlock.ts
└── table/                   ← 表格相关
    ├── TableParticle.ts
    ├── TableOperate.ts
    └── TableTool.ts
```

---

## 2. 粒子分类总览

### 2.1 内容粒子

| 粒子 | 绘制内容 | 复杂度 |
|-----|---------|-------|
| `textParticle` | 普通文本 | 中 |
| `imageParticle` | 图片（含浮动、环绕） | 高 |
| `tableParticle` | 表格（含合并单元格） | 极高 |
| `hyperlinkParticle` | 超链接下划线 | 低 |
| `laTexParticle` | LaTeX 公式（SVG 转图片） | 中 |
| `listParticle` | 列表标记（序号、圆点等） | 中 |
| `separatorParticle` | 分隔线 | 低 |
| `pageBreakParticle` | 分页符 | 低 |
| `checkboxParticle` | 复选框 | 低 |
| `radioParticle` | 单选框 | 低 |
| `dateParticle` | 日期控件 | 高 |
| `labelParticle` | 标签 | 低 |
| `superscriptParticle` | 上标 | 低 |
| `subscriptParticle` | 下标 | 低 |
| `blockParticle` | 块级元素（iframe/视频） | 高 |

### 2.2 渲染顺序

页面元素的绘制顺序（后绘制的在上层）：

```
1. background        页面背景
2. margin          页边距
3. header          页眉
4. lineNumber       行号
5. watermark        水印（半透明）
6. 逐行绘制：
   ├─ row elementList  每行元素
   │   ├─ textParticle
   │   ├─ imageParticle
   │   ├─ tableParticle
   │   └─ ... 其他粒子
   ├─ listParticle    列表标记（在文字之前）
   ├─ underline      下划线
   ├─ strikeout      删除线
   └─ highlight      高亮
7. pageBorder       页面边框
8. footer           页脚
9. pageNumber       页码
```

---

## 3. 粒子与元素的对应关系

Draw 通过元素类型找到对应的粒子：

```typescript
// src/editor/core/draw/Draw.ts, lines 2449-2477
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

---

## 4. TextParticle：文本绘制的优化

`TextParticle`（`src/editor/core/draw/particle/TextParticle.ts`）是最常用的粒子，采用**批量绘制优化**。

### 4.1 批量绘制原理

为了减少 Canvas 状态切换，TextParticle 不会每字符单独绘制，而是将**相同样式的连续字符**合并绘制：

```typescript
// src/editor/core/draw/particle/TextParticle.ts, lines 128-159
public record(
  ctx: CanvasRenderingContext2D,
  element: IRowElement,
  x: number,
  y: number
) {
  this.ctx = ctx
  // 样式发生改变时，先完成之前的绘制
  if (
    (this.curStyle && element.style !== this.curStyle) ||
    element.color !== this.curColor
  ) {
    this.complete()  // 绘制之前累积的文本
    this._setCurXY(x, y)  // 重置位置
  }
  this.text += element.value  // 累积文本
  this.curStyle = element.style
  this.curColor = element.color
}

// 完成绘制
private _render() {
  if (!this.text || !~this.curX || !~this.curY) return
  this.ctx.save()
  this.ctx.font = this.curStyle
  this.ctx.fillStyle = this.curColor || this.options.defaultColor
  this.ctx.fillText(this.text, this.curX, this.curY)  // 一次绘制整个字符串
  this.ctx.restore()
}
```

### 4.2 文本测量与缓存

```typescript
// src/editor/core/draw/particle/TextParticle.ts, lines 88-114
public measureText(
  ctx: CanvasRenderingContext2D,
  element: IElement
): ITextMetrics {
  const id = `${element.value}${ctx.font}`
  const cacheTextMetrics = this.cacheMeasureText.get(id)
  if (cacheTextMetrics) {
    return cacheTextMetrics  // 命中缓存
  }
  const textMetrics = ctx.measureText(element.value)
  this.cacheMeasureText.set(id, textMetrics)  // 写入缓存
  return textMetrics
}
```

### 4.3 度量基础字

```typescript
public measureBasisWord(ctx, font): ITextMetrics {
  ctx.save()
  ctx.font = font
  const textMetrics = this.measureText(ctx, {
    value: METRICS_BASIS_TEXT  // "BaJPGyj"
  })
  ctx.restore()
  return textMetrics
}
```

---

## 5. ImageParticle：图片加载与缓存

`ImageParticle`（`src/editor/core/draw/particle/ImageParticle.ts`）处理图片的加载、缓存和绘制。

### 5.1 图片缓存

```typescript
// src/editor/core/draw/particle/ImageParticle.ts, lines 12-24
export class ImageParticle {
  protected imageCache: Map<string, HTMLImageElement>

  public render(ctx, element, x, y) {
    const width = element.width * scale
    const height = element.height * scale

    if (this.imageCache.has(element.value)) {
      // 缓存命中，直接绘制
      const img = this.imageCache.get(element.value)!
      ctx.drawImage(img, x, y, width, height)
    } else {
      // 缓存未命中，创建 Image 对象加载
      const img = new Image()
      img.setAttribute('crossOrigin', 'Anonymous')
      img.src = element.value
      img.onload = () => {
        this.imageCache.set(element.value, img)
        ctx.drawImage(img, x, y, width, height)
      }
      img.onerror = () => {
        // 加载失败显示占位图
        const fallbackImage = this.getFallbackImage(width, height)
        // ...
      }
    }
  }
}
```

### 5.2 图片裁剪

```typescript
// src/editor/core/draw/particle/ImageParticle.ts, lines 142-172
private _drawImageWithCrop(
  ctx, img, element, x, y, width, height
) {
  if (element.imgCrop) {
    const { x: cropX, y: cropY, width: cropWidth, height: cropHeight } = element.imgCrop
    ctx.drawImage(
      img,
      cropX, cropY, cropWidth, cropHeight,  // 源图像裁剪区域
      x, y, width, height                     // 目标绘制区域
    )
  } else {
    ctx.drawImage(img, x, y, width, height)
  }
}
```

### 5.3 图片说明文字

```typescript
// src/editor/core/draw/particle/ImageParticle.ts, lines 174-221
private _renderCaption(ctx, element, x, y, width, height) {
  if (!element.imgCaption?.value) return
  const captionText = element.imgCaption.value
  ctx.save()
  ctx.font = `${fontSize}px ${fontFamily}`
  ctx.fillStyle = color
  ctx.textAlign = 'center'
  // 超出宽度时省略
  let displayText = captionText
  if (textMetrics.width > width) {
    displayText = captionText.substring(0, left) + '...'
  }
  ctx.fillText(displayText, captionX, captionY)
  ctx.restore()
}
```

### 5.4 异步加载与观察者

图片加载是异步的，需要通知观察者：

```typescript
protected addImageObserver(promise: Promise<unknown>) {
  this.draw.getImageObserver().add(promise)
}
```

---

## 6. TableParticle：复杂表格的绘制

`TableParticle`（`src/editor/core/draw/particle/table/TableParticle.ts`）是最复杂的粒子，需要处理：

- 单元格合并（rowspan/colspan）
- 边框样式（外边框、内边框、虚线等）
- 单元格背景色
- 单元格斜线
- 垂直对齐

### 6.1 行列信息计算

```typescript
// src/editor/core/draw/particle/table/TableParticle.ts, lines 391-492
public computeRowColInfo(element: IElement) {
  const { colgroup, trList } = element

  for (let t = 0; t < trList.length; t++) {
    const tr = trList[t]
    for (let d = 0; d < tr.tdList.length; d++) {
      const td = tr.tdList[d]

      // 计算单元格宽度（colspan 合并）
      let width = 0
      for (let col = 0; col < td.colspan; col++) {
        width += colgroup[col + colIndex].width
      }

      // 计算单元格高度（rowspan 合并）
      let height = 0
      for (let row = 0; row < td.rowspan; row++) {
        height += trList[row + t].height
      }

      // 计算坐标
      td.x = preX
      td.y = preY
      td.width = width
      td.height = height
    }
  }
}
```

### 6.2 边框绘制

```typescript
// src/editor/core/draw/particle/table/TableParticle.ts, lines 104-137
private _drawOuterBorder(payload) {
  ctx.save()
  ctx.lineWidth = borderWidth * scale
  ctx.strokeStyle = borderColor
  if (borderType === TableBorder.DASH) {
    ctx.setLineDash([3, 3])
  }
  ctx.beginPath()
  if (isDrawFullBorder) {
    ctx.rect(x, y, width, height)  // 完整矩形
  } else {
    ctx.moveTo(x, y + height)
    ctx.lineTo(x, y)
    ctx.lineTo(x + width, y)     // L 形（左、上边）
  }
  ctx.stroke()
  ctx.restore()
}
```

### 6.3 单元格边框

```typescript
// src/editor/core/draw/particle/table/TableParticle.ts, lines 234-254
// 每个单元格可以单独控制四条边的显示
if (td.borderTypes?.includes(TdBorder.TOP)) {
  ctx.moveTo(x - width, y)
  ctx.lineTo(x, y)
  ctx.stroke()
}
if (td.borderTypes?.includes(TdBorder.RIGHT)) {
  ctx.moveTo(x, y)
  ctx.lineTo(x, y + height)
  ctx.stroke()
}
// ...
```

### 6.4 选区高亮

```typescript
// src/editor/core/draw/particle/table/TableParticle.ts, lines 494+
public drawRange(ctx, element, startX, startY) {
  // 获取选中的单元格
  const rowCol = this.getRangeRowCol()
  if (!rowCol) return

  ctx.save()
  ctx.fillStyle = this.options.rangeColor
  ctx.globalAlpha = this.options.rangeAlpha

  for (const row of rowCol) {
    for (const td of row) {
      const x = td.x * scale + startX
      const y = td.y * scale + startY
      ctx.fillRect(x, y, td.width * scale, td.height * scale)
    }
  }
  ctx.restore()
}
```

---

## 7. LaTeXParticle：数学公式渲染

`LaTeXParticle`（`src/editor/core/draw/particle/latex/LaTexParticle.ts`）继承自 `ImageParticle`，因为 LaTeX 公式最终也是作为图片绘制。

### 7.1 渲染流程

```typescript
// src/editor/core/draw/particle/latex/LaTexParticle.ts
public render(ctx, element, x, y) {
  const width = element.width * scale
  const height = element.height * scale

  if (this.imageCache.has(element.value)) {
    // 已有缓存（SVG 已转为图片）
    const img = this.imageCache.get(element.value)!
    ctx.drawImage(img, x, y, width, height)
  } else {
    // 第一次渲染，需要将 SVG 转为图片
    const img = new Image()
    img.src = element.laTexSVG  // SVG 数据内联在元素中
    img.onload = () => {
      ctx.drawImage(img, x, y, width, height)
      this.imageCache.set(element.value, img)
    }
  }
}
```

### 7.2 SVG 生成

公式的 SVG 生成在 `LaTexUtils` 中完成：

```typescript
// src/editor/core/draw/particle/latex/utils/LaTexUtils.ts
public svg(options: LaTexOptions): LaTexSVG {
  // 使用 Hershey 字体渲染数学符号
  // 生成 SVG 字符串
  return {
    svg: '<svg>...</svg>',
    width: measuredWidth,
    height: measuredHeight
  }
}
```

---

## 8. ListParticle：列表标记

`ListParticle`（`src/editor/core/draw/particle/ListParticle.ts`）负责绘制列表的标记（序号、圆点、方块等）。

### 8.1 列表样式

```typescript
enum ListType {
  ORDERED = 'ordered',    // 有序列表（1. 2. 3.）
  UNORDERED = 'unordered' // 无序列表（• • •）
}

enum ListStyle {
  NUMBER = 'number',           // 数字
  LOWER_LETTER = 'lowerLetter', // 小写字母 a. b. c.
  UPPER_LETTER = 'upperLetter', // 大写字母 A. B. C.
  LOWER_ROMAN = 'lowerRoman',   // 小写罗马数字 i. ii. iii.
  UPPER_ROMAN = 'upperRoman',   // 大写罗马数字 I. II. III.
  DISC = 'disc',               // 实心圆 •
  CIRCLE = 'circle',          // 空心圆 ○
  SQUARE = 'square',           // 方块 ■
  CHECKBOX = 'checkbox'        // 复选框 ☑
}
```

### 8.2 列表标记绘制

```typescript
// src/editor/core/draw/particle/ListParticle.ts, lines 191-254
public drawListStyle(ctx, row, position) {
  const { elementList, offsetX, listIndex, ascent } = row
  const startElement = elementList[0]

  // 列表标记只在行首换行符处绘制
  if (startElement.value !== ZERO || startElement.listWrap) return

  const x = startX - offsetX + tabWidth
  const y = startY + ascent

  if (startElement.listType === ListType.UL) {
    // 无序列表：圆点、方块等
    text = ulStyleMapping[startElement.listStyle] || '•'
  } else {
    // 有序列表：1. 2. 3. 或 a. b. c.
    text = `${listIndex + 1}.`
  }

  ctx.save()
  ctx.font = listFontStyle
  ctx.fillText(text, x, y)  // 在行首绘制标记
  ctx.restore()
}
```

### 8.3 列表宽度计算

```typescript
// 计算有序列表的序号最大宽度，用于缩进
public getListStyleWidth(ctx, listElementList): number {
  if (listStyle !== ListStyle.DECIMAL) {
    return fixedWidth  // 非数字样式返回固定宽度
  }
  // 计算最大序号的宽度
  const count = listElementList.filter(e => e.value === ZERO).length
  const maxText = String(count).repeat('9')
  return ctx.measureText(maxText).width + LIST_GAP
}
```

---

## 9. HyperlinkParticle：超链接装饰

`HyperlinkParticle`（`src/editor/core/draw/particle/HyperlinkParticle.ts`）负责超链接的装饰（下划线、颜色、弹窗）。

### 9.1 超链接样式

```typescript
// 超链接元素
{
  type: ElementType.HYPERLINK,
  value: '',  // 超链接容器，valueList 包含实际内容
  valueList: [...],  // 链接文本元素
  url: 'https://example.com',
  hyperlinkId: 'link-001'
}
```

### 9.2 超链接弹窗

```typescript
// 单击时显示弹窗
public drawHyperlinkPopup(element, position) {
  // 在链接上方显示一个小浮层
  // 包含：链接地址、复制按钮、打开按钮
}

// Ctrl+单击时打开链接
public openHyperlink(element) {
  window.open(element.url, '_blank')
}
```

---

## 10. ControlParticle：表单控件

表单控件（复选框、单选框、日期选择器等）是特殊的粒子，它们使用 **DOM 而非 Canvas** 绘制在覆盖层上。

### 10.1 控件覆盖层

```
.page-container
  ├── .page-canvas（每页一个）
  ├── .cursor（光标）
  ├── .control-float（控件覆盖层）
  │   ├── <input type="checkbox">
  │   ├── <input type="radio">
  │   └── <input type="date">
  └── ...
```

### 10.2 复选框绘制

```typescript
// src/editor/core/draw/particle/CheckboxParticle.ts
public render(options) {
  const { ctx, element, x, y } = options
  // Canvas 绘制复选框外观
  ctx.save()
  ctx.strokeStyle = borderColor
  ctx.fillStyle = checked ? checkedColor : 'transparent'
  // 绘制方框
  ctx.strokeRect(x, y, size, size)
  // 绘制勾选标记
  if (checked) {
    ctx.beginPath()
    ctx.moveTo(...)
    ctx.stroke()
  }
  ctx.restore()
}
```

### 10.3 日期控件

日期控件使用原生的 `<input type="date">`：

```typescript
// src/editor/core/draw/particle/date/DateParticle.ts
public renderDatePicker(element, position) {
  // 创建原生日期输入框
  const input = document.createElement('input')
  input.type = 'date'
  input.value = element.value

  // 定位到文档坐标
  const { x, y } = this.calculatePosition(position)
  input.style.left = `${x}px`
  input.style.top = `${y}px`

  // 添加到控件覆盖层
  this.floatContainer.appendChild(input)
}
```

---

## 11. 核心要点

1. **粒子接口统一**——所有粒子都实现 `draw(ctx, element, row)` 接口。

2. **TextParticle 批量优化**——将相同样式的连续文本合并绘制，减少 Canvas 状态切换。

3. **ImageParticle 缓存机制**——图片加载后缓存，避免重复加载。

4. **TableParticle 最复杂**——需要处理 rowspan/colspan 合并、多种边框样式、单元格背景、斜线等。

5. **LaTeXParticle 继承 ImageParticle**——公式的 SVG 转图片后，按图片方式绘制。

6. **ListParticle 负责标记**——只在行首换行符处绘制列表标记（序号、圆点等）。

7. **控件使用 DOM 覆盖层**——复选框、单选框、日期选择器等使用原生 DOM 元素而非 Canvas 绘制。

8. **粒子与 ElementType 一一对应**——Draw 通过 `getParticleByElement()` 找到对应粒子。

---

## 下一步

现在你已经理解了 Particle 粒子系统，下一步可以学习 **Plugin 插件系统**（教程 10），了解如何扩展编辑器的功能。

**值得思考的问题**：

- TextParticle 的批量绘制如何处理换行符的情况？
- 当表格单元格有 rowspan/colspan 时，位置如何计算？
- 为什么 LaTeX 公式可以复用 ImageParticle 的缓存机制？
- 表单控件为什么不直接画在 Canvas 上，而要用 DOM 覆盖层？

这些问题将在后续教程中一一解答。
