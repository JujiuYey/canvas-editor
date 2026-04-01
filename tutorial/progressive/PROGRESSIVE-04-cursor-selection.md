# 第4章：光标与选区

> 实现鼠标点击定位和文本选中功能

---

## 本章目标

```
学完本章后，你将能够：
1. 实现通过鼠标点击定位光标
2. 实现拖拽选区
3. 理解 Range（选区）数据结构
4. 实现选区高亮的绘制
```

---

## 4.1 上一章的局限

在前几章中，我们的光标只能通过键盘移动。用户无法：
- 点击鼠标定位光标
- 拖拽选中文本
- Shift + 点击选中一段文字

本章将解决这些问题。

---

## 4.2 核心数据结构

### Range 接口

```typescript
interface IRange {
  startIndex: number  // 选区起始位置
  endIndex: number    // 选区结束位置
}
```

- `startIndex === endIndex`：光标位置（没有选区）
- `startIndex < endIndex`：从左到右的选区
- `startIndex > endIndex`：从右到左的选区

### Position 接口

```typescript
interface IPosition {
  index: number    // 字符索引
  x: number        // 像素 x 坐标
  y: number        // 像素 y 坐标
}
```

---

## 4.3 鼠标点击定位光标

### 核心思路

1. 监听 Canvas 的点击事件
2. 获取点击的 (x, y) 坐标
3. 遍历行和字符，找到最近的字符位置

### 实现

```typescript
class SelectionEditor {
  // ... 其他属性 ...
  
  // 选区状态
  private startIndex: number = 0  // 选区起始
  private endIndex: number = 0    // 选区结束
  
  constructor(container: HTMLElement, options?: Partial<IEditorOptions>) {
    // ... 初始化代码 ...
    
    this.bindMouseEvents()
  }
  
  // ========== 鼠标事件 ==========
  
  private bindMouseEvents() {
    // 点击事件
    this.canvas.addEventListener('mousedown', (e) => {
      this.handleMouseDown(e)
    })
    
    // 拖拽事件
    this.canvas.addEventListener('mousemove', (e) => {
      this.handleMouseMove(e)
    })
    
    // 释放事件
    document.addEventListener('mouseup', () => {
      this.handleMouseUp()
    })
  }
  
  private handleMouseDown(e: MouseEvent) {
    // 获取点击坐标
    const rect = this.canvas.getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top
    
    // 将坐标转换为索引
    const index = this.getIndexByPosition(x, y)
    
    // 如果没有选区，开始新的选区
    this.startIndex = index
    this.endIndex = index
    
    // 聚焦
    this.canvas.focus()
    
    // 重渲染
    this.render()
  }
  
  private handleMouseMove(e: MouseEvent) {
    // 只有在拖拽时才处理
    if (!this.isDragging) return
    
    const rect = this.canvas.getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top
    
    const index = this.getIndexByPosition(x, y)
    this.endIndex = index
    
    this.render()
  }
  
  private handleMouseUp() {
    this.isDragging = false
  }
  
  // ========== 坐标转索引 ==========
  
  private getIndexByPosition(x: number, y: number): number {
    const { padding, defaultFontSize, lineHeight } = this.options
    const lineHeightPx = defaultFontSize * lineHeight
    
    // 找到最近的行
    let targetRow: IRow | null = null
    let rowIndex = 0
    
    for (let i = 0; i < this.rowList.length; i++) {
      const row = this.rowList[i]
      const rowTop = row.y
      const rowBottom = row.y + row.height
      
      if (y >= rowTop && y <= rowBottom) {
        targetRow = row
        rowIndex = i
        break
      }
      
      // 如果 y 小于当前行顶部，说明在上方
      if (y < rowTop) {
        rowIndex = i
        break
      }
    }
    
    if (!targetRow) {
      // 点击在最后一行之后
      return this.elementList.length
    }
    
    // 计算该行内最近的字符
    let charIndex = 0
    let currentX = targetRow.x
    let minDistance = Infinity
    
    // 统计到这一行之前的字符总数
    let charCountBeforeRow = 0
    for (let i = 0; i < rowIndex; i++) {
      charCountBeforeRow += this.rowList[i].elements.length
    }
    
    for (let i = 0; i < targetRow.elements.length; i++) {
      const element = targetRow.elements[i]
      const elementWidth = this.getElementWidth(element)
      const charX = currentX + elementWidth / 2
      
      const distance = Math.abs(x - charX)
      if (distance < minDistance) {
        minDistance = distance
        charIndex = i
      }
      
      currentX += elementWidth
    }
    
    // 如果点击位置在最后一个字符之后
    if (x > currentX) {
      charIndex = targetRow.elements.length
    }
    
    return charCountBeforeRow + charIndex
  }
  
  // 获取元素宽度
  private getElementWidth(element: IElement): number {
    const fontSize = element.fontSize || this.options.defaultFontSize
    const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
    this.ctx.font = `${fontSize}px ${fontFamily}`
    return this.ctx.measureText(element.value).width
  }
}
```

