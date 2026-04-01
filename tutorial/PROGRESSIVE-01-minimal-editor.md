# 第1章：最小可运行编辑器

> 构建一个只能输入文字的最简单的编辑器

---

## 本章目标

```
学完本章后，你将能够：
1. 创建一个 HTML Canvas 并正确初始化
2. 让用户在 Canvas 上输入文字
3. 理解"数据驱动视图"的基本概念
4. 实现文字的回车换行
```

---

## 1.1 创建项目结构

首先，创建一个简单的项目结构：

```
minimal-editor/
├── index.html      # 入口 HTML
├── editor.ts       # 编辑器核心代码
└── main.ts         # 初始化代码
```

### index.html

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>最小编辑器</title>
  <style>
    body {
      margin: 0;
      padding: 20px;
      font-family: sans-serif;
    }
    #editor {
      border: 1px solid #ccc;
      cursor: text;
    }
  </style>
</head>
<body>
  <div id="editor"></div>
  <script type="module" src="./main.ts"></script>
</body>
</html>
```

---

## 1.2 创建 Canvas 编辑器类

这是最基础的一步：创建一个能显示文字的 Canvas。

### editor.ts

```typescript
// 编辑器配置接口
interface IEditorOptions {
  width: number      // Canvas 宽度
  height: number     // Canvas 高度
  fontSize: number   // 字体大小
  fontFamily: string // 字体
  color: string      // 文字颜色
  backgroundColor: string // 背景色
}

// 编辑器类
class MinimalEditor {
  private container: HTMLElement
  private canvas: HTMLCanvasElement
  private ctx: CanvasRenderingContext2D
  private options: IEditorOptions
  
  // 光标位置（用字符索引表示）
  private cursorIndex: number = 0
  
  // 文本数据（每个字符一个元素）
  private textData: string[] = []
  
  constructor(container: HTMLElement, options?: Partial<IEditorOptions>) {
    this.container = container
    // 默认配置
    this.options = {
      width: 800,
      height: 600,
      fontSize: 16,
      fontFamily: 'sans-serif',
      color: '#000000',
      backgroundColor: '#ffffff',
      ...options
    }
    
    // 初始化
    this.initCanvas()
    this.bindEvents()
    this.render()
  }
  
  private initCanvas() {
    // 创建 Canvas 元素
    this.canvas = document.createElement('canvas')
    this.canvas.width = this.options.width
    this.canvas.height = this.options.height
    this.canvas.style.display = 'block'
    
    // 设置样式
    this.canvas.style.backgroundColor = this.options.backgroundColor
    
    // 添加到容器
    this.container.appendChild(this.canvas)
    
    // 获取绘图上下文
    this.ctx = this.canvas.getContext('2d')!
  }
  
  private bindEvents() {
    // 点击时聚焦
    this.canvas.addEventListener('click', () => {
      this.canvas.focus()
    })
    
    // 键盘输入
    this.canvas.addEventListener('keydown', (e) => {
      if (e.key === 'Backspace') {
        this.handleBackspace()
      } else if (e.key === 'Enter') {
        this.handleEnter()
      } else if (e.key.length === 1) {
        this.handleInput(e.key)
      }
    })
    
    // 使 Canvas 可以接收键盘事件
    this.canvas.tabIndex = 1
  }
  
  // ========== 核心方法 ==========
  
  // 处理文字输入
  private handleInput(char: string) {
    // 在光标位置插入字符
    this.textData.splice(this.cursorIndex, 0, char)
    this.cursorIndex++
    this.render()
  }
  
  // 处理退格
  private handleBackspace() {
    if (this.cursorIndex > 0) {
      this.textData.splice(this.cursorIndex - 1, 1)
      this.cursorIndex--
      this.render()
    }
  }
  
  // 处理回车（这里我们简单地换行，不做实际换行处理）
  private handleEnter() {
    this.handleInput('\n')
  }
  
  // 渲染画布
  private render() {
    // 清空画布
    this.ctx.fillStyle = this.options.backgroundColor
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height)
    
    // 设置字体
    this.ctx.font = `${this.options.fontSize}px ${this.options.fontFamily}`
    this.ctx.fillStyle = this.options.color
    this.ctx.textBaseline = 'top'
    
    // 绘制文字（这里先简单地把所有文字连起来）
    const text = this.textData.join('')
    this.ctx.fillText(text, 10, 10)
    
