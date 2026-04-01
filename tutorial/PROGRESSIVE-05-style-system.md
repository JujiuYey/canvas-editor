# 第5章：样式系统

> 实现加粗、斜体、下划线、颜色等富文本样式

---

## 本章目标

```
学完本章后，你将能够：
1. 实现加粗、斜体、下划线样式
2. 实现字体大小和颜色切换
3. 理解样式应用的两种模式
4. 实现样式与工具栏的联动
```

---

## 5.1 样式分类

在富文本编辑器中，样式可以分为两类：

| 类型 | 示例 | 特点 |
|------|------|------|
| **内联样式** | 加粗、斜体、颜色 | 附着在单个 Element 上 |
| **块级样式** | 标题级别、段落对齐 | 应用于整个段落或块 |

本章重点讨论**内联样式**。

---

## 5.2 Element 样式接口扩展

```typescript
interface IElement {
  // 基础
  value: string
  
  // 内联样式
  bold?: boolean         // 加粗
  italic?: boolean       // 斜体
  underline?: boolean    // 下划线
  strikethrough?: boolean // 删除线
  color?: string        // 文字颜色
  highlight?: string    // 背景高亮
  fontSize?: number    // 字号
  fontFamily?: string  // 字体
  
  // 块级样式
  level?: number        // 标题级别 (1-6)
  rowFlex?: 'left' | 'center' | 'right' | 'justify' // 对齐
}
```

---

## 5.3 样式切换的核心逻辑

### 两种模式

1. **覆盖模式**（Toggle）：有则取消，无则添加
2. **应用模式**：无则添加，有则保持

### 实现

```typescript
class StyleEditor {
  // 当前默认样式
  private defaultStyle: IStyle = {
    bold: false,
    italic: false,
    underline: false,
    strikethrough: false,
    color: '#000000',
    fontSize: 16,
    fontFamily: 'sans-serif'
  }
  
  // ========== 样式切换 ==========
  
  // 切换加粗
  public toggleBold() {
    this.defaultStyle.bold = !this.defaultStyle.bold
    // 如果有选区，批量修改
    if (this.startIndex !== this.endIndex) {
      this.applyStyleToSelection('bold', this.defaultStyle.bold)
    }
    this.render()
  }
  
  // 切换斜体
  public toggleItalic() {
    this.defaultStyle.italic = !this.defaultStyle.italic
    if (this.startIndex !== this.endIndex) {
      this.applyStyleToSelection('italic', this.defaultStyle.italic)
    }
    this.render()
  }
  
  // 切换下划线
  public toggleUnderline() {
    this.defaultStyle.underline = !this.defaultStyle.underline
    if (this.startIndex !== this.endIndex) {
      this.applyStyleToSelection('underline', this.defaultStyle.underline)
    }
    this.render()
  }
  
  // 设置颜色
  public setColor(color: string) {
    this.defaultStyle.color = color
    if (this.startIndex !== this.endIndex) {
      this.applyStyleToSelection('color', color)
    }
    this.render()
  }
  
  // 设置字号
  public setFontSize(size: number) {
    this.defaultStyle.fontSize = size
    if (this.startIndex !== this.endIndex) {
      this.applyStyleToSelection('fontSize', size)
    }
    this.render()
  }
  
  // ========== 批量应用样式 ==========
  
  private applyStyleToSelection(styleName: string, value: any) {
    const [start, end] = this.normalizeRange()
    
    for (let i = start; i < end; i++) {
      this.elementList[i][styleName as keyof IElement] = value
    }
  }
  
  // 标准化选区（确保 start <= end）
  private normalizeRange(): [number, number] {
    return this.startIndex < this.endIndex
      ? [this.startIndex, this.endIndex]
      : [this.endIndex, this.startIndex]
  }
}
```

---

## 5.4 下划线和删除线绘制

### 核心思路

下划线和删除线不是文本样式，而是**装饰性元素**，需要在渲染时单独绘制。

