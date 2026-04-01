# 第3章：排版系统

> 让文字排版更加专业和美观

---

## 本章目标

```
学完本章后，你将能够：
1. 实现英文单词不断词处理
2. 处理中英文混排场景
3. 处理标点符号的悬挂
4. 理解字体度量（Font Metrics）
```

---

## 3.1 上一章的问题

在第2章的排版中，我们简单地按字符累积宽度。这会导致以下问题：

1. **英文单词被截断**：international → inter-national
2. **标点符号不美观**：句号、逗号在行尾时看起来很丑
3. **中英文间距**：中文和英文之间没有合适的间距

---

## 3.2 英文单词不断词

### 核心思路

当我们发现当前行放不下下一个单词时，应该：
1. 往前查找，找到空格或换行符的位置
2. 把剩余的字符移到下一行

### 实现

```typescript
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
  
  // 遍历元素的索引（需要知道前后元素）
  let i = 0
  while (i < this.elementList.length) {
    const element = this.elementList[i]
    const fontSize = element.fontSize || defaultFontSize
    const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
    
    this.ctx.font = `${fontSize}px ${fontFamily}`
    
    // 换行符
    if (element.value === '\n') {
      this.rowList.push(currentRow)
      currentRow = this.createNewRow(currentRow)
      currentX = padding
      i++
      continue
    }
    
    // 测量元素宽度
    const elementWidth = this.ctx.measureText(element.value).width
    
    // 超出边界
    if (currentX + elementWidth > maxWidth && currentRow.elements.length > 0) {
      // 检查是否是英文单词（包含字母）
      if (this.isEnglishWord(element.value) && !element.value.includes(' ')) {
        // 尝试断词
        const breakResult = this.tryBreakWord(element, maxWidth - currentX)
        if (breakResult) {
          // 添加可以放下的部分
          const breakElement = { ...element, value: breakResult.leftPart }
          currentRow.elements.push(breakElement)
          currentX += this.ctx.measureText(breakResult.leftPart).width
          
          // 把剩余部分拆成新元素
          const remainingElement = { ...element, value: breakResult.rightPart }
          this.elementList.splice(i + 1, 0, remainingElement)
        }
      }
      
      // 开始新行
      this.rowList.push(currentRow)
      currentRow = this.createNewRow(currentRow)
      currentX = padding
      continue // 不 i++，重新处理当前元素
    }
    
    // 添加到当前行
    currentRow.elements.push(element)
    currentX += elementWidth
    i++
  }
  
  // 最后一行
  if (currentRow.elements.length > 0) {
    this.rowList.push(currentRow)
  }
  
  // 空文档
  if (this.rowList.length === 0) {
    this.rowList.push(this.createNewRow(currentRow))
  }
}

// 创建新行
private createNewRow(preRow: IRow): IRow {
  return {
    elements: [],
    x: this.options.padding,
    y: preRow.y + preRow.height,
    width: 0,
    height: preRow.height
  }
}

// 判断是否是英文单词
private isEnglishWord(str: string): boolean {
  return /[a-zA-Z]/.test(str)
}

// 尝试断词
private tryBreakWord(element: IElement, maxWidth: number): { leftPart: string; rightPart: string } | null {
  const fontSize = element.fontSize || this.options.defaultFontSize
  const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
  this.ctx.font = `${fontSize}px ${fontFamily}`
  
  // 从中间开始找合适的断点
  const chars = element.value.split('')
  let leftPart = ''
  let rightPart = ''
  let canBreak = false
  
  for (let i = Math.floor(chars.length / 2); i < chars.length; i++) {
    const char = chars[i]
    const testWidth = this.ctx.measureText(leftPart + char).width
    
    if (testWidth > maxWidth) {
      break
    }
    
    // 优先在元音字母后断词
    if ('aeiouAEIOU'.includes(char)) {
      canBreak = true
    }
    
    if (canBreak || i === chars.length - 1) {
      leftPart += char
      rightPart = chars.slice(i + 1).join('')
      break
    }
    
    leftPart += char
  }
  
  if (!rightPart) return null
  return { leftPart, rightPart: '-' + rightPart }
}
```

---

## 3.3 标点符号悬挂

### 什么是标点悬挂？

在传统排版中，标点符号应该"悬挂"在行尾或行首，不占用正常文字间距：

```
正常：     你好，    世界。
悬挂后：  你好，    世界。
           ↑        ↑
         逗号突出   句号突出
```

### 实现思路

1. 行首的标点不缩进
2. 行尾的标点（包括中文标点）稍微超出边界
3. 行尾是标点时，下一行首不缩进

### 简化实现

