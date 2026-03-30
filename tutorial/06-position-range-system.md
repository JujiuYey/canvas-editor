# 教程 06：Position 与 Range 系统 —— 光标与选区的位置追踪

> **前置知识**：建议先阅读教程 05（Element 系统），理解 IElement 数据结构。

---

## 目录

1. [概述：位置系统的核心职责](#1-概述位置系统的核心职责)
2. [Position 类：元素位置计算](#2-position-类元素位置计算)
3. [IElementPosition 接口：位置的完整信息](#3-ielementposition-接口位置的完整信息)
4. [坐标系统：coordinate 四角坐标](#4-坐标系统coordinate-四角坐标)
5. [RangeManager 类：选区管理](#5-rangemanager-类选区管理)
6. [IRange 接口：选区的数据模型](#6-irange-接口选区的数据模型)
7. [选区操作：setRange 与 getSelection](#7-选区操作setrange-与-getselection)
8. [光标系统：Cursor 类](#8-光标系统cursor-类)
9. [坐标映射：从鼠标到索引](#9-坐标映射从鼠标到索引)
10. [表格内位置：跨行跨列的特殊处理](#10-表格内位置跨行跨列的特殊处理)
11. [核心要点](#11-核心要点)

---

## 1. 概述：位置系统的核心职责

Position 与 Range 系统是编辑器中最核心的子系统之一，负责：

- **坐标映射**：将鼠标的 (x, y) 坐标转换为元素索引
- **位置计算**：为每个元素计算精确的绘制坐标
- **选区管理**：追踪用户的开始/结束位置（startIndex/endIndex）
- **光标渲染**：在正确的位置绘制闪烁的光标

```
用户点击画布
    ↓
MouseObserver 捕获坐标 (x, y)
    ↓
Position.getPositionByXY(x, y)  ← 坐标 → 索引
    ↓
RangeManager.setRange(startIndex, endIndex)
    ↓
Cursor.drawCursor()  ← 在索引位置绘制光标
```

---

## 2. Position 类：元素位置计算

`Position`（`src/editor/core/position/Position.ts`）负责所有与位置相关的计算。

### 2.1 核心属性

```typescript
// src/editor/core/position/Position.ts, lines 28-50
export class Position {
  private cursorPosition: IElementPosition | null    // 当前光标位置
  private positionContext: IPositionContext         // 位置上下文（表格、控件状态）
  private positionList: IElementPosition[]           // 所有元素的位置列表
  private floatPositionList: IFloatPosition[]        // 浮动元素位置列表

  private draw: Draw
  private eventBus: EventBus<EventBusMap>
  private options: DeepRequired<IEditorOption>
}
```

### 2.2 位置列表的类型

Position 维护三种位置列表：

| 属性 | 类型 | 说明 |
|-----|------|------|
| `positionList` | `IElementPosition[]` | 主文档所有元素的位置 |
| `floatPositionList` | `IFloatPosition[]` | 浮动元素（环绕图片）的位置 |
| `cursorPosition` | `IElementPosition \| null` | 当前光标的精确位置 |

### 2.3 位置上下文 IPositionContext

```typescript
// src/editor/interface/Position.ts, lines 44-58
export interface IPositionContext {
  isTable: boolean           // 是否在表格内
  isCheckbox?: boolean       // 是否在复选框上
  isRadio?: boolean          // 是否在单选框上
  isControl?: boolean        // 是否在控件内
  isImage?: boolean          // 是否在图片上
  isLabel?: boolean          // 是否在标签上
  isDirectHit?: boolean      // 是否精确命中元素
  index?: number             // 元素索引
  trIndex?: number           // 表格行索引
  tdIndex?: number           // 表格列索引
  tdId?: string              // 单元格 ID
  trId?: string              // 行 ID
  tableId?: string           // 表格 ID
}
```

---

## 3. IElementPosition 接口：位置的完整信息

每个元素的精确位置信息：

```typescript
// src/editor/interface/Element.ts, lines 222-240
export interface IElementPosition {
  pageNo: number              // 页码
  index: number               // 元素索引
  value: string               // 元素值
  rowIndex: number            // 行索引（全局）
  rowNo: number               // 行号（当前页内）
  ascent: number              // 基线上方高度
  lineHeight: number          // 行高
  left: number                // 左偏移
  metrics: IElementMetrics    // 尺寸信息
  isFirstLetter: boolean      // 是否行首字符
  isLastLetter: boolean       // 是否行尾字符
  coordinate: {
    leftTop: number[]         // 左上坐标 [x, y]
    leftBottom: number[]      // 左下坐标 [x, y]
    rightTop: number[]        // 右上坐标 [x, y]
    rightBottom: number[]     // 右下坐标 [x, y]
  }
}
```

### 3.1 metrics 尺寸信息

```typescript
// src/editor/interface/Element.ts, lines 215-220
export interface IElementMetrics {
  width: number                    // 宽度
  height: number                   // 高度
  boundingBoxAscent: number        // 基线上方高度
  boundingBoxDescent: number       // 基线下方高度
}
```

---

## 4. 坐标系统：coordinate 四角坐标

为了精确判断点击位置和绘制选区，每个元素记录四个角的坐标：

```
leftTop [x1, y1]
    ●────────────● rightTop [x2, y1]
    │            │
    │   element  │
    │            │
    ●────────────● rightBottom [x2, y2]
leftBottom [x1, y2]
```

### 4.1 坐标计算逻辑

```typescript
// src/editor/core/position/Position.ts, lines 179-184
const positionItem: IElementPosition = {
  // ...
  coordinate: {
    leftTop: [x, y],
    leftBottom: [x, y + curRow.height],
    rightTop: [x + metrics.width, y],
    rightBottom: [x + metrics.width, y + curRow.height]
  }
}
```

### 4.2 四角坐标的用途

| 用途 | 说明 |
|-----|------|
| **命中检测** | 判断鼠标坐标是否在元素范围内 |
| **选区绘制** | 计算选区的矩形边界 |
| **光标定位** | 根据左右边界判断插入位置 |
| **表格嵌套** | 判断点击是否在表格的特定单元格内 |

---

## 5. RangeManager 类：选区管理

`RangeManager`（`src/editor/core/range/RangeManager.ts`）负责管理选区的开始和结束位置。

### 5.1 核心属性

```typescript
// src/editor/core/range/RangeManager.ts, lines 25-46
export class RangeManager {
  private range: IRange                    // 当前选区
  private defaultStyle: IRangeElementStyle | null  // 选区默认样式

  private draw: Draw
  private position: Position               // 依赖 Position 计算位置
  private historyManager: HistoryManager   // 依赖 HistoryManager
  private listener: Listener
  private eventBus: EventBus<EventBusMap>
}
```

### 5.2 选区的三种状态

```
collapsed = false, selection = false  → 无选区，光标未激活
collapsed = true, selection = false    → 光标位置（无选中文字）
collapsed = false, selection = true     → 有选区（选中了文字）
```

### 5.3 判断方法

```typescript
// 判断是否折叠（光标位置，无选区）
public getIsCollapsed(): boolean {
  const { startIndex, endIndex } = this.range
  return startIndex === endIndex
}

// 判断是否有选区
public getIsSelection(): boolean {
  const { startIndex, endIndex } = this.range
  if (!~startIndex && !~endIndex) return false  // -1 表示未初始化
  return startIndex !== endIndex
}
```

---

## 6. IRange 接口：选区的数据模型

```typescript
// src/editor/interface/Range.ts, lines 4-14
export interface IRange {
  startIndex: number        // 起始索引
  endIndex: number         // 结束索引
  isCrossRowCol?: boolean  // 是否跨行跨列（表格选择）
  tableId?: string         // 表格 ID（表格内选择时）
  startTdIndex?: number    // 起始单元格索引
  endTdIndex?: number      // 结束单元格索引
  startTrIndex?: number    // 起始行索引
  endTrIndex?: number      // 结束行索引
  zone?: EditorZone        // 当前区域（header/main/footer）
}
```

---

## 7. 选区操作：setRange 与 getSelection

### 7.1 设置选区

```typescript
// src/editor/core/range/RangeManager.ts, lines 416-463
public setRange(
  startIndex: number,
  endIndex: number,
  tableId?: string,
  startTdIndex?: number,
  endTdIndex?: number,
  startTrIndex?: number,
  endTrIndex?: number
) {
  // 判断光标是否改变
  const isChange = this.getIsRangeChange(
    startIndex, endIndex, tableId,
    startTdIndex, endTdIndex, startTrIndex, endTrIndex
  )
  if (isChange) {
    this.range.startIndex = startIndex
    this.range.endIndex = endIndex
    // ... 更新其他属性
  }
}
```

### 7.2 获取选区元素

```typescript
// 获取选中的元素列表
public getSelection(): IElement[] | null {
  const { startIndex, endIndex } = this.range
  if (startIndex === endIndex) return null  // 无选区
  const elementList = this.draw.getElementList()
  return elementList.slice(startIndex + 1, endIndex + 1)
}

// 获取纯文本内容
public toString(): string {
  const selection = this.getTextLikeSelection()
  if (!selection) return ''
  return selection
    .map(s => s.value)
    .join('')
    .replace(new RegExp(ZERO, 'g'), '')  // 移除零宽字符
}
```

### 7.3 获取选区样式

`setRangeStyle()` 方法收集选区元素的样式，用于更新工具栏状态：

```typescript
// src/editor/core/range/RangeManager.ts, lines 486-566
public setRangeStyle() {
  // 从选区元素中提取样式
  const font = curElement.font || this.options.defaultFont
  const size = curElement.size || this.options.defaultSize
  const bold = !~curElementList.findIndex(el => !el.bold)
  const italic = !~curElementList.findIndex(el => !el.italic)
  const underline = !~curElementList.findIndex(el => !el.underline)
  const color = curElement.color || null
  // ... 更多样式

  // 通知监听器
  if (rangeStyleChangeListener) {
    rangeStyleChangeListener(rangeStyle)
  }
}
```

---

## 8. 光标系统：Cursor 类

`Cursor`（`src/editor/core/cursor/Cursor.ts`）负责光标的渲染和交互。

### 8.1 光标 DOM 结构

```typescript
// src/editor/core/cursor/Cursor.ts, lines 45-47
this.cursorDom = document.createElement('div')
this.cursorDom.classList.add(`${EDITOR_PREFIX}-cursor`)
this.container.append(this.cursorDom)
```

光标使用 CSS 动画实现闪烁效果：

```typescript
// src/editor/core/cursor/Cursor.ts, lines 76-96
private _blinkStart() {
  this.cursorDom.classList.add(this.ANIMATION_CLASS)  // 添加闪烁动画
}

private _blinkStop() {
  this.cursorDom.classList.remove(this.ANIMATION_CLASS)
}

private _setBlinkTimeout() {
  this._clearBlinkTimeout()
  this.blinkTimeout = window.setTimeout(() => {
    this._blinkStart()
  }, 500)  // 500ms 后开始闪烁
}
```

### 8.2 光标位置计算

```typescript
// src/editor/core/cursor/Cursor.ts, lines 110-193
public drawCursor(payload?: IDrawCursorOption) {
  const cursorPosition = this.position.getCursorPosition()
  const { metrics, coordinate: { leftTop, rightTop }, ascent, pageNo } = cursorPosition

  // 计算光标高度（字体高度 + 1/4 偏移）
  const increaseHeight = Math.min(metrics.height / 4, defaultOffsetHeight)
  const cursorHeight = metrics.height + increaseHeight * 2

  // 计算光标顶部位置
  const descent = metrics.boundingBoxDescent < 0 ? 0 : metrics.boundingBoxDescent
  const cursorTop = leftTop[1] + ascent + descent - (cursorHeight - increaseHeight) + preY

  // 计算光标左侧位置
  const cursorLeft = hitLineStartIndex ? leftTop[0] : rightTop[0]

  // 设置 DOM 位置
  this.cursorDom.style.left = `${cursorLeft}px`
  this.cursorDom.style.top = `${cursorTop}px`
  this.cursorDom.style.height = `${cursorHeight}px`
}
```

### 8.3 光标代理 AgentCursor

为了处理移动端输入和 IME（输入法编辑器），编辑器使用一个隐藏的 `<textarea>` 作为光标代理：

```typescript
// src/editor/core/cursor/CursorAgent.ts
// 这个代理 textarea 捕获移动端键盘输入
```

---

## 9. 坐标映射：从鼠标到索引

这是位置系统最核心的算法：将鼠标的 (x, y) 坐标转换为元素索引。

### 9.1 getPositionByXY 方法

```typescript
// src/editor/core/position/Position.ts, lines 360-698
public getPositionByXY(payload: IGetPositionByXYPayload): ICurrentPosition {
  const { x, y, isTable } = payload
  // ...
}
```

### 9.2 命中检测流程

```
1. 遍历 positionList（所有元素位置）
   │
   ├─ 判断是否在同一页（pageNo 匹配）
   │
   ├─ 判断坐标是否在元素范围内
   │   x >= leftTop[0] - left && x < rightTop[0]
   │   y >= leftTop[1] && y < leftBottom[1]
   │
   └─ 命中则返回该元素的索引
```

### 9.3 边界情况处理

```typescript
// 判断是否在文字中间前后（用于精确定位插入点）
if (elementList[index].value !== ZERO) {
  const valueWidth = rightTop[0] - leftTop[0]
  if (x < leftTop[0] + valueWidth / 2) {
    // 点击左半边，插入点在前一个字符
    curPositionIndex = j - 1
  }
}
```

### 9.4 非命中区域处理

当鼠标点击空白区域时：

```typescript
// 查找最近的行尾元素
const lastLetterList = positionList.filter(
  p => p.isLastLetter && p.pageNo === positionNo
)

// 判断点击 Y 坐标在哪两行之间
for (const lastLetter of lastLetterList) {
  if (y > leftTop[1] && y <= leftBottom[1]) {
    // 在这一行的范围内
    curPositionIndex = lastLetter.index
    break
  }
}
```

---

## 10. 表格内位置：跨行跨列的特殊处理

表格内的位置处理比普通文本复杂得多。

### 10.1 表格位置的表示

当光标在表格内时，`positionContext` 需要记录更多上下文：

```typescript
// 在表格内时返回
{
  isTable: true,
  index: tableElementIndex,      // 表格元素的索引
  tdIndex: columnIndex,          // 列索引
  trIndex: rowIndex,            // 行索引
  tdValueIndex: innerIndex,     // 单元格内元素的索引
  tdId: 'cell-1-1',             // 单元格 ID
  trId: 'row-1',                // 行 ID
  tableId: 'table-1'            // 表格 ID
}
```

### 10.2 表格递归查找

```typescript
// src/editor/core/position/Position.ts, lines 401-444
if (element.type === ElementType.TABLE) {
  for (let t = 0; t < element.trList!.length; t++) {
    const tr = element.trList![t]
    for (let d = 0; d < tr.tdList.length; d++) {
      const td = tr.tdList[d]
      // 递归调用 getPositionByXY 检查单元格
      const tablePosition = this.getPositionByXY({
        x, y,
        td,
        isTable: true,
        elementList: td.value,           // 单元格内的元素列表
        positionList: td.positionList    // 单元格内的位置列表
      })
      if (~tablePosition.index) {
        return {
          // 返回表格内的精确位置
          isTable: true,
          tdIndex: d,
          trIndex: t,
          // ...
        }
      }
    }
  }
}
```

### 10.3 选区跨单元格

当用户在表格中选择多个单元格时：

```typescript
// src/editor/interface/Range.ts, lines 4-14
export interface IRange {
  startIndex: number
  endIndex: number
  isCrossRowCol?: boolean    // 标记跨行跨列
  tableId?: string
  startTdIndex?: number
  endTdIndex?: number
  startTrIndex?: number
  endTrIndex?: number
}
```

---

## 11. 核心要点

1. **Position 维护位置列表**——`positionList` 是核心数据结构，按元素顺序存储每个元素的精确坐标。

2. **四角坐标是命中检测的基础**——通过 `coordinate.leftTop/rightTop/leftBottom/rightBottom` 可以精确判断点击位置。

3. **Range 使用 startIndex/endIndex**——简单高效，但需要配合 Position 的坐标计算才能正确定位。

4. **Cursor 使用 DOM 而非 Canvas**——光标是 HTML div，方便处理输入法和移动端交互。

5. **表格位置需要递归处理**——`getPositionByXY` 对表格会递归检查每个单元格。

6. **位置上下文 IPositionContext**——记录当前是否在表格/控件/图片上，影响后续操作行为。

7. **选区样式汇总**——`setRangeStyle` 遍历选区元素，汇总样式状态用于工具栏更新。

---

## 下一步

现在你已经理解了位置和选区系统，下一步可以学习 **Command 命令系统**（教程 07），了解加粗、删除、格式化等操作是如何实现的。

**值得思考的问题**：

- 当用户点击两个字符之间的位置时，如何判断光标应该在左边还是右边？
- 表格内选中多个单元格时，`startIndex/endIndex` 和 `startTdIndex/endTdIndex` 的关系是什么？
- 为什么不把 Cursor 直接画在 Canvas 上，而是用 DOM 实现？
- `isCrossRowCol` 在什么时候会被设置为 `true`？

这些问题将在后续教程中一一解答。
