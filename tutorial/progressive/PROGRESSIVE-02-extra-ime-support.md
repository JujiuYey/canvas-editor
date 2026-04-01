# 章节附加：支持中文输入（IME 支持）

> ⚠️ 如果你发现输入英文字母正常，但输入中文时会直接显示在 Canvas 上然后消失，或者根本无法输入中文，那么这一节就是为你准备的！

---

## 问题分析

在上一节的代码中，我们只监听了 `keydown` 事件：

```typescript
// 只监听了 keydown 事件
this.canvas.addEventListener('keydown', this.handleKeydown)
```

但是，**中文、日文、韩文等需要使用输入法（IME - Input Method Editor）来输入**。IME 的工作流程是：

```
1. 用户按下键盘 → keydown 事件触发
2. 用户选择候选词 → 继续触发 keydown
3. 用户确认输入 → 触发 input 事件 ← 这里才是最终的中文文本！
```

**关键点**：`keydown` 事件只能获取到单个按键（如 `a`、`b`），但无法获取到 IME 组合后的中文（如 `你好`）。

---

## 解决方案：监听 input 事件

`input` 事件会在输入框内容发生变化时触发，包括：
- 普通字符输入
- IME 输入完成
- 粘贴内容
- 删除内容

---

## 修改代码

在构造函数中，我们需要：

1. 移除 `keydown` 中对普通字符的处理
2. 添加 `input` 事件监听

```typescript
constructor(container: HTMLElement, options?: Partial<IEditorOptions>) {
  // ... 省略前面的代码 ...

  // 6. 绑定事件处理
  this.handleKeydown = this.handleKeydown.bind(this)
  this.handleInputEvent = this.handleInputEvent.bind(this)
  
  // 使用 keydown 处理控制键（退格、换行等）
  this.canvas.addEventListener('keydown', this.handleKeydown)
  
  // 使用 input 事件处理普通字符和 IME 输入
  this.canvas.addEventListener('input', this.handleInputEvent)

  // 7. 让 Canvas 可以接收键盘输入
  this.canvas.tabIndex = 1
  this.canvas.focus()

  // 8. 初始渲染
  this.computeRows()
  this.render()
}
```

---

## 处理 input 事件

我们需要通过 `inputType` 来判断输入类型：

```typescript
// 处理 input 事件（处理普通字符和 IME 输入）
private handleInputEvent(event: InputEvent) {
  // 获取 input 的 data（输入的文本）
  const inputData = event.data
  
  // inputType 包含：
  // - 'insertText': 插入文本
  // - 'insertParagraph': 插入换行
  // - 'deleteContentBackward': 删除内容
  // - 等等
  
  // 对于 IME 输入，event.data 会包含组合后的完整文本
  if (inputData === null) {
    // 某些情况下 data 为 null，跳过处理
    return
  }
  
  // 处理换行
  if (inputData === '\n' || event.inputType === 'insertParagraph') {
    this.handleInput('\n')
    return
  }
  
  // 处理普通文本输入（包括中文）
  if (inputData.length > 0) {
    event.preventDefault()
    this.handleInput(inputData)
  }
}

// 修改 handleKeydown，只处理控制键
private handleKeydown(event: KeyboardEvent) {
  const key = event.key

  // 处理退格
  if (key === 'Backspace') {
    event.preventDefault()
    this.handleBackspace()
    return
  }

  // 处理换行（有些浏览器在 keydown 阶段触发）
  if (key === 'Enter' && !event.shiftKey) {
    // 注意：Enter 由 input 事件处理，这里只处理 Shift+Enter 的情况
    return
  }

  // 其他控制键的处理可以保留
  // 比如方向键移动光标
  if (key === 'ArrowLeft') {
    event.preventDefault()
    this.moveCursor(-1)
    return
  }
  
  if (key === 'ArrowRight') {
    event.preventDefault()
    this.moveCursor(1)
    return
  }
}

// 添加光标移动方法
private moveCursor(delta: number) {
  const newIndex = this.cursorIndex + delta
  if (newIndex >= 0 && newIndex <= this.elementList.length) {
    this.cursorIndex = newIndex
    this.render()
  }
}
```

---

## 完整的修改后的 handleInput 方法

注意：现在 `handleInput` 接收的可能是多字符的字符串（中文），我们需要逐字符插入还是整段插入？

**两种策略**：

### 策略1：保持每个字符一个 Element（推荐）

```typescript
private handleInput(text: string) {
  // 遍历输入的每个字符
  for (const char of text) {
    const newElement: IElement = {
      value: char,
      ...this.defaultStyle
    }
    this.elementList.splice(this.cursorIndex, 0, newElement)
    this.cursorIndex++
  }

  // 重新排版和渲染
  this.computeRows()
  this.render()
}
```

### 策略2：整段文本作为一个 Element

```typescript
private handleInput(text: string) {
  const newElement: IElement = {
    value: text,  // 可能是多个字符
    ...this.defaultStyle
  }
  this.elementList.splice(this.cursorIndex, 0, newElement)
  this.cursorIndex++

  this.computeRows()
  this.render()
}
```

两种策略的区别在于元素粒度。策略1更灵活但元素更多，策略2更高效但修改单个字符需要拆分元素。

---

## 验证修复

现在你可以：

1. 切换到中文输入法
2. 输入拼音
3. 选择候选词
4. 按空格或回车确认

中文应该能够正确显示了！

---

## 进阶：检测 IME 状态

有时候你可能需要知道用户当前是否正在使用 IME：

```typescript
// 添加 IME 状态跟踪
private isComposing: boolean = false

// 在 keydown 中检测 compositionstart 和 compositionend
this.canvas.addEventListener('compositionstart', () => {
  this.isComposing = true
})

this.canvas.addEventListener('compositionend', () => {
  this.isComposing = false
  // compositionend 后会触发 input 事件
})
```

---

## 小结

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 输入英文字母正常 | keydown 能获取单字符 | - |
| 输入中文直接消失 | keydown 无法获取 IME 组合文本 | 监听 `input` 事件 |
| 退格键异常 | input 事件不触发退格 | 保持 keydown 处理退格 |

---

## 下一步

继续阅读 [第2章：元素与数据模型](./PROGRESSIVE-02-element-model.md) 的其他内容，或者进入 [第3章：排版系统](./PROGRESSIVE-03-typesetting.md)。
