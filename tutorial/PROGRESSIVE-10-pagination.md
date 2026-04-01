# 第10章：分页系统

> 实现多页文档、分页符和页眉页脚

---

## 本章目标

```
学完本章后，你将能够：
1. 实现多页文档的排版
2. 实现分页符
3. 实现页码显示
4. 实现页眉页脚
```

---

## 10.1 分页的核心思路

### 连续模式 vs 分页模式

| 模式 | 特点 |
|------|------|
| 连续模式 | 一个无限长的 Canvas |
| 分页模式 | 多个固定高度的 Canvas |

### 分页算法

```typescript
interface IPage {
  rows: IRow[]      // 当前页的行
  startIndex: number // 起始元素索引
  height: number    // 页面高度
}

class PaginationEditor {
  private pageList: IPage[] = []
  
  // 计算分页
  private computePages() {
    const pageHeight = this.options.pageHeight
    const margins = this.options.margins
    const contentHeight = pageHeight - margins.top - margins.bottom
    
    this.pageList = []
    let currentPage: IPage = { rows: [], startIndex: 0, height: margins.top }
    let currentIndex = 0
    
    for (const row of this.rowList) {
      // 检查是否需要换页
      if (currentPage.height + row.height > contentHeight) {
        this.pageList.push(currentPage)
        currentPage = { rows: [], startIndex: currentIndex, height: margins.top }
      }
      
      // 添加行到当前页
      currentPage.rows.push(row)
      currentPage.height += row.height
      currentIndex++
    }
    
    // 最后一页
    if (currentPage.rows.length > 0) {
      this.pageList.push(currentPage)
    }
  }
}
```

---

## 10.2 分页符

### 分页符的数据结构

```typescript
interface IPageBreakElement {
  type: 'pageBreak'
  value: ''
}
```

### 分页符处理

```typescript
class PaginationEditor {
  private computePages() {
    // ... 
    
    for (const row of this.rowList) {
      // 检查当前行之前是否有分页符
      const elementBeforeRow = this.elementList[row.startIndex - 1]
      if (elementBeforeRow?.type === 'pageBreak') {
        // 强制换页
        this.pageList.push(currentPage)
        currentPage = { rows: [], startIndex: currentIndex, height: margins.top }
      }
      
      // ... 其他逻辑
    }
  }
}
```

---

## 10.3 多页渲染

### 渲染结构

```typescript
class PaginationEditor {
  private render() {
    // 清空所有页面
    this.clearAllPages()
    
    // 渲染每一页
    for (let pageIndex = 0; pageIndex < this.pageList.length; pageIndex++) {
      this.renderPage(pageIndex)
    }
    
    // 渲染光标（根据当前页）
    this.renderCursor()
  }
  
  private renderPage(pageIndex: number) {
    const page = this.pageList[pageIndex]
    const ctx = this.ctxList[pageIndex]
    
    // 清空页面
    ctx.fillStyle = '#ffffff'
    ctx.fillRect(0, 0, this.options.pageWidth, this.options.pageHeight)
    
    // 绘制背景
    this.renderPageBackground(ctx, pageIndex)
    
    // 绘制页眉
    if (this.header) {
      this.renderHeader(ctx, pageIndex)
    }
    
    // 绘制内容
    for (const row of page.rows) {
      this.renderRow(ctx, row)
    }
    
    // 绘制页脚
    if (this.footer) {
      this.renderFooter(ctx, pageIndex)
    }
    
    // 绘制页码
    this.renderPageNumber(ctx, pageIndex)
  }
}
```

---

## 10.4 页码显示

```typescript
class PaginationEditor {
  private renderPageNumber(ctx: CanvasRenderingContext2D, pageIndex: number) {
    const { pageWidth, pageHeight } = this.options
    const pageNumber = `${pageIndex + 1} / ${this.pageList.length}`
    
    ctx.font = '12px sans-serif'
    ctx.fillStyle = '#000'
    ctx.textAlign = 'center'
    ctx.fillText(pageNumber, pageWidth / 2, pageHeight - 20)
  }
}
```

---

## 10.5 页眉和页脚

### 数据结构

```typescript
interface IHeader {
  content: IElement[]
  height: number
}

interface IFooter {
  content: IElement[]
  height: number
}

class PaginationEditor {
  private header: IHeader | null = null
  private footer: IFooter | null = null
  
  public setHeader(elements: IElement[]) {
    this.header = {
      content: elements,
      height: 50  // 默认高度
    }
    this.computePages()
    this.render()
  }
}
```

### 渲染页眉

```typescript
private renderHeader(ctx: CanvasRenderingContext2D, pageIndex: number) {
  if (!this.header) return
  
  const { margins } = this.options
  
  ctx.font = '14px sans-serif'
  ctx.fillStyle = '#000'
  ctx.textAlign = 'center'
  
  let x = margins.left
  let y = margins.top / 2
  
  for (const element of this.header.content) {
    ctx.fillText(element.value, x, y)
    x += ctx.measureText(element.value).width
  }
}
```

---

## 10.6 滚动与页面切换

```typescript
class PaginationEditor {
  private currentPage: number = 0
  
  private handleScroll(deltaY: number) {
    // 计算滚动的页数
    const scrollAmount = deltaY / this.options.pageHeight
    const newPage = Math.max(0, Math.min(
      this.pageList.length - 1,
      this.currentPage + Math.round(scrollAmount)
    ))
    
    if (newPage !== this.currentPage) {
      this.currentPage = newPage
      this.scrollToPage(newPage)
    }
  }
  
  private scrollToPage(pageIndex: number) {
    const container = this.pageContainer
    const pageHeight = this.options.pageHeight + this.options.pageGap
    container.scrollTop = pageIndex * pageHeight
  }
}
```

---

## 10.7 懒加载渲染

```typescript
class LazyPaginationEditor {
  private intersectionObserver: IntersectionObserver
  
  constructor() {
    this.intersectionObserver = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            const pageIndex = Number(entry.target.dataset.pageIndex)
            this.renderPage(pageIndex)
          }
        })
      },
      { root: this.pageContainer }
    )
  }
  
  private observePages() {
    for (let i = 0; i < this.pageCount; i++) {
      this.intersectionObserver.observe(this.pages[i])
    }
  }
}
```

---

## 10.8 本章总结

### 你学到了什么

1. **分页算法**：计算每页的行数
2. **分页符**：强制换页
3. **多页渲染**：每个页面独立的 Canvas
4. **页眉页脚**：固定内容的绘制

### 核心概念

| 概念 | 说明 |
|------|------|
| 页面计算 | 根据高度分割内容 |
| 分页符 | 强制换页标记 |
| 页码 | 当前页/总页数 |
| 懒加载 | IntersectionObserver |

---

## 10.9 练习题

1. **基础练习**：实现"首页不同页眉"功能

2. **进阶练习**：实现"横排页面"（Landscape）

3. **挑战练习**：实现"书籍折手"排版

---

## 下一步

[第11章：增量渲染](./PROGRESSIVE-11-incremental-render.md)

我们将学习性能优化：
- 只重绘变化的区域
- 减少渲染次数
- 优化大数据量场景
