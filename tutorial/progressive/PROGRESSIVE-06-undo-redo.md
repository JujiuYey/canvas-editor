# 第6章：撤销与重做

> 实现 Ctrl+Z 撤销和 Ctrl+Y 重做功能

---

## 本章目标

```
学完本章后，你将能够：
1. 理解历史记录栈的设计
2. 实现撤销和重做功能
3. 处理历史记录的边界情况
4. 理解防抖在历史记录中的应用
```

---

## 6.1 历史记录的核心思想

### 什么是历史记录？

历史记录就是保存文档的"快照"，在需要时可以恢复到之前的状态。

### 两种实现方式

| 方式 | 优点 | 缺点 |
|------|------|------|
| **快照模式** | 实现简单 | 内存占用大 |
| **命令模式** | 内存占用小 | 实现复杂 |

本章实现**快照模式**，简单直观。

---

## 6.2 历史记录管理器

```typescript
interface IHistoryState {
  elementList: IElement[]  // 文档快照
  startIndex: number      // 光标位置
  endIndex: number        // 选区结束
}

class HistoryManager {
  private undoStack: IHistoryState[] = []  // 撤销栈
  private redoStack: IHistoryState[] = []  // 重做栈
  private maxCount: number = 100            // 最大记录数
  
  // 保存当前状态
  public save(state: IHistoryState) {
    // 深拷贝，避免引用问题
    this.undoStack.push(this.deepClone(state))
    
    // 清空重做栈（因为新操作会打断重做序列）
    this.redoStack = []
    
    // 限制栈大小
    if (this.undoStack.length > this.maxCount) {
      this.undoStack.shift()
    }
  }
  
  // 撤销
  public undo(): IHistoryState | null {
    if (this.undoStack.length <= 1) {
      return null // 至少保留初始状态
    }
    
    const currentState = this.undoStack.pop()!
    this.redoStack.push(currentState)
    
    return this.undoStack[this.undoStack.length - 1]
  }
  
  // 重做
  public redo(): IHistoryState | null {
    if (this.redoStack.length === 0) {
      return null
    }
    
    const state = this.redoStack.pop()!
    this.undoStack.push(state)
    
    return state
  }
  
  // 是否可以撤销
  public canUndo(): boolean {
    return this.undoStack.length > 1
  }
  
  // 是否可以重做
  public canRedo(): boolean {
    return this.redoStack.length > 0
  }
  
  // 深拷贝
  private deepClone<T>(obj: T): T {
    return JSON.parse(JSON.stringify(obj))
  }
  
  // 清空历史（用于加载新文档）
  public clear() {
    this.undoStack = []
    this.redoStack = []
  }
}
```

---

## 6.3 集成到编辑器

```typescript
class UndoRedoEditor {
  private history: HistoryManager = new HistoryManager()
  
  // 获取当前状态
  private getCurrentState(): IHistoryState {
    return {
      elementList: JSON.parse(JSON.stringify(this.elementList)), // 深拷贝
      startIndex: this.startIndex,
      endIndex: this.endIndex
    }
  }
  
  // 恢复状态
  private restoreState(state: IHistoryState) {
    this.elementList = JSON.parse(JSON.stringify(state.elementList))
    this.startIndex = state.startIndex
    this.endIndex = state.endIndex
    
    this.computeRows()
    this.render()
  }
  
  // 在操作前保存状态（由调用者触发）
  public saveHistory() {
    this.history.save(this.getCurrentState())
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
  
  // 快捷键
  private bindKeyboardEvents() {
    this.canvas.addEventListener('keydown', (e) => {
      // Ctrl/Cmd + Z：撤销
      if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) {
        e.preventDefault()
        this.undo()
      }
      
      // Ctrl/Cmd + Y 或 Ctrl/Cmd + Shift + Z：重做
      if ((e.ctrlKey || e.metaKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) {
        e.preventDefault()
        this.redo()
      }
    })
  }
}
```

---

## 6.4 保存时机

### 问题

什么时候应该保存历史记录？

### 策略

