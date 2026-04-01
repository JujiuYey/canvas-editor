# 第11章：增量渲染

> 性能优化：减少渲染次数和渲染范围

---

## 本章目标

```
学完本章后，你将能够：
1. 理解全量渲染 vs 增量渲染
2. 实现脏区域检测
3. 实现批量渲染
4. 优化大数据量场景
```

---

## 11.1 性能问题的根源

### 全量渲染的问题

```typescript
class NaiveEditor {
  public handleInput(char: string) {
    // 插入字符
    this.elementList.splice(this.cursorIndex, 0, char)
    this.cursorIndex++
    
    // 每次输入都重新计算所有行
    this.computeRows()  // 💀 1000行就要计算1000次
    
    // 每次输入都重新渲染所有内容
    this.render()       // 💀 1000行就要渲染1000次
  }
}
```

### 性能杀手

1. **全量计算**：每次输入都计算所有行
2. **全量渲染**：每次输入都渲染所有内容
3. **频繁调用**：输入时每秒可能调用几十次

---

## 11.2 增量渲染的核心思想

### 核心原则

> **只重绘变化的部分**

### 增量 vs 全量

| 方式 | 计算量 | 渲染量 | 适用场景 |
|------|--------|--------|----------|
| 全量 | O(n) | O(n) | 小文档 |
| 增量 | O(1) | O(1) | 大文档 |

---

## 11.3 脏区域检测

### 思路

记录哪些区域"变脏了"，只重绘脏的区域。

```typescript
interface IDirtyRegion {
  startRow: number  // 起始行
  endRow: number    // 结束行
  y: number         // y 坐标
  height: number    // 高度
}

class DirtyEditor {
  private dirtyRegions: IDirtyRegion[] = []
  
  // 标记脏区域
  private markDirty(startRow: number, endRow: number) {
    // 合并重叠的区域
    const newRegion = {
      startRow,
      endRow,
      y: this.rowList[startRow]?.y || 0,
      height: this.getRowsHeight(startRow, endRow)
    }
    
    this.dirtyRegions.push(newRegion)
    this.mergeDirtyRegions()
  }
  
  // 合并重叠的脏区域
  private mergeDirtyRegions() {
    if (this.dirtyRegions.length <= 1) return
    
    this.dirtyRegions.sort((a, b) => a.startRow - b.startRow)
    
    const merged = [this.dirtyRegions[0]]
    
    for (let i = 1; i < this.dirtyRegions.length; i++) {
      const last = merged[merged.length - 1]
      const current = this.dirtyRegions[i]
      
      if (current.startRow <= last.endRow + 1) {
        // 重叠，合并
        last.endRow = Math.max(last.endRow, current.endRow)
        last.height = this.getRowsHeight(last.startRow, last.endRow)
      } else {
        merged.push(current)
      }
    }
    
    this.dirtyRegions = merged
  }
  
  // 获取行范围的高度
  private getRowsHeight(startRow: number, endRow: number): number {
    let height = 0
    for (let i = startRow; i <= endRow; i++) {
      height += this.rowList[i].height
    }
    return height
  }
}
```

---

## 11.4 增量计算

### 问题

插入一个字符后，后面的所有行 x 坐标都可能变化。

### 解决方案

1. **只重新计算受影响的行**
2. **延迟计算**：批量处理

```typescript
class IncrementalEditor {
  private needsRecompute: boolean = false
  
  // 插入字符
  public handleInput(char: string) {
    // 记录插入位置
    const insertIndex = this.cursorIndex
    
    // 插入
    this.elementList.splice(insertIndex, 0, char)
    this.cursorIndex++
    
    // 标记需要重新计算
    this.needsRecompute = true
    this.markDirty(insertIndex, this.elementList.length - 1)
    
    // 触发增量渲染
    this.scheduleIncrementalRender()
  }
  
  // 批量调度渲染
  private scheduleIncrementalRender() {
    if (this.renderScheduled) return
    
    this.renderScheduled = true
    
    requestAnimationFrame(() => {
      if (this.needsRecompute) {
        // 重新计算受影响的行
        this.recomputeAffectedRows()
      }
      
      // 增量渲染
      this.incrementalRender()
      
      this.needsRecompute = false
      this.renderScheduled = false
    })
  }
  
  // 重新计算受影响的行
  private recomputeAffectedRows() {
    // 找到需要重新计算的行范围
    const affectedStartRow = this.findAffectedStartRow()
    const affectedEndRow = this.rowList.length - 1
    
    if (affectedStartRow > affectedEndRow) return
    
    // 从受影响的行开始，重新计算
    let currentX = this.options.padding
    let currentY = this.rowList[affectedStartRow]?.y || this.options.padding
    
    for (let i = affectedStartRow; i <= affectedEndRow; i++) {
      const row = this.rowList[i]
      
      // 重新计算这一行
      this.recomputeRow(row, currentX, currentY)
      
      currentY += row.height
    }
  }
  
  // 找到受影响的起始行
  private findAffectedStartRow(): number {
    // 从插入位置开始，找到对应的行
    let charCount = 0
    for (let i = 0; i < this.rowList.length; i++) {
      const row = this.rowList[i]
      if (charCount + row.elements.length > this.cursorIndex) {
        return i
      }
      charCount += row.elements.length
    }
    return 0
  }
}
```