### 实现

```typescript
private render() {
  // 清空画布
  this.ctx.fillStyle = this.options.backgroundColor
  this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height)
  
  // 逐行绘制
  for (const row of this.rowList) {
    this.renderRowDecorations(row) // 绘制下划线、删除线
    this.renderRowText(row)        // 绘制文字
    this.renderRowDecorationsAfter(row) // 在文字之后绘制（如果有）
  }
  
  // 绘制选区
  this.renderSelection()
  
  // 绘制光标
  if (this.startIndex === this.endIndex) {
    this.drawCursor()
  }
}

// 绘制行的装饰（装饰线条）
private renderRowDecorations(row: IRow) {
  const { defaultFontSize } = this.options
  let x = row.x
  let y = row.y
  
  for (const element of row.elements) {
    if (!element.value || element.value === '\n') {
      continue
    }
    
    const fontSize = element.fontSize || defaultFontSize
    const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
    this.ctx.font = `${fontSize}px ${fontFamily}`
    
    const width = this.ctx.measureText(element.value).width
    
    // 下划线
    if (element.underline) {
      this.ctx.strokeStyle = element.color || this.defaultStyle.color!
      this.ctx.lineWidth = 1
      this.ctx.beginPath()
      this.ctx.moveTo(x, y + fontSize + 2) // 文字下方 2px
      this.ctx.lineTo(x + width, y + fontSize + 2)
      this.ctx.stroke()
    }
    
    // 删除线
    if (element.strikethrough) {
      this.ctx.strokeStyle = element.color || this.defaultStyle.color!
      this.ctx.lineWidth = 1
      this.ctx.beginPath()
      this.ctx.moveTo(x, y + fontSize / 2) // 文字中间
      this.ctx.lineTo(x + width, y + fontSize / 2)
      this.ctx.stroke()
    }
    
    x += width
  }
}

// 绘制行的文字
private renderRowText(row: IRow) {
  const { defaultFontSize } = this.options
  
  for (const element of row.elements) {
    if (!element.value || element.value === '\n') {
      continue
    }
    
    const fontSize = element.fontSize || defaultFontSize
    const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
    const color = element.color || this.defaultStyle.color!
    
    // 构建字体字符串
    let fontStyle = ''
    if (element.italic) fontStyle += 'italic '
    if (element.bold) fontStyle += 'bold '
    fontStyle += `${fontSize}px ${fontFamily}`
    
    this.ctx.font = fontStyle
    this.ctx.fillStyle = color
    this.ctx.textBaseline = 'top'
    
    // 绘制文字
    this.ctx.fillText(element.value, row.x, row.y)
    
    // 移动到下一个字符
    row.x += this.ctx.measureText(element.value).width
  }
}
```

---

## 5.5 背景高亮

### 核心思路

背景高亮是在文字下面绘制一个半透明矩形。

### 实现

```typescript
private renderRowText(row: IRow) {
  let x = row.x
  
  for (const element of row.elements) {
    if (!element.value || element.value === '\n') {
      continue
    }
    
    // ... 设置样式 ...
    const width = this.ctx.measureText(element.value).width
    const fontSize = element.fontSize || this.options.defaultFontSize
    
    // 绘制背景高亮
    if (element.highlight) {
      this.ctx.fillStyle = element.highlight
      this.ctx.fillRect(x, row.y, width, fontSize)
    }
    
    // 绘制文字
    this.ctx.fillStyle = color
    this.ctx.fillText(element.value, x, row.y)
    
    x += width
  }
}
```

---

## 5.6 样式连续性

### 问题

当用户在"加粗文字"后输入新文字时，新文字应该自动变成加粗吗？

### 实现

在光标位置插入新元素时，继承前一个元素的样式：