1. **操作后保存**：每次操作后保存
2. **防抖保存**：连续快速输入时，只保存最后一次
3. **智能保存**：区分"输入"和"命令"

### 实现

```typescript
class SmartHistoryEditor {
  private history: HistoryManager = new HistoryManager()
  private isComposing: boolean = false  // 是否在输入中
  private historyTimer: number | null = null
  
  // 文字输入（防抖）
  private handleInput(char: string) {
    // 先执行操作
    // ...
    
    // 防抖保存：延迟 500ms
    this.debouncedSaveHistory()
  }
  
  // 防抖保存
  private debouncedSaveHistory() {
    if (this.historyTimer) {
      clearTimeout(this.historyTimer)
    }
    
    this.historyTimer = setTimeout(() => {
      this.saveHistory()
      this.historyTimer = null
    }, 500)
  }
  
  // 命令操作（立即保存）
  private toggleBold() {
    this.saveHistory() // 立即保存
    
    // 执行命令
    // ...
  }
  
  // 删除操作（立即保存）
  private handleBackspace() {
    this.saveHistory() // 立即保存
    
    // 执行删除
    // ...
  }
}
```

---

## 6.5 性能优化

### 问题

每次保存都深拷贝整个文档，当文档很大时会很慢。

### 优化方案

1. **懒拷贝**：只在数据变化时才拷贝
2. **差量记录**：只记录变化的部分
3. **延迟保存**：使用 requestAnimationFrame

### 实现

```typescript
class OptimizedHistoryEditor {
  private history: HistoryManager = new HistoryManager()
  private isDirty: boolean = false  // 是否有未保存的变化
  
  // 标记为脏
  private markDirty() {
    this.isDirty = true
  }
  
  // 保存（优化版）
  private saveHistory() {
    if (!this.isDirty) return // 没有变化，不需要保存
    
    this.history.save(this.getCurrentState())
    this.isDirty = false
  }
  
  // 在每次操作后调用
  private handleInput(char: string) {
    // ...
    this.markDirty()
    // 渲染
    this.render()
    
    // 使用 requestAnimationFrame 保存
    requestAnimationFrame(() => {
      this.saveHistory()
    })
  }
}
```

---

## 6.6 边界情况处理

```typescript
class BoundaryAwareEditor {
  // 保存历史
  public saveHistory() {
    // 防止在输入过程中保存
    if (this.isComposing) return
    
    // 防止在渲染过程中保存
    if (this.isRendering) return
    
    this.history.save(this.getCurrentState())
  }
  
  // 撤销
  public undo() {
    if (!this.history.canUndo()) return
    
    const state = this.history.undo()
    if (state) {
      this.restoreState(state)
    }
  }
  
  // 重做
  public redo() {
    if (!this.history.canRedo()) return
    
    const state = this.history.redo()
    if (state) {
      this.restoreState(state)
    }
  }
  
  // 切换历史记录模式时
  public setHistoryMode(mode: 'sync' | 'async') {
    if (mode === 'sync') {
      // 同步模式：立即保存
      this.saveHistory()
    } else {
      // 异步模式：使用防抖
    }
  }
}
```

---

## 6.7 本章总结

### 你学到了什么

1. **历史记录栈**：撤销栈和重做栈
2. **状态快照**：保存完整的文档状态
3. **防抖优化**：避免频繁保存
4. **边界处理**：输入中、渲染中的特殊情况

### 核心概念

| 概念 | 作用 |
|------|------|
| 撤销栈 | 保存历史状态 |
| 重做栈 | 保存已撤销的状态 |
| 深拷贝 | 避免引用问题 |
| 防抖 | 减少保存次数 |

---

## 6.8 练习题

1. **基础练习**：实现"历史记录上限"功能，超过后删除最早的记录

2. **进阶练习**：实现"多步撤销"选择器（类似 Word 的下拉撤销列表）

3. **挑战练习**：实现"差量历史记录"，只记录变化的部分而不是整个文档

---

## 下一步

[第7章：命令系统](./PROGRESSIVE-07-command-system.md)

我们将学习：
- 命令模式
- 命令 API 设计
- 撤销重做与命令的结合