---

## 11.5 增量渲染实现

```typescript
class IncrementalEditor {
  private incrementalRender() {
    if (this.dirtyRegions.length === 0) return
    
    // 渲染每个脏区域
    for (const region of this.dirtyRegions) {
      this.renderDirtyRegion(region)
    }
    
    // 清空脏区域
    this.dirtyRegions = []
  }
  
  private renderDirtyRegion(region: IDirtyRegion) {
    const { y, height } = region
    
    // 1. 清除脏区域（包括下面一小段，防止残留）
    this.ctx.clearRect(0, y - 10, this.canvas.width, height + 20)
    
    // 2. 重绘脏区域
    const startRow = region.startRow
    const endRow = region.endRow
    let currentY = this.rowList[startRow].y
    
    for (let i = startRow; i <= endRow; i++) {
      const row = this.rowList[i]
      this.renderRow(row, row.x, currentY)
      currentY += row.height
    }
  }
}
```

---

## 11.6 光标闪烁优化

### 问题

光标闪烁时每次都会触发全量渲染。

### 解决方案

光标单独渲染，不影响内容。

```typescript
class CursorOptimizedEditor {
  private cursorVisible: boolean = true
  private cursorAnimationId: number | null = null
  
  // 启动光标闪烁
  private startCursorBlink() {
    this.cursorAnimationId = setInterval(() => {
      this.cursorVisible = !this.cursorVisible
      
      // 只重绘光标区域（不重绘内容）
      this.renderCursorOnly()
    }, 530)  // 约每秒2次
  }
  
  // 只渲染光标
  private renderCursorOnly() {
    if (!this.cursorVisible) {
      // 清除光标
      const cursorRect = this.getCursorRect()
      this.ctx.clearRect(
        cursorRect.x - 1,
        cursorRect.y,
        cursorRect.width + 2,
        cursorRect.height
      )
      
      // 恢复光标位置的背景
      // ...
    } else {
      // 绘制光标
      this.drawCursor()
    }
  }
  
  // 内容变化时，光标闪烁重置
  private handleInput() {
    this.cursorVisible = true
    // 完整渲染（包含光标）
    this.render()
  }
}
```

---

## 11.7 批处理优化

### 问题

快速连续输入时，每次都触发渲染。

### 解决方案

使用防抖/节流，合并渲染。

```typescript
class BatchRenderEditor {
  private renderTask: number | null = null
  private readonly FRAME_BUDGET = 16  // 约60fps
  
  private scheduleRender() {
    // 取消之前的渲染任务
    if (this.renderTask !== null) {
      cancelAnimationFrame(this.renderTask)
    }
    
    this.renderTask = requestAnimationFrame(() => {
      this.performRender()
      this.renderTask = null
    })
  }
  
  private performRender() {
    // 如果需要重新计算
    if (this.needsRecompute) {
      this.recomputeAffectedRows()
      this.needsRecompute = false
    }
    
    // 如果有脏区域，增量渲染
    if (this.dirtyRegions.length > 0) {
      this.incrementalRender()
    } else {
      // 否则全量渲染
      this.fullRender()
    }
  }
}
```

---

## 11.8 虚拟滚动

### 问题

当文档有上万行时，即使增量渲染也可能很慢。

### 解决方案

只渲染可见区域的内容。