```typescript
private handleInput(char: string) {
  // 获取前一个元素的样式
  const prevElement = this.elementList[this.startIndex - 1]
  
  // 创建新元素，继承样式
  const newElement: IElement = {
    value: char,
    bold: this.defaultStyle.bold,
    italic: this.defaultStyle.italic,
    underline: this.defaultStyle.underline,
    color: this.defaultStyle.color,
    fontSize: this.defaultStyle.fontSize,
    fontFamily: this.defaultStyle.fontFamily,
    
    // 或者直接从 prevElement 继承：
    // ...prevElement
  }
  
  // 插入
  this.elementList.splice(this.startIndex, 0, newElement)
  this.startIndex++
  this.endIndex = this.startIndex
  
  this.computeRows()
  this.render()
}
```

---

## 5.7 样式状态检测

### 问题

当选中一段文字时，工具栏需要显示正确的样式状态：
- 如果选中的文字有加粗，工具栏加粗按钮应该高亮
- 如果选中的文字没有统一样式，应该显示"混合"状态

### 实现

```typescript
public getSelectionStyle(): IStyle {
  const [start, end] = this.normalizeRange()
  
  if (start === end) {
    // 光标位置，返回默认样式
    return { ...this.defaultStyle }
  }
  
  // 统计选区内的样式
  const firstElement = this.elementList[start]
  const style: IStyle = {
    bold: firstElement.bold,
    italic: firstElement.italic,
    underline: firstElement.underline,
    strikethrough: firstElement.strikethrough,
    color: firstElement.color,
    fontSize: firstElement.fontSize,
    fontFamily: firstElement.fontFamily
  }
  
  // 检查是否所有元素样式一致
  for (let i = start + 1; i < end; i++) {
    const element = this.elementList[i]
    // 如果有任何不一致，设置为 undefined（表示混合）
  }
  
  return style
}

// 在渲染工具栏时调用
public updateToolbarState() {
  const style = this.getSelectionStyle()
  
  // 更新按钮状态
  boldButton.classList.toggle('active', !!style.bold)
  italicButton.classList.toggle('active', !!style.italic)
  underlineButton.classList.toggle('active', !!style.underline)
  
  // 更新颜色选择器
  colorPicker.value = style.color || '#000000'
}
```

---

## 5.8 快捷键绑定

```typescript
private bindKeyboardEvents() {
  this.canvas.addEventListener('keydown', (e) => {
    // Ctrl/Cmd + B：加粗
    if ((e.ctrlKey || e.metaKey) && e.key === 'b') {
      e.preventDefault()
      this.toggleBold()
    }
    
    // Ctrl/Cmd + I：斜体
    if ((e.ctrlKey || e.metaKey) && e.key === 'i') {
      e.preventDefault()
      this.toggleItalic()
    }
    
    // Ctrl/Cmd + U：下划线
    if ((e.ctrlKey || e.metaKey) && e.key === 'u') {
      e.preventDefault()
      this.toggleUnderline()
    }
    
    // ... 其他按键 ...
  })
}
```

---

## 5.9 本章总结

### 你学到了什么

1. **内联样式 vs 块级样式**：样式的作用范围不同
2. **装饰性元素绘制**：下划线、删除线需要单独绘制
3. **样式连续性**：新输入的文字应该继承当前样式
4. **样式检测**：当选中多段不同样式的文字时，显示"混合"状态

### 核心概念

| 概念 | 实现位置 |
|------|----------|
| 样式切换 | `toggleBold()` 等方法 |
| 下划线绘制 | `renderRowDecorations()` |
| 样式继承 | `handleInput()` |
| 状态检测 | `getSelectionStyle()` |

---

## 5.10 练习题

1. **基础练习**：实现"字体选择"功能，让用户选择不同字体

2. **进阶练习**：实现"格式刷"功能，复制一个位置的样式应用到另一个位置

3. **挑战练习**：实现"清除格式"功能，恢复默认样式

---

## 下一步

[第6章：撤销与重做](./PROGRESSIVE-06-undo-redo.md)

我们将学习：
- 历史记录栈
- 撤销和重做操作
- 防抖和节流
