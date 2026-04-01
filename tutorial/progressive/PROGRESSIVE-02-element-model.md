# 第2章：元素与数据模型

> 从简单字符串到结构化元素的演进

---

## 本章目标

```
学完本章后，你将能够：
1. 理解为什么需要"元素"（Element）而不是简单字符串
2. 设计一个基础的 Element 数据结构
3. 用元素数组替代字符数组存储文本
4. 理解"数据驱动视图"在复杂场景下的应用
```

---

## 2.1 回顾第1章的数据模型

在第1章中，我们用 `string[]` 存储文本：

```typescript
// 第1章的做法
private textData: string[] = ['你', '好', '世', '界']
```

**问题**：每个元素只是一个字符，无法描述：

- 加粗、斜体、颜色等样式
- 字体大小
- 图片、超链接等特殊元素

---

## 2.2 什么是 Element？

**Element（元素）** 是一个包含"值"和"属性"的对象：

```typescript
// 元素接口
interface IElement {
  value: string // 元素的文本值
  bold?: boolean // 是否加粗
  italic?: boolean // 是否斜体
  color?: string // 文字颜色
  fontSize?: number // 字体大小
  fontFamily?: string // 字体
}

// 示例
const element: IElement = {
  value: '你好',
  bold: true,
  color: '#FF0000'
}
```

### 对比

| 存储方式     | 示例                                     | 能否表示样式 |
| ------------ | ---------------------------------------- | ------------ |
| `string[]`   | `['你', '好']`                           | ❌ 不能      |
| `IElement[]` | `[{value:'你',bold:true}, {value:'好'}]` | ✅ 可以      |

---

## 2.3 为什么要用 Element 数组？

想象一个场景：用户输入"Hello **World**"

### 用字符串数组（❌ 不好）

```
['H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd']
```

问题：你不知道 W 到 d 之间是加粗的

### 用 Element 数组（✅ 好）

```typescript
;[{ value: 'Hello ' }, { value: 'World', bold: true }, { value: '!' }]
```

优势：

- 样式信息内聚在元素中
- 便于遍历、筛选、修改
- 可以表示任意复杂的内容结构

---

## 2.4 重构编辑器：基于 Element

### 新的数据结构

```typescript
// 编辑器选项
interface IEditorOptions {
  width: number
  height: number
  defaultFontSize: number
  defaultFontFamily: string
  defaultColor: string
  backgroundColor: string
  padding: number
  lineHeight: number
}

// 元素接口
interface IElement {
  value: string
  bold?: boolean
  italic?: boolean
  color?: string
  fontSize?: number
  fontFamily?: string
}

// 行数据（包含元素列表和坐标信息）
interface IRow {
  elements: IElement[] // 这一行的所有元素
  x: number // 行起始 x 坐标
  y: number // 行起始 y 坐标
  width: number // 行宽
  height: number // 行高
}
```

### 改进后的编辑器