    // 绘制光标
    this.drawCursor()
  }
  
  // 绘制光标
  private drawCursor() {
    // 测量光标之前的文字宽度
    const textBeforeCursor = this.textData.slice(0, this.cursorIndex).join('')
    const cursorX = 10 + this.ctx.measureText(textBeforeCursor).width
    
    // 光标高度等于字体大小
    const cursorHeight = this.options.fontSize
    
    // 绘制光标（一条竖线）
    this.ctx.fillStyle = '#000'
    this.ctx.fillRect(cursorX, 10, 2, cursorHeight)
  }
}
```

### main.ts

```typescript
import { MinimalEditor } from './editor'

// 创建编辑器实例
const container = document.getElementById('editor')!
const editor = new MinimalEditor(container)
```

---

## 1.3 运行看看效果

用任意 HTTP 服务器启动，比如：

```bash
# Python 3
python -m http.server 3000

# 或者 Node.js
npx serve .
```

打开浏览器访问，你应该能看到：
- 一个白色背景的 Canvas
- 点击后可以输入文字
- Backspace 可以删除
- Enter 可以换行

---

## 1.4 问题分析

上面的代码有一个**严重问题**：文字不会自动换行！

当输入的文字超过 Canvas 宽度时，文字会超出边界。这不是一个完整的编辑器。

让我们修复这个问题。

---

## 1.5 添加自动换行

换行的核心思路：
1. 把文本分割成多行
2. 逐行绘制
3. 每行累积宽度，超过最大宽度时换行

### 改进后的 editor.ts

```typescript
interface IEditorOptions {
  width: number
  height: number
  fontSize: number
  fontFamily: string
  color: string
  backgroundColor: string
  padding: number  // 新增：内边距
  lineHeight: number  // 新增：行高倍率
}

class MinimalEditor {
  private container: HTMLElement
  private canvas: HTMLCanvasElement
  private ctx: CanvasRenderingContext2D
  private options: IEditorOptions
  
  private cursorIndex: number = 0
  private textData: string[] = []
  
  // 新增：缓存的行数据
  private lineData: string[][] = []
  
  constructor(container: HTMLElement, options?: Partial<IEditorOptions>) {
    this.container = container
    this.options = {
      width: 800,
      height: 600,
      fontSize: 16,
      fontFamily: 'sans-serif',
      color: '#000000',
      backgroundColor: '#ffffff',
      padding: 10,
      lineHeight: 1.5,
      ...options
    }
    
    this.initCanvas()
    this.bindEvents()
    this.render()
  }
  
  // ... initCanvas, bindEvents 方法保持不变 ...
  
  private bindEvents() {
    this.canvas.addEventListener('click', () => {
      this.canvas.focus()
    })
    
    this.canvas.addEventListener('keydown', (e) => {
      if (e.key === 'Backspace') {
        this.handleBackspace()
      } else if (e.key === 'Enter') {
        this.handleEnter()
      } else if (e.key.length === 1 && !e.ctrlKey && !e.metaKey) {
        this.handleInput(e.key)
      }
    })
    
    this.canvas.tabIndex = 1
  }
  
  // 处理输入
  private handleInput(char: string) {
    this.textData.splice(this.cursorIndex, 0, char)
    this.cursorIndex++
    this.computeLines()  // 新增：重新计算行
    this.render()
  }
  
  // 处理退格
  private handleBackspace() {
    if (this.cursorIndex > 0) {
      this.textData.splice(this.cursorIndex - 1, 1)
      this.cursorIndex--
      this.computeLines()
      this.render()
    }
  }
  
  // 处理回车
  private handleEnter() {
    this.handleInput('\n')
  }
  
  // ========== 核心：计算行数据 ==========
  private computeLines() {
    const { padding, fontSize, lineHeight, width } = this.options
    const maxWidth = width - padding * 2
    const lineHeightPx = fontSize * lineHeight
    
    this.lineData = []
    let currentLine: string[] = []
    let currentWidth = 0
    
    for (const char of this.textData) {
      // 换行符直接创建新行
      if (char === '\n') {
        this.lineData.push(currentLine)
        currentLine = []
        currentWidth = 0
        continue
      }
      
      // 测量字符宽度
      const charWidth = this.ctx.measureText(char).width
      
      // 如果加上这个字符会超过最大宽度，则换行
      if (currentWidth + charWidth > maxWidth && currentLine.length > 0) {
        this.lineData.push(currentLine)
        currentLine = [char]
        currentWidth = charWidth
      } else {
        currentLine.push(char)
        currentWidth += charWidth
      }
    }
    
    // 最后一行
    if (currentLine.length > 0) {
      this.lineData.push(currentLine)
    }
    
    // 处理空文档
    if (this.lineData.length === 0) {
      this.lineData.push([])
    }
  }
  