---

## 4.4 绘制选区高亮

### 核心思路

选区高亮需要：
1. 找出选区覆盖的区域（可能跨多行）
2. 绘制蓝色半透明背景

### 实现

```typescript
private render() {
  // 清空画布
  this.ctx.fillStyle = this.options.backgroundColor
  this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height)
  
  // 绘制文本
  this.renderText()
  
  // 绘制选区高亮
  this.renderSelection()
  
  // 绘制光标（仅当没有选区时）
  if (this.startIndex === this.endIndex) {
    this.drawCursor()
  }
}

// 绘制选区
private renderSelection() {
  if (this.startIndex === this.endIndex) return
  
  const { padding, defaultFontSize, lineHeight } = this.options
  const lineHeightPx = defaultFontSize * lineHeight
  
  // 确保 startIndex <= endIndex
  const [start, end] = this.startIndex < this.endIndex
    ? [this.startIndex, this.endIndex]
    : [this.endIndex, this.startIndex]
  
  // 设置高亮样式
  this.ctx.fillStyle = 'rgba(0, 120, 215, 0.3)' // 蓝色半透明
  
  // 遍历所有行
  let globalIndex = 0
  
  for (const row of this.rowList) {
    const rowStartIndex = globalIndex
    const rowEndIndex = globalIndex + row.elements.length - 1
    
    // 检查这一行是否与选区有交集
    if (rowEndIndex < start || rowStartIndex > end) {
      globalIndex += row.elements.length
      continue
    }
    
    // 计算选区在这一行的范围
    const selectionStart = Math.max(0, start - rowStartIndex)
    const selectionEnd = Math.min(row.elements.length - 1, end - rowStartIndex)
    
    if (selectionStart > selectionEnd) {
      globalIndex += row.elements.length
      continue
    }
    
    // 计算 x 坐标
    let x = row.x
    for (let i = 0; i < selectionStart; i++) {
      x += this.getElementWidth(row.elements[i])
    }
    
    // 计算选区宽度
    let width = 0
    for (let i = selectionStart; i <= selectionEnd; i++) {
      width += this.getElementWidth(row.elements[i])
    }
    
    // 绘制选区矩形
    this.ctx.fillRect(x, row.y, width, row.height)
    
    globalIndex += row.elements.length
  }
}
```

---

## 4.5 处理选区内的删除

### 核心思路

当选区存在时（startIndex !== endIndex）：
- 按 Delete 或 Backspace 键应该删除选中的所有文本
- 输入新文字应该替换选中的文字

### 实现

```typescript
private handleBackspace() {
  if (this.startIndex === this.endIndex) {
    // 没有选区，正常删除
    if (this.cursorIndex > 0) {
      this.elementList.splice(this.cursorIndex - 1, 1)
      this.cursorIndex--
    }
  } else {
    // 有选区，删除选中的内容
    this.deleteSelection()
  }
  
  this.computeRows()
  this.render()
}

private handleDelete() {
  if (this.startIndex === this.endIndex) {
    // 没有选区，正常删除后面的字符
    if (this.cursorIndex < this.elementList.length) {
      this.elementList.splice(this.cursorIndex, 1)
    }
  } else {
    // 有选区，删除选中的内容
    this.deleteSelection()
  }
  
  this.computeRows()
  this.render()
}

private handleInput(char: string) {
  // 创建新元素
  const newElement: IElement = {
    value: char,
    ...this.defaultStyle
  }
  
  if (this.startIndex === this.endIndex) {
    // 没有选区，在光标位置插入
    this.elementList.splice(this.startIndex, 0, newElement)
    this.startIndex++
    this.endIndex = this.startIndex
  } else {
    // 有选区，替换选中的内容
    this.deleteSelection()
    this.elementList.splice(this.startIndex, 0, newElement)
    this.endIndex = this.startIndex + 1
  }
  
  this.computeRows()
  this.render()
}

private deleteSelection() {
  const [start, end] = this.startIndex < this.endIndex
    ? [this.startIndex, this.endIndex]
    : [this.endIndex, this.startIndex]
  
  this.elementList.splice(start, end - start)
  this.startIndex = start
  this.endIndex = start
}
```