```typescript
class ElementEditor {
  private container: HTMLElement
  private canvas: HTMLCanvasElement
  private ctx: CanvasRenderingContext2D
  private options: IEditorOptions

  // 核心变化：使用 Element 数组
  private elementList: IElement[] = []
  private cursorIndex: number = 0

  // 排版结果：行数组
  private rowList: IRow[] = []

  // 默认样式（用于新插入的元素）
  private defaultStyle: Pick<
    IElement,
    'fontSize' | 'fontFamily' | 'color' | 'bold' | 'italic'
  > = {
    fontSize: 16,
    fontFamily: 'sans-serif',
    color: '#000000',
    bold: false,
    italic: false
  }

  constructor(container: HTMLElement, options?: Partial<IEditorOptions>) {
    // 1. 保存容器引用
    this.container = container

    // 2. 合并默认选项
    this.options = {
      width: options?.width ?? 600,
      height: options?.height ?? 400,
      defaultFontSize: options?.defaultFontSize ?? 16,
      defaultFontFamily: options?.defaultFontFamily ?? 'sans-serif',
      defaultColor: options?.defaultColor ?? '#000000',
      backgroundColor: options?.backgroundColor ?? '#FFFFFF',
      padding: options?.padding ?? 10,
      lineHeight: options?.lineHeight ?? 1.5
    }

    // 3. 创建 Canvas 元素
    this.canvas = document.createElement('canvas')
    this.canvas.width = this.options.width
    this.canvas.height = this.options.height
    this.canvas.style.display = 'block'
    this.canvas.style.border = '1px solid #ccc'
    this.canvas.style.cursor = 'text'
    container.appendChild(this.canvas)

    // 4. 获取 2D 渲染上下文
    this.ctx = this.canvas.getContext('2d')!

    // 5. 初始化数据结构
    this.elementList = []
    this.rowList = []
    this.cursorIndex = 0

    // 6. 绑定事件处理
    this.handleKeydown = this.handleKeydown.bind(this)
    this.canvas.addEventListener('keydown', this.handleKeydown)

    // 7. 让 Canvas 可以接收键盘输入
    this.canvas.tabIndex = 1
    this.canvas.focus()

    // 8. 初始渲染
    this.computeRows()
    this.render()
  }

  // ========== 核心方法 ==========

  // 处理键盘事件
  private handleKeydown(event: KeyboardEvent) {
    const key = event.key

    // 处理退格
    if (key === 'Backspace') {
      event.preventDefault()
      this.handleBackspace()
      return
    }

    // 处理换行
    if (key === 'Enter') {
      event.preventDefault()
      this.handleInput('\n')
      return
    }

    // 处理普通字符输入
    if (key.length === 1 && !event.ctrlKey && !event.metaKey) {
      event.preventDefault()
      this.handleInput(key)
    }
  }

  // 处理文字输入
  private handleInput(char: string) {
    // 创建一个新的 Element
    const newElement: IElement = {
      value: char,
      ...this.defaultStyle // 继承默认样式
    }

    // 插入到光标位置
    this.elementList.splice(this.cursorIndex, 0, newElement)
    this.cursorIndex++

    // 重新排版和渲染
    this.computeRows()
    this.render()
  }

  // 处理退格
  private handleBackspace() {
    if (this.cursorIndex > 0) {
      this.elementList.splice(this.cursorIndex - 1, 1)
      this.cursorIndex--
      this.computeRows()
      this.render()
    }
  }

  // ========== 排版：计算行数据 ==========
  private computeRows() {
    const { padding, lineHeight, defaultFontSize, width } = this.options
    const maxWidth = width - padding * 2
    const lineHeightPx = defaultFontSize * lineHeight

    this.rowList = []
    let currentRow: IRow = {
      elements: [],
      x: padding,
      y: padding,
      width: 0,
      height: lineHeightPx
    }
    let currentX = padding

    for (const element of this.elementList) {
      // 获取元素的字体信息
      const fontSize = element.fontSize || this.defaultStyle.fontSize!
      const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!

      // 测量元素宽度
      this.ctx.font = `${fontSize}px ${fontFamily}`
      const elementWidth = this.ctx.measureText(element.value).width

      // 换行符或超出边界
      if (
        element.value === '\n' ||
        (currentX + elementWidth > maxWidth && currentRow.elements.length > 0)
      ) {
        // 保存当前行，开始新行
        this.rowList.push(currentRow)
        currentRow = {
          elements: [],
          x: padding,
          y: currentRow.y + currentRow.height,
          width: 0,
          height: lineHeightPx
        }
        currentX = padding

        // 如果是换行符，不添加元素
        if (element.value === '\n') {
          continue
        }
      }

      // 添加到当前行
      currentRow.elements.push(element)
      currentRow.width = currentX + elementWidth - padding
      currentX += elementWidth
    }

    // 最后一行
    if (currentRow.elements.length > 0) {
      this.rowList.push(currentRow)
    }

    // 空文档
    if (this.rowList.length === 0) {
      this.rowList.push({
        elements: [],
        x: padding,
        y: padding,
        width: 0,
        height: lineHeightPx
      })
    }
  }

  // ========== 渲染 ==========
  private render() {
    // 清空画布
    this.ctx.fillStyle = this.options.backgroundColor
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height)

    const { padding, defaultFontSize } = this.options

    // 逐行绘制
    for (const row of this.rowList) {
      let x = row.x

      for (const element of row.elements) {
        // 获取样式
        const fontSize = element.fontSize || defaultFontSize
        const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
        const color = element.color || this.defaultStyle.color!
        const fontStyle = `${element.italic ? 'italic ' : ''}${element.bold ? 'bold ' : ''}${fontSize}px ${fontFamily}`

        // 设置样式
        this.ctx.font = fontStyle
        this.ctx.fillStyle = color
        this.ctx.textBaseline = 'top'

        // 绘制
        this.ctx.fillText(element.value, x, row.y)

        // 更新 x 坐标
        x += this.ctx.measureText(element.value).width
      }
    }

    // 绘制光标
    this.drawCursor()
  }

  // 绘制光标
  private drawCursor() {
    const { padding, defaultFontSize } = this.options
    const lineHeightPx = defaultFontSize * this.options.lineHeight

    // 找到光标所在的行和列
    let elementCount = 0
    let cursorX = padding
    let cursorY = padding

    for (const row of this.rowList) {
      const rowElementCount = row.elements.length

      if (elementCount + rowElementCount >= this.cursorIndex) {
        // 光标在这一行
        const colInRow = this.cursorIndex - elementCount
        cursorY = row.y

        // 计算该行光标前的宽度
        cursorX = padding
        for (let i = 0; i < colInRow; i++) {
          const element = row.elements[i]
          const fontSize = element.fontSize || defaultFontSize
          const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
          this.ctx.font = `${fontSize}px ${fontFamily}`
          cursorX += this.ctx.measureText(element.value).width
        }
        break
      }

      elementCount += rowElementCount
    }

    // 绘制光标
    this.ctx.fillStyle = '#000'
    this.ctx.fillRect(cursorX, cursorY, 2, defaultFontSize)
  }
}
```