  // ========== 渲染 ==========
  private render() {
    // 清空
    this.ctx.fillStyle = this.options.backgroundColor
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height)
    
    // 设置字体
    this.ctx.font = `${this.options.fontSize}px ${this.options.fontFamily}`
    this.ctx.fillStyle = this.options.color
    this.ctx.textBaseline = 'top'
    
    const { padding, fontSize, lineHeight } = this.options
    const lineHeightPx = fontSize * lineHeight
    
    // 逐行绘制
    for (let lineIndex = 0; lineIndex < this.lineData.length; lineIndex++) {
      const line = this.lineData[lineIndex]
      const y = padding + lineIndex * lineHeightPx
      const text = line.join('')
      this.ctx.fillText(text, padding, y)
    }
    
    // 绘制光标
    this.drawCursor()
  }
  
  // 绘制光标（改进版）
  private drawCursor() {
    const { padding, fontSize, lineHeight } = this.options
    const lineHeightPx = fontSize * lineHeight
    
    // 计算光标在哪一行、哪一列
    let charCount = 0
    let cursorLineIndex = 0
    let cursorX = padding
    let cursorY = padding
    
    for (let i = 0; i < this.lineData.length; i++) {
      const line = this.lineData[i]
      const lineCharCount = line.length
      
      if (charCount + lineCharCount >= this.cursorIndex) {
        // 光标在这一行
        cursorLineIndex = i
        const colInLine = this.cursorIndex - charCount
        const textBeforeCursor = line.slice(0, colInLine).join('')
        cursorX = padding + this.ctx.measureText(textBeforeCursor).width
        cursorY = padding + i * lineHeightPx
        break
      }
      
      charCount += lineCharCount + 1 // +1 是因为换行符
    }
    
    // 绘制光标
    this.ctx.fillStyle = '#000'
    this.ctx.fillRect(cursorX, cursorY, 2, fontSize)
  }
}
```

---

## 1.6 现在的效果

现在你应该有了一个：
- ✅ 可以输入文字
- ✅ 自动换行
- ✅ Backspace 删除
- ✅ Enter 换行
- ✅ 光标位置正确

的基础文本编辑器了！

---

## 1.7 本章总结

### 你学到了什么

1. **Canvas 基础**：创建 Canvas、获取绘图上下文、绘制文字
2. **数据驱动视图**：数据（textData）变化 → 重新计算 → 重新渲染
3. **排版概念**：如何把一维文字分割成多行

### 核心代码模式

```typescript
// 数据
private textData: string[] = []
private lineData: string[][] = []

// 输入时更新数据
private handleInput(char: string) {
  this.textData.splice(this.cursorIndex, 0, char)
  this.cursorIndex++
  this.computeLines()  // 重新计算
  this.render()        // 重新渲染
}

// 渲染时绘制数据
private render() {
  // 清空
  // 设置样式
  // 绘制文字
  // 绘制光标
}
```

### 待解决的问题

| 问题 | 原因 | 后续章节 |
|------|------|----------|
| 光标无法通过鼠标点击定位 | 还没实现鼠标位置计算 | 第4章 |
| 没有选中文本功能 | 还没实现选区管理 | 第4章 |
| 文字样式单一 | 没有样式系统 | 第5章 |
| 不能撤销重做 | 没有历史记录 | 第6章 |

---

## 1.8 练习题

1. **基础练习**：修改代码，让编辑器支持 Tab 键输入（显示为4个空格）

2. **进阶练习**：添加"限制最大行数"功能，超过后不再接受输入

3. **挑战练习**：尝试实现"按住 Shift + 点击"来选中一段文字（提示：需要追踪 startIndex 和 endIndex）

---

## 下一步

[第2章：元素与数据模型](./PROGRESSIVE-02-element-model.md)

我们将学习如何用"元素"（Element）来描述更复杂的文本结构，为添加样式支持打下基础。
