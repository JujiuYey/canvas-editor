# 第7章：命令系统

> 封装清晰的 API 接口

---

## 本章目标

```
学完本章后，你将能够：
1. 理解命令模式的设计
2. 设计清晰的命令 API
3. 实现命令的撤销/重做集成
4. 理解命令与渲染的解耦
```

---

## 7.1 为什么需要命令系统？

### 直接调用的问题

```typescript
// 直接调用 - 混乱
editor.elementList.splice(0, 1)
editor.computeRows()
editor.render()

// 命令模式 - 清晰
editor.command.delete()  // 或者 executeDelete()
```

### 命令模式的优点

1. **封装**：把操作封装成独立的方法
2. **可撤销**：每个命令都可以撤销
3. **可组合**：复杂操作可以由简单命令组合
4. **可记录**：方便实现宏和脚本

---

## 7.2 命令接口设计

```typescript
interface ICommand {
  execute(): void      // 执行命令
  undo(): void         // 撤销命令
  redo(): void        // 重做命令
}

class CommandManager {
  private commands: ICommand[] = []
  
  public execute(command: ICommand) {
    command.execute()
    this.commands.push(command)
  }
  
  public undo() {
    const command = this.commands.pop()
    if (command) {
      command.undo()
    }
  }
}
```

---

## 7.3 简化版命令系统

对于编辑器来说，命令不需要那么复杂：

```typescript
class CommandEditor {
  // ========== 文本操作 ==========
  
  public executeInsert(text: string) {
    this.saveHistory()
    const element = this.createElement(text)
    this.elementList.splice(this.cursorIndex, 0, element)
    this.cursorIndex++
    this.computeRows()
    this.render()
  }
  
  public executeDelete() {
    if (this.startIndex === this.endIndex) {
      // 没有选区，删除光标前的字符
      if (this.cursorIndex > 0) {
        this.saveHistory()
        this.elementList.splice(this.cursorIndex - 1, 1)
        this.cursorIndex--
      }
    } else {
      // 有选区，删除选区内容
      this.executeDeleteSelection()
    }
    this.computeRows()
    this.render()
  }
  
  public executeDeleteSelection() {
    this.saveHistory()
    const [start, end] = this.normalizeRange()
    this.elementList.splice(start, end - start)
    this.startIndex = start
    this.endIndex = start
    this.computeRows()
    this.render()
  }
  
  // ========== 样式操作 ==========
  
  public executeBold() {
    this.saveHistory()
    if (this.startIndex === this.endIndex) {
      // 光标位置，切换默认样式
      this.defaultStyle.bold = !this.defaultStyle.bold
    } else {
      // 选区，应用到选区
      this.applyStyleToSelection('bold', this.defaultStyle.bold)
    }
    this.computeRows()
    this.render()
  }
  
  public executeColor(color: string) {
    this.saveHistory()
    if (this.startIndex === this.endIndex) {
      this.defaultStyle.color = color
    } else {
      this.applyStyleToSelection('color', color)
    }
    this.computeRows()
    this.render()
  }
  
  // ========== 选区操作 ==========
  
  public executeSelectAll() {
    this.startIndex = 0
    this.endIndex = this.elementList.length
    this.render()
  }
  
  public executeSetCursor(index: number) {
    this.startIndex = index
    this.endIndex = index
    this.render()
  }
}
```

---

## 7.4 命令分组

### 问题

连续操作（如拖动选区）会产生很多小命令，撤销时会逐个撤销，用户体验不好。

### 解决方案

使用**命令分组**：

```typescript
class GroupedCommandEditor {
  private commandGroup: IElement[] | null = null
  private isGrouping: boolean = false
  
  // 开始分组
  public startGroup() {
    this.isGrouping = true
  }
  
  // 结束分组
  public endGroup() {
    this.isGrouping = false
    this.saveHistory()
  }
  
  // 执行命令（带分组）
  public executeWithGroup(fn: () => void) {
    if (this.isGrouping) {
      fn()
    } else {
      this.saveHistory()
      fn()
    }
    this.computeRows()
    this.render()
  }
  
  // 拖拽选区时使用分组
  private handleMouseMove(e: MouseEvent) {
    if (!this.isDragging) return
    
    this.executeWithGroup(() => {
      this.endIndex = this.getIndexByPosition(x, y)
    })
  }
}
```

