# 第9章：表格基础

> 实现表格的创建、渲染和编辑

---

## 本章目标

```
学完本章后，你将能够：
1. 设计表格的数据结构
2. 实现表格的绘制
3. 处理单元格合并
4. 实现表格的选区和编辑
```

---

## 9.1 表格的数据结构

### 简单方案：每个单元格是一个元素

```typescript
interface ITableElement {
  type: 'table'
  rows: ITableRow[]
  colCount: number
  rowCount: number
}

interface ITableRow {
  cells: ITableCell[]
  height: number
}

interface ITableCell {
  value: IElement[]  // 单元格内容
  colSpan: number     // 跨列数
  rowSpan: number     // 跨行数
  width: number       // 宽度
  height: number      // 高度
  backgroundColor?: string
  borderColor?: string
}
```

### 简化方案：一个表格作为一个元素

```typescript
interface ITableElement {
  type: 'table'
  value: ''  // 空值
  colgroup: { width: number }[]  // 列宽定义
  trList: ITr[]  // 行列表
  width: number
  height: number
}

interface ITr {
  id: string
  height: number
  tdList: ITd[]
}

interface ITd {
  id: string
  value: IElement[]  // 单元格内容
  rowspan: number
  colspan: number
  width: number
  height: number
  backgroundColor?: string
}
```

---

## 9.2 插入表格

```typescript
interface ITableOptions {
  rows: number
  cols: number
  colWidths?: number[]
}

class TableEditor {
  public executeInsertTable(options: ITableOptions) {
    this.saveHistory()
    
    const { rows, cols, colWidths } = options
    const maxWidth = this.options.width - this.options.padding * 2
    
    // 计算列宽
    const defaultColWidth = colWidths 
      ? colWidths.reduce((a, b) => a + b, 0) / cols
      : maxWidth / cols
    
    // 构建表格结构
    const colgroup = Array.from({ length: cols }, (_, i) => ({
      width: colWidths?.[i] || defaultColWidth
    }))
    
    const trList: ITr[] = Array.from({ length: rows }, () => ({
      id: this.generateId(),
      height: 40,  // 默认行高
      tdList: Array.from({ length: cols }, () => ({
        id: this.generateId(),
        value: [],  // 空单元格
        rowspan: 1,
        colspan: 1,
        width: defaultColWidth,
        height: 40
      }))
    }))
    
    // 计算表格总宽度和高度
    const width = colgroup.reduce((sum, col) => sum + col.width, 0)
    const height = rows * 40
    
    const tableElement: ITableElement = {
      type: 'table',
      value: '',
      colgroup,
      trList,
      width,
      height
    }
    
    // 如果有选区，先删除
    if (this.startIndex !== this.endIndex) {
      this.deleteSelection()
    }
    
    // 插入表格（作为独立元素）
    this.elementList.splice(this.startIndex, 0, tableElement as any)
    this.startIndex++
    this.endIndex = this.startIndex
    
    this.computeRows()
    this.render()
  }
}
```

---

## 9.3 表格渲染

```typescript
class TableEditor {
  private renderTable(element: ITableElement, x: number, y: number) {
    const { colgroup, trList } = element
    
    // 绘制边框
    this.ctx.strokeStyle = '#000'
    this.ctx.lineWidth = 1
    
    let currentY = y
    for (const tr of trList) {
      let currentX = x
      
      for (const td of tr.tdList) {
        // 绘制单元格背景
        if (td.backgroundColor) {
          this.ctx.fillStyle = td.backgroundColor
          this.ctx.fillRect(currentX, currentY, td.width, td.height)
        }
        
        // 绘制单元格边框
        this.ctx.strokeRect(currentX, currentY, td.width, td.height)
        
        // 绘制单元格内容
        this.renderCellContent(td, currentX, currentY)
        
        currentX += td.width
      }
      
      currentY += tr.height
    }
    
    // 绘制外边框
    this.ctx.strokeRect(x, y, element.width, element.height)
  }
  
  private renderCellContent(td: ITd, x: number, y: number) {
    // 渲染单元格内的文本
    let cellX = x + 5  // 内边距
    const cellY = y + 20  // 垂直居中
    
    for (const element of td.value) {
      this.renderElement(element, cellX, cellY)
      cellX += this.getElementWidth(element)
    }
  }
}
```

---

## 9.4 单元格合并