---

## 4.6 Shift + 点击选区

### 实现

```typescript
private handleMouseDown(e: MouseEvent) {
  const rect = this.canvas.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top
  
  const index = this.getIndexByPosition(x, y)
  
  // 如果按住了 Shift，以当前位置为起点
  if (e.shiftKey && this.canvas === document.activeElement) {
    // 保持 startIndex 不变，更新 endIndex
    this.endIndex = index
  } else {
    // 普通点击，重置选区
    this.startIndex = index
    this.endIndex = index
    this.selectionStartIndex = index // 记录选区的起点
  }
  
  this.isDragging = e.shiftKey || true
  this.canvas.focus()
  this.render()
}
```

---

## 4.7 双击选中单词

```typescript
private handleMouseDown(e: MouseEvent) {
  // 双击检测
  const now = Date.now()
  if (now - this.lastClickTime < 300) {
    // 双击，选中整个单词
    const rect = this.canvas.getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top
    
    const index = this.getIndexByPosition(x, y)
    const [wordStart, wordEnd] = this.selectWord(index)
    
    this.startIndex = wordStart
    this.endIndex = wordEnd
    this.selectionStartIndex = wordStart
  } else {
    // 单击
    // ... 正常逻辑 ...
  }
  
  this.lastClickTime = now
}

// 选中单词
private selectWord(index: number): [number, number] {
  let start = index
  let end = index
  
  // 向前找
  while (start > 0 && !this.isWhitespace(this.elementList[start - 1].value)) {
    start--
  }
  
  // 向后找
  while (end < this.elementList.length && !this.isWhitespace(this.elementList[end].value)) {
    end++
  }
  
  return [start, end]
}

private isWhitespace(char: string): boolean {
  return /\s/.test(char)
}
```

---

## 4.8 完整的事件绑定

```typescript
private bindMouseEvents() {
  let isDragging = false
  let lastClickTime = 0
  
  this.canvas.addEventListener('mousedown', (e) => {
    // ... 上述逻辑 ...
    isDragging = true
  })
  
  this.canvas.addEventListener('mousemove', (e) => {
    if (!isDragging) return
    
    const rect = this.canvas.getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top
    
    this.endIndex = this.getIndexByPosition(x, y)
    this.render()
  })
  
  document.addEventListener('mouseup', () => {
    isDragging = false
  })
  
  // 防止在拖拽时触发文本选择
  this.canvas.addEventListener('selectstart', (e) => {
    e.preventDefault()
  })
}
```

---

## 4.9 本章总结

### 你学到了什么

1. **坐标转索引**：通过点击坐标找到最近的字符位置
2. **选区数据结构**：Range 接口表示选区
3. **选区高亮**：绘制半透明蓝色背景
4. **交互组合**：Shift + 点击、双击选词等

### 核心概念

| 概念 | 作用 |
|------|------|
| `getIndexByPosition` | 坐标 → 索引 |
| `startIndex/endIndex` | 选区起止 |
| `renderSelection` | 绘制高亮 |
| `deleteSelection` | 删除选中文本 |

---

## 4.10 练习题

1. **基础练习**：实现"Ctrl + A"全选功能

2. **进阶练习**：实现"Ctrl + Shift + Left/Right"按单词选择

3. **挑战练习**：实现"三击选中整段"功能

---

## 下一步

[第5章：样式系统](./PROGRESSIVE-05-style-system.md)

我们将学习：
- 加粗、斜体、下划线
- 字体大小和颜色
- 样式切换的快捷键