---

## 7.5 命令与渲染的分离

### 问题

当前所有命令都包含 `computeRows()` 和 `render()`，代码重复。

### 解决方案

使用"命令 + 批量渲染"模式：

```typescript
class DecoupledEditor {
  private isDirty: boolean = false
  
  // 标记需要重绘
  private markDirty() {
    this.isDirty = true
  }
  
  // 批量渲染（在动画帧中）
  private scheduleRender() {
    requestAnimationFrame(() => {
      if (this.isDirty) {
        this.computeRows()
        this.render()
        this.isDirty = false
      }
    })
  }
  
  // 所有命令都不再直接调用 render
  public executeInsert(text: string) {
    this.saveHistory()
    const element = this.createElement(text)
    this.elementList.splice(this.cursorIndex, 0, element)
    this.cursorIndex++
    this.markDirty()
    this.scheduleRender()
  }
  
  public executeBold() {
    this.saveHistory()
    // ...
    this.markDirty()
    this.scheduleRender()
  }
}
```

---

## 7.6 命令历史集成

```typescript
class HistoryCommandEditor {
  private history: HistoryManager = new HistoryManager()
  
  // 每个需要撤销的命令都要先保存
  public executeInsert(text: string) {
    this.saveHistory()  // ← 关键：保存状态
    
    // 执行操作
    const element = this.createElement(text)
    this.elementList.splice(this.cursorIndex, 0, element)
    this.cursorIndex++
    
    this.markDirty()
    this.scheduleRender()
  }
  
  // 撤销
  public undo() {
    const state = this.history.undo()
    if (state) {
      this.restoreState(state)
    }
  }
  
  // 重做
  public redo() {
    const state = this.history.redo()
    if (state) {
      this.restoreState(state)
    }
  }
}
```

---

## 7.7 公开 API 设计

```typescript
class Editor {
  private command: Command
  private canvas: HTMLCanvasElement
  private ctx: CanvasRenderingContext2D
  
  constructor(container: HTMLElement, options?: IEditorOptions) {
    // 初始化...
    this.command = new Command(this)
  }
  
  // 对外暴露的命令接口
  public getCommand(): Command {
    return this.command
  }
}

// 命令类
class Command {
  private editor: Editor
  
  constructor(editor: Editor) {
    this.editor = editor
  }
  
  // 文本操作
  public insert(text: string): void {
    this.editor.executeInsert(text)
  }
  
  public delete(): void {
    this.editor.executeDelete()
  }
  
  // 样式操作
  public bold(): void {
    this.editor.executeBold()
  }
  
  public setColor(color: string): void {
    this.editor.executeColor(color)
  }
  
  // 选区操作
  public selectAll(): void {
    this.editor.executeSelectAll()
  }
  
  // 历史操作
  public undo(): void {
    this.editor.undo()
  }
  
  public redo(): void {
    this.editor.redo()
  }
}
```

### 使用方式

```typescript
const editor = new Editor(container, options)

// 用户代码
editor.command.insert('Hello')
editor.command.bold()
editor.command.setColor('#FF0000')
editor.command.undo()
editor.command.redo()
```

---

## 7.8 本章总结

### 你学到了什么

1. **命令模式**：封装操作为独立方法
2. **API 设计**：清晰的命令接口
3. **命令分组**：避免连续操作产生碎片
4. **渲染分离**：命令与渲染解耦

### 核心概念

| 概念 | 作用 |
|------|------|
| 命令封装 | 把操作封装成方法 |
| 命令分组 | 避免碎片化撤销 |
| 批量渲染 | 减少渲染次数 |
| API 设计 | 对外提供清晰的接口 |

---

## 7.9 练习题

1. **基础练习**：为编辑器添加"复制"和"粘贴"命令

2. **进阶练习**：实现"组合命令"，比如"剪切" = 保存 + 删除 + 返回删除内容

3. **挑战练习**：实现"命令历史视图"，让用户可以看到所有执行过的命令

---

## 下一步

[第8章：图片与浮动元素](./PROGRESSIVE-08-image-float.md)

我们将学习：
- 图片元素的存储和渲染
- 图片的插入和删除
- 浮动元素的布局