```typescript
class VirtualScrollEditor {
  private visibleStartRow: number = 0
  private visibleEndRow: number = 0
  private readonly OVERSCAN = 5  // 上下各多渲染5行
  
  private updateVisibleRange() {
    const viewportHeight = this.canvas.height
    const scrollTop = this.scrollContainer.scrollTop
    
    // 找到可见范围的起始行
    let accumulatedHeight = 0
    for (let i = 0; i < this.rowList.length; i++) {
      accumulatedHeight += this.rowList[i].height
      if (accumulatedHeight >= scrollTop) {
        this.visibleStartRow = Math.max(0, i - this.OVERSCAN)
        break
      }
    }
    
    // 找到可见范围的结束行
    accumulatedHeight = 0
    for (let i = 0; i < this.rowList.length; i++) {
      accumulatedHeight += this.rowList[i].height
      if (accumulatedHeight >= scrollTop + viewportHeight) {
        this.visibleEndRow = Math.min(this.rowList.length - 1, i + this.OVERSCAN)
        break
      }
    }
  }
  
  private renderVisibleOnly() {
    // 只渲染可见行
    for (let i = this.visibleStartRow; i <= this.visibleEndRow; i++) {
      const row = this.rowList[i]
      this.renderRow(row, row.x, row.y)
    }
  }
}
```

---

## 11.9 Web Worker 优化

### 问题

大量文本计算会阻塞主线程。

### 解决方案

将排版计算移到 Web Worker。

```typescript
// layout.worker.ts
self.onmessage = (e: MessageEvent) => {
  const { type, data } = e.data
  
  if (type === 'computeRows') {
    const result = computeRows(data.elements, data.options)
    self.postMessage({ type: 'rowsComputed', result })
  }
}

function computeRows(elements: IElement[], options: IOptions): IRow[] {
  // 排版计算逻辑
  // ...
  return rowList
}
```

```typescript
// main.ts
class WorkerEditor {
  private worker: Worker
  
  private async computeRowsAsync(elements: IElement[]) {
    return new Promise<IRow[]>((resolve) => {
      this.worker.postMessage({
        type: 'computeRows',
        data: { elements, options: this.options }
      })
      
      this.worker.onmessage = (e) => {
        if (e.data.type === 'rowsComputed') {
          resolve(e.data.result)
        }
      }
    })
  }
}
```

---

## 11.10 本章总结

### 你学到了什么

1. **脏区域检测**：只重绘变化的部分
2. **增量计算**：只重新计算受影响的行
3. **批量渲染**：合并多次渲染请求
4. **虚拟滚动**：只渲染可见区域
5. **Web Worker**：将计算移到后台线程

### 性能优化清单

| 优化点 | 方法 | 效果 |
|--------|------|------|
| 渲染范围 | 脏区域检测 | 减少 O(n) → O(1) |
| 计算范围 | 增量计算 | 减少 O(n) → O(Δn) |
| 渲染频率 | requestAnimationFrame | 合并渲染 |
| 可见范围 | 虚拟滚动 | 减少渲染量 |
| 主线程阻塞 | Web Worker | 不阻塞 UI |

---

## 11.11 练习题

1. **基础练习**：实现"撤销时闪烁"效果

2. **进阶练习**：实现"双缓冲渲染"，先画到离屏 Canvas，再拷贝到主 Canvas

3. **挑战练习**：实现"智能预渲染"，预测用户滚动方向，提前渲染下一页

---

## 下一步

[回到目录](./PROGRESSIVE-00-overview.md)

你已经完成了 Canvas 编辑器从零构建的完整教程！

### 回顾所学

1. **最小编辑器**：Canvas 绑定和基础渲染
2. **数据模型**：Element 和 Row 的设计
3. **排版系统**：文字换行和行高计算
4. **光标选区**：鼠标交互和选区高亮
5. **样式系统**：加粗、斜体、颜色等
6. **撤销重做**：历史记录栈
7. **命令系统**：清晰的 API 设计
8. **图片浮动**：图片插入和环绕布局
9. **表格基础**：表格渲染和单元格合并
10. **分页系统**：多页文档和页眉页脚
11. **增量渲染**：性能优化

### 下一步

- 阅读实际项目代码：`src/editor/`
- 运行 Cypress 测试：`npm run cypress:open`
- 尝试扩展功能：超链接、代码块、LaTeX 公式...

祝你编码愉快！ 🚀