---

## 2.5 效果对比

现在编辑器可以：

- ✅ 输入文字
- ✅ 自动换行
- ✅ **支持不同的样式**（需要添加样式切换功能）

### 未来可扩展性

因为使用了 Element 结构，添加样式功能变得很简单：

```typescript
// 添加加粗功能
private toggleBold() {
  if (this.cursorIndex > 0) {
    const element = this.elementList[this.cursorIndex - 1]
    element.bold = !element.bold
    this.computeRows()
    this.render()
  }
}
```

---

## 2.6 Element 的设计原则

### 原则1：最小化

每个 Element 应该只包含它需要的属性：

```typescript
// ✅ 好：只包含实际使用的属性
{ value: 'Hello', bold: true }

// ❌ 不好：冗余属性
{ value: 'Hello', bold: true, italic: false, color: '#000', fontSize: 16, fontFamily: 'sans-serif' }
```

### 原则2：一致性

相同属性的连续元素可以合并，不同属性的需要分开：

```
[你, 好] + [加粗: 世, 界] = [你, 好, 世(bold), 界(bold)]
```

### 原则3：原子性

每个 Element 应该代表最小的不可分割单元：

```
// ✅ 一个 Element 可以包含多个字符（相同样式）
{ value: 'Hello', bold: true }

// ✅ 也可以只有一个字符（需要不同样式时）
{ value: 'H', bold: true }
{ value: 'i', bold: false }
```

---

## 2.7 完整的 Element 接口（参考）

随着编辑器功能增加，Element 接口会越来越丰富：

```typescript
interface IElement {
  // 基础属性
  id?: string // 唯一标识
  value: string // 文本值

  // 样式属性
  bold?: boolean
  italic?: boolean
  underline?: boolean
  strikethrough?: boolean
  color?: string
  highlight?: string // 背景高亮
  fontSize?: number
  fontFamily?: string
  letterSpacing?: number

  // 段落属性
  rowFlex?: 'left' | 'center' | 'right' | 'justify' // 行对齐
  rowMargin?: number // 段落间距

  // 特殊元素类型
  type?: 'image' | 'table' | 'hyperlink' | 'latex' | 'checkbox'

  // 图片属性
  imageUrl?: string
  imageWidth?: number
  imageHeight?: number

  // 分组（用于批注等）
  groupIds?: string[]

  // 隐藏规则
  hide?: boolean
}
```

---

## 2.8 本章总结

### 你学到了什么

1. **Element 的概念**：用对象而不是字符串来描述文本
2. **数据结构演进**：从 `string[]` 到 `IElement[]`
3. **排版结果结构**：从 `string[][]` 到 `IRow[]`
4. **样式继承**：新元素继承默认样式

### 核心代码模式

```typescript
// 数据结构
interface IElement {
  value: string
  bold?: boolean
  color?: string
  // ...
}

interface IRow {
  elements: IElement[]
  x: number
  y: number
  width: number
  height: number
}

// 数据流
用户输入 → 创建 Element → 插入 elementList → computeRows() → render()
```

### 待解决问题

| 问题                       | 后续章节 |
| -------------------------- | -------- |
| 如何切换文字样式（加粗等） | 第5章    |
| 如何支持图片等特殊元素     | 第8章    |
| 如何处理表格               | 第9章    |

---

## 2.9 练习题

1. **基础练习**：修改代码，让用户按 `Ctrl+B` 切换光标前元素的加粗状态

2. **进阶练习**：实现"颜色选择"，让用户可以改变光标前元素的文字颜色

3. **挑战练习**：实现"样式连续性"——当用户在新位置输入文字时，自动继承前一个元素的样式

---

## 下一步

在继续之前，你可能需要先阅读 [章节附加：支持中文输入（IME 支持）](./PROGRESSIVE-02-extra-ime-support.md)，了解如何让编辑器支持中文、日文等需要输入法输入的文字。

然后继续学习 [第3章：排版系统](./PROGRESSIVE-03-typesetting.md)，我们将深入学习排版系统，包括：

- 英文单词不断词
- 中英文混排
- 标点符号处理
- 字体度量