### 合并的数学关系

```
colspan: 单元格占据的列数
rowspan: 单元格占据的行数

如果 (row=0, col=0) 的单元格有 colspan=2, rowspan=2：
┌───────────┬───────────┐
│ colspan=2 │           │
│ rowspan=2 │           │
│           │           │
├───────────┼───────────┤
│           │           │
└───────────┴───────────┘

视觉上有 3 个单元格（1 个大 + 2 个小）
逻辑上有 4 个单元格（2x2 网格）
```

### 合并后的数据模型

```typescript
// 合并单元格时，标记被合并的单元格
const td: ITd = {
  id: this.generateId(),
  value: [...],  // 合并后的内容
  rowspan: 2,     // 跨 2 行
  colspan: 2,     // 跨 2 列
  width: colWidth0 + colWidth1,
  height: rowHeight0 + rowHeight1,
  // 被合并的单元格标记为 invisible
  invisible: false
}

// 被占据的单元格需要跳过
const invisibleTd: ITd = {
  id: this.generateId(),
  value: [],
  rowspan: 1,
  colspan: 1,
  invisible: true  // 标记为不可见
}
```

### 合并命令

```typescript
public executeMergeCells(startRow: number, startCol: number, endRow: number, endCol: number) {
  this.saveHistory()
  
  const table = this.getTableAtCursor()
  if (!table) return
  
  // 计算合并后的宽高
  let newWidth = 0
  let newHeight = 0
  
  for (let r = startRow; r <= endRow; r++) {
    newHeight += table.trList[r].height
  }
  
  for (let c = startCol; c <= endCol; c++) {
    newWidth += table.colgroup[c].width
  }
  
  // 收集所有被合并单元格的内容
  const content: IElement[] = []
  for (let r = startRow; r <= endRow; r++) {
    for (let c = startCol; c <= endCol; c++) {
      const td = table.trList[r].tdList[c]
      content.push(...td.value)
    }
  }
  
  // 设置合并后的主单元格
  const mainTd = table.trList[startRow].tdList[startCol]
  mainTd.rowspan = endRow - startRow + 1
  mainTd.colspan = endCol - startCol + 1
  mainTd.width = newWidth
  mainTd.height = newHeight
  mainTd.value = content
  
  // 标记被合并的单元格
  for (let r = startRow; r <= endRow; r++) {
    for (let c = startCol; c <= endCol; c++) {
      if (r === startRow && c === startCol) continue
      table.trList[r].tdList[c].invisible = true
      table.trList[r].tdList[c].value = []
    }
  }
  
  this.computeRows()
  this.render()
}
```

---

## 9.5 表格选区

### 表格选区的特点

1. 可以跨单元格选择
2. 选中区域可能是不规则的（因为合并）
3. 需要特殊的光标位置计算

### 实现

```typescript
class TableEditor {
  private tableSelection: {
    table: ITableElement
    startRow: number
    startCol: number
    endRow: number
    endCol: number
  } | null = null
  
  private getTableAtCursor(): ITableElement | null {
    // 获取光标所在的元素
    const element = this.elementList[this.startIndex]
    if (element?.type === 'table') {
      return element
    }
    return null
  }
  
  private getCellAtPosition(row: number, col: number): ITd | null {
    const table = this.getTableAtCursor()
    if (!table) return null
    
    const td = table.trList[row]?.tdList[col]
    if (td && !td.invisible) {
      return td
    }
    return null
  }
}
```

---

## 9.6 本章总结

### 你学到了什么

1. **表格数据结构**：行列结构、单元格合并
2. **表格渲染**：逐行逐列绘制
3. **单元格合并**：用 rowspan/colspan 表示
4. **表格选区**：特殊的选区处理

### 核心概念

| 概念 | 说明 |
|------|------|
| colgroup | 列宽定义 |
| trList | 行列表 |
| rowspan | 跨行数 |
| colspan | 跨列数 |
| invisible | 被合并的单元格 |

---

## 9.7 练习题

1. **基础练习**：实现"拆分单元格"功能

2. **进阶练习**：实现"插入行/列"功能

3. **挑战练习**：实现"拖拽调整列宽"功能

---

## 下一步

[第10章：分页系统](./PROGRESSIVE-10-pagination.md)

我们将学习：
- 多页文档
- 分页符
- 页码显示
- 页眉页脚