```typescript
// 在渲染时处理
private render() {
  // ... 清空画布 ...
  
  const { padding, maxWidth } = this.options
  
  for (let rowIndex = 0; rowIndex < this.rowList.length; rowIndex++) {
    const row = this.rowList[rowIndex]
    let x = row.x
    
    // 计算行首缩进
    const firstElement = row.elements[0]
    const isFirstRowPunctuation = this.isPunctuation(firstElement?.value)
    
    if (rowIndex > 0 && isFirstRowPunctuation) {
      x -= this.ctx.measureText(firstElement.value).width * 0.3
    }
    
    for (const element of row.elements) {
      // ... 设置样式 ...
      this.ctx.fillText(element.value, x, row.y)
      x += this.ctx.measureText(element.value).width
    }
    
    // 行尾标点悬挂
    const lastElement = row.elements[row.elements.length - 1]
    if (this.isPunctuation(lastElement?.value)) {
      // 标点稍微突出
      // （这里简化处理，实际需要更复杂的算法）
    }
  }
}

private isPunctuation(char: string | undefined): boolean {
  if (!char) return false
  return /[，。！？、；：""''【】《》（）]/.test(char)
}
```

---

## 3.4 中英文间距处理

### 问题

中文和英文之间应该有一定的间距，但浏览器默认不会自动添加。

### 解决方案

在渲染时检测并添加间距：

```typescript
private render() {
  // ... 清空画布 ...
  
  for (const row of this.rowList) {
    let x = row.x
    
    for (let i = 0; i < row.elements.length; i++) {
      const element = row.elements[i]
      const nextElement = row.elements[i + 1]
      
      // ... 设置样式 ...
      this.ctx.fillText(element.value, x, row.y)
      
      let charWidth = this.ctx.measureText(element.value).width
      
      // 检测是否需要添加间距
      if (nextElement) {
        const isCurrentChinese = this.isChinese(element.value)
        const isNextChinese = this.isChinese(nextElement.value)
        const isCurrentEnglish = this.isEnglish(element.value)
        const isNextEnglish = this.isEnglish(nextElement.value)
        
        // 中文和英文之间添加间距
        if ((isCurrentChinese && isNextEnglish) || (isCurrentEnglish && isNextChinese)) {
          charWidth += this.options.fontSize * 0.2 // 添加 0.2em 间距
        }
      }
      
      x += charWidth
    }
  }
}

private isChinese(char: string): boolean {
  return /[\u4e00-\u9fa5]/.test(char)
}

private isEnglish(char: string): boolean {
  return /[a-zA-Z]/.test(char)
}
```

---

## 3.5 字体度量（Font Metrics）

### 什么是字体度量？

字体度量描述了字体中字符的尺寸关系：

```
        ↑ baseline（基线）
     ╔════╗
     ║    ║ ↑
     ║ gg ║ | ascent（上行高度）
  ───╨────╨───
       xx    |
       xx    ↓| descent（下行高度）
```

### Canvas 中的度量方法

```typescript
const metrics = ctx.measureText('Hello')

// 返回对象包含：
metrics.width                    // 文本宽度
metrics.actualBoundingBoxLeft   // 左边界偏移
metrics.actualBoundingBoxRight  // 右边界偏移
metrics.actualBoundingBoxAscent // 基线上方高度
metrics.actualBoundingBoxDescent // 基线下方高度
```

### 用于精确排版

```typescript
interface IElementMetrics {
  width: number
  height: number
  ascent: number   // 基线上方
  descent: number  // 基线下方
}

private measureElement(element: IElement): IElementMetrics {
  const fontSize = element.fontSize || this.options.defaultFontSize
  const fontFamily = element.fontFamily || this.defaultStyle.fontFamily!
  
  this.ctx.font = `${fontSize}px ${fontFamily}`
  const metrics = this.ctx.measureText(element.value)
  
  return {
    width: metrics.width,
    height: fontSize, // 简单处理
    ascent: metrics.actualBoundingBoxAscent,
    descent: metrics.actualBoundingBoxDescent
  }
}
```

---

## 3.6 本章总结

### 你学到了什么

1. **单词断词**：在合适位置断词，保持英文单词可读性
2. **标点悬挂**：让标点符号排版更美观
3. **中英文间距**：自动添加合适的间距
4. **字体度量**：理解字符的尺寸关系

### 核心概念

| 概念 | 作用 |
|------|------|
| 断词 | 保持英文单词完整 |
| 标点悬挂 | 排版更美观 |
| 字间距 | 中英文混排更协调 |
| Font Metrics | 精确控制字符位置 |

---

## 3.7 练习题

1. **基础练习**：实现"两端对齐"（Justify），让每行文字都撑满宽度

2. **进阶练习**：实现"首行缩进"功能（两个字符宽度）

3. **挑战练习**：实现"避头避尾"——某些标点不能出现在行首或行尾

---

## 下一步

[第4章：光标与选区](./PROGRESSIVE-04-cursor-selection.md)

我们将学习：
- 如何通过鼠标点击定位光标
- 如何实现文本选区
- 选区高亮的绘制
