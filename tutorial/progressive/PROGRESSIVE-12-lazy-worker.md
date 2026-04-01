# 第12章：懒加载与 Worker

> 性能优化：懒加载图片和 Web Worker 异步计算

---

## 本章目标

```
学完本章后，你将能够：
1. 实现图片懒加载，减少首屏加载时间
2. 使用 IntersectionObserver 监听元素可见性
3. 将复杂计算移至 Web Worker，不阻塞主线程
4. 实现大文档下的流畅滚动
```

---

## 12.1 懒加载的必要性

### 问题

当文档中包含大量图片时：

1. **首屏加载慢**：需要下载所有图片
2. **内存占用高**：所有图片都驻留在内存
3. **流量浪费**：用户可能只看了前几页

### 解决方案

> **按需加载：只加载用户看得见的图片**

---

## 12.2 IntersectionObserver 简介

### 传统方案的问题

```typescript
// 传统方案：监听滚动事件
window.addEventListener('scroll', () => {
  const images = document.querySelectorAll('img[data-src]')
  images.forEach(img => {
    // 检查是否在视口内
    if (isInViewport(img)) {
      loadImage(img)
    }
  })
})

// 问题：滚动时会疯狂触发，性能差
```

### IntersectionObserver 方案

```typescript
// 创建观察者
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        // 进入视口，加载图片
        const img = entry.target as HTMLImageElement
        loadImage(img)
        // 停止观察这个图片
        observer.unobserve(img)
      }
    })
  },
  {
    root: null,           // 使用视口作为根
    rootMargin: '100px',  // 提前 100px 开始加载
    threshold: 0.1        // 10% 可见时触发
  }
)

// 开始观察
document.querySelectorAll('img[data-src]').forEach(img => {
  observer.observe(img)
})
```

---

## 12.3 Canvas 图片懒加载

### 图片数据结构

```typescript
interface ImageElement extends IElement {
  type: 'image'
  src: string            // 图片地址
  width: number          // 宽度
  height: number         // 高度
  loading: boolean       // 是否正在加载
  loaded: boolean        // 是否已加载
  naturalWidth?: number   // 原始宽度
  naturalHeight?: number  // 原始高度
  placeholder?: string   // 占位图
}
```

### 占位图机制

```typescript
class LazyImageEditor {
  private loadedImages: Map<string, HTMLImageElement> = new Map()
  private imageObserver: IntersectionObserver | null = null
  private pendingImages: Set<string> = new Set()
  
  // 初始化图片观察者
  private initImageObserver() {
    this.imageObserver = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            const img = entry.target as HTMLImageElement
            const src = img.dataset.src
            
            if (src && !this.loadedImages.has(src)) {
              this.loadImage(img, src)
            }
            
            this.imageObserver?.unobserve(img)
          }
        })
      },
      {
        rootMargin: '200px 0px',  // 提前 200px 加载
        threshold: 0
      }
    )
  }
  
  // 加载图片
  private loadImage(img: HTMLImageElement, src: string) {
    if (this.pendingImages.has(src)) return
    
    this.pendingImages.add(src)
    
    const image = new Image()
    
    image.onload = () => {
      // 缓存已加载的图片
      this.loadedImages.set(src, image)
      this.pendingImages.delete(src)
      
      // 标记元素已加载
      this.markImageLoaded(src, image)
      
      // 触发重绘
      this.scheduleRender()
    }
    
    image.onerror = () => {
      this.pendingImages.delete(src)
      console.error(`Failed to load image: ${src}`)
    }
    
    image.src = src
  }
  
  // 标记图片已加载
  private markImageLoaded(src: string, image: HTMLImageElement) {
    for (const element of this.elementList) {
      if (element.type === 'image' && element.src === src) {
        element.loaded = true
        element.naturalWidth = image.naturalWidth
        element.naturalHeight = image.naturalHeight
        
        // 根据宽高比计算显示尺寸
        if (!element.width || !element.height) {
          const maxWidth = 300
          const ratio = image.naturalWidth / image.naturalHeight
          element.width = Math.min(maxWidth, image.naturalWidth)
          element.height = element.width / ratio
        }
      }
    }
  }
  
  // 注册待观察的图片元素
  private observeImages() {
    const images = this.canvas.querySelectorAll('img[data-src]')
    images.forEach(img => {
      this.imageObserver?.observe(img)
    })
  }
}
```

### 渲染占位图

```typescript
class LazyImageEditor {
  // 渲染图片元素
  private renderImage(ctx: CanvasRenderingContext2D, element: ImageElement, x: number, y: number) {
    const { width, height, src, loaded } = element
    
    if (!loaded) {
      // 渲染占位图
      this.renderPlaceholder(ctx, x, y, width, height, src)
      return
    }
    
    // 渲染已加载的图片
    const image = this.loadedImages.get(src)
    if (image) {
      ctx.drawImage(image, x, y, width, height)
    }
  }
  
  // 渲染占位图
  private renderPlaceholder(ctx: CanvasRenderingContext2D, x: number, y: number, width: number, height: number, src: string) {
    // 背景
    ctx.fillStyle = '#f0f0f0'
    ctx.fillRect(x, y, width, height)
    
    // 边框
    ctx.strokeStyle = '#ddd'
    ctx.strokeRect(x, y, width, height)
    
    // 加载图标
    const centerX = x + width / 2
    const centerY = y + height / 2
    const iconSize = Math.min(width, height) * 0.3
    
    ctx.strokeStyle = '#999'
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.arc(centerX, centerY, iconSize, 0, Math.PI * 2)
    ctx.stroke()
    
    // 图片路径
    ctx.beginPath()
    ctx.moveTo(centerX - iconSize, centerY)
    ctx.lineTo(centerX, centerY - iconSize)
    ctx.lineTo(centerX + iconSize, centerY + iconSize)
    ctx.lineTo(centerX - iconSize, centerY + iconSize)
    ctx.closePath()
    ctx.fillStyle = '#999'
    ctx.fill()
  }
}
```

---

## 12.4 虚拟列表优化

### 问题

当文档有上万行时，渲染所有行仍然很慢。

### 解决方案

只渲染可见区域内的行 + 上下缓冲区。

```typescript
class VirtualListEditor {
  private viewportHeight: number = 0
  private scrollTop: number = 0
  private visibleStartIndex: number = 0
  private visibleEndIndex: number = 0
  private readonly OVERSCAN: number = 10  // 上下各多渲染 10 行
  
  // 更新可见范围
  public updateVisibleRange() {
    this.scrollTop = this.scrollContainer.scrollTop
    this.viewportHeight = this.scrollContainer.clientHeight
    
    // 找到可见起始行
    let accumulatedHeight = 0
    for (let i = 0; i < this.rowList.length; i++) {
      accumulatedHeight += this.rowList[i].height
      if (accumulatedHeight >= this.scrollTop) {
        this.visibleStartIndex = Math.max(0, i - this.OVERSCAN)
        break
      }
    }
    
    // 找到可见结束行
    accumulatedHeight = 0
    const visibleBottom = this.scrollTop + this.viewportHeight
    for (let i = 0; i < this.rowList.length; i++) {
      accumulatedHeight += this.rowList[i].height
      if (accumulatedHeight >= visibleBottom) {
        this.visibleEndIndex = Math.min(this.rowList.length - 1, i + this.OVERSCAN)
        break
      }
    }
  }
  
  // 计算总内容高度
  private computeTotalHeight(): number {
    return this.rowList.reduce((sum, row) => sum + row.height, 0)
  }
  
  // 渲染可见区域
  private renderVisibleRange() {
    // 清除整个画布
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
    
    // 计算偏移量（用于滚动定位）
    let yOffset = 0
    for (let i = 0; i < this.visibleStartIndex; i++) {
      yOffset += this.rowList[i].height
    }
    
    // 渲染可见行
    for (let i = this.visibleStartIndex; i <= this.visibleEndIndex; i++) {
      const row = this.rowList[i]
      this.renderRow(row, row.x, yOffset)
      yOffset += row.height
    }
  }
  
  // 处理滚动
  private handleScroll = () => {
    this.updateVisibleRange()
    this.renderVisibleRange()
  }
}
```

### 占位容器

```typescript
class VirtualListEditor {
  // 创建占位容器（用于正确滚动条高度）
  private createSpacer() {
    const spacer = document.createElement('div')
    spacer.className = 'virtual-spacer'
    spacer.style.height = `${this.computeTotalHeight()}px`
    spacer.style.width = '0'
    spacer.style.position = 'absolute'
    spacer.style.top = '0'
    spacer.style.left = '0'
    
    this.scrollContainer.appendChild(spacer)
    this.spacer = spacer
  }
  
  // 更新占位容器高度
  private updateSpacerHeight() {
    if (this.spacer) {
      this.spacer.style.height = `${this.computeTotalHeight()}px`
    }
  }
}
```

---

## 12.5 Web Worker 深入

### 什么时候用 Worker

适合移入 Worker 的任务：

| 任务 | 适合度 | 原因 |
|------|--------|------|
| 排版计算 | ⭐⭐⭐ | 计算密集，不涉及 DOM |
| 拼写检查 | ⭐⭐⭐ | 计算密集，异步返回 |
| 图片压缩 | ⭐⭐ | 依赖 Canvas API |
| 滚动处理 | ❌ | 需要即时响应 |

### Worker 通信协议

```typescript
// 定义消息类型
type WorkerMessage =
  | { type: 'computeRows'; payload: { elements: IElement[]; options: IOptions } }
  | { type: 'computeRowHeight'; payload: { text: string; font: string } }
  | { type: 'computePageBreak'; payload: { elements: IElement[]; pageHeight: number } }

type WorkerResponse =
  | { type: 'rowsComputed'; payload: IRow[] }
  | { type: 'rowHeightComputed'; payload: number }
  | { type: 'pageBreakComputed'; payload: number[] }
```

### Worker 实现

```typescript
// layout.worker.ts
interface IElement {
  text: string
  style?: { font?: string; bold?: boolean; italic?: boolean }
}

interface IRow {
  elements: IElement[]
  height: number
  y: number
}

interface IOptions {
  width: number
  fontSize: number
  fontFamily: string
  lineHeight: number
}

self.onmessage = (e: MessageEvent<WorkerMessage>) => {
  const { type } = e.data
  
  switch (type) {
    case 'computeRows':
      const rows = computeRows(e.data.payload.elements, e.data.payload.options)
      self.postMessage({ type: 'rowsComputed', payload: rows } as WorkerResponse)
      break
      
    case 'computeRowHeight':
      const height = measureTextHeight(e.data.payload.text, e.data.payload.font)
      self.postMessage({ type: 'rowHeightComputed', payload: height } as WorkerResponse)
      break
      
    case 'computePageBreak':
      const breaks = findPageBreaks(e.data.payload.elements, e.data.payload.pageHeight)
      self.postMessage({ type: 'pageBreakComputed', payload: breaks } as WorkerResponse)
      break
  }
}

function computeRows(elements: IElement[], options: IOptions): IRow[] {
  const rows: IRow[] = []
  let currentRow: IRow = { elements: [], height: 0, y: 0 }
  let currentX = 0
  
  for (const element of elements) {
    const elementWidth = measureTextWidth(element.text, element.style?.font || options.fontFamily)
    
    if (currentX + elementWidth > options.width) {
      // 需要换行
      rows.push(currentRow)
      currentRow = { elements: [], height: 0, y: rows.length * options.lineHeight }
      currentX = 0
    }
    
    currentRow.elements.push(element)
    currentRow.height = Math.max(currentRow.height, options.lineHeight)
    currentX += elementWidth
  }
  
  if (currentRow.elements.length > 0) {
    rows.push(currentRow)
  }
  
  return rows
}

function measureTextWidth(text: string, font: string): number {
  // 简化的宽度计算（实际需要使用 OffscreenCanvas）
  return text.length * 10
}

function measureTextHeight(text: string, font: string): number {
  return 20
}

function findPageBreaks(elements: IElement[], pageHeight: number): number[] {
  // 计算分页位置
  return [pageHeight, pageHeight * 2]
}
```

### 主线程调用

```typescript
// main.ts
class WorkerEditor {
  private worker: Worker | null = null
  private pendingRequests: Map<string, (data: any) => void> = new Map()
  
  // 初始化 Worker
  private initWorker() {
    this.worker = new Worker(new URL('./layout.worker.ts', import.meta.url), {
      type: 'module'
    })
    
    this.worker.onmessage = (e: MessageEvent<WorkerResponse>) => {
      const { type, payload } = e.data
      const callback = this.pendingRequests.get(type)
      
      if (callback) {
        callback(payload)
        this.pendingRequests.delete(type)
      }
    }
    
    this.worker.onerror = (error) => {
      console.error('Worker error:', error)
    }
  }
  
  // 异步计算行
  public async computeRowsAsync(elements: IElement[]): Promise<IRow[]> {
    return new Promise((resolve) => {
      this.pendingRequests.set('rowsComputed', resolve)
      
      this.worker?.postMessage({
        type: 'computeRows',
        payload: { elements, options: this.options }
      } as WorkerMessage)
    })
  }
  
  // 异步计算行高
  public async computeRowHeightAsync(text: string, font: string): Promise<number> {
    return new Promise((resolve) => {
      this.pendingRequests.set('rowHeightComputed', resolve)
      
      this.worker?.postMessage({
        type: 'computeRowHeight',
        payload: { text, font }
      } as WorkerMessage)
    })
  }
  
  // 销毁 Worker
  public destroy() {
    this.worker?.terminate()
    this.worker = null
  }
}
```

---

## 12.6 分片计算

### 问题

对于超长文档，即使在 Worker 中，计算也可能耗时过长。

### 解决方案

分片执行，每片执行后让出主线程。

```typescript
class ChunkedProcessor {
  private readonly CHUNK_SIZE = 1000  // 每片处理 1000 个元素
  
  // 分片计算
  public async computeInChunks<T, R>(
    items: T[],
    processor: (chunk: T[]) => R,
    onProgress?: (progress: number) => void
  ): Promise<R[]> {
    const results: R[] = []
    const totalChunks = Math.ceil(items.length / this.CHUNK_SIZE)
    
    for (let i = 0; i < totalChunks; i++) {
      const start = i * this.CHUNK_SIZE
      const end = Math.min(start + this.CHUNK_SIZE, items.length)
      const chunk = items.slice(start, end)
      
      // 处理这片数据
      const chunkResult = processor(chunk)
      results.push(chunkResult)
      
      // 报告进度
      if (onProgress) {
        onProgress((i + 1) / totalChunks)
      }
      
      // 让出主线程
      await this.yieldToMain()
    }
    
    return results
  }
  
  // 让出主线程
  private yieldToMain(): Promise<void> {
    return new Promise(resolve => {
      requestAnimationFrame(() => resolve())
    })
  }
}

// 使用示例
class ChunkedEditor {
  private async processLongDocument() {
    const processor = new ChunkedProcessor()
    
    const rows = await processor.computeInChunks(
      this.elementList,
      (chunk) => this.computeRowsForChunk(chunk),
      (progress) => {
        console.log(`Processing: ${Math.round(progress * 100)}%`)
      }
    )
    
    this.rowList = rows.flat()
  }
}
```

---

## 12.7 懒加载与 Worker 结合

### 完整实现

```typescript
class LazyWorkerEditor {
  private imageObserver: IntersectionObserver
  private worker: Worker
  private virtualList: VirtualListEditor
  
  constructor(canvas: HTMLCanvasElement) {
    this.imageObserver = this.createImageObserver()
    this.worker = this.createWorker()
    this.virtualList = new VirtualListEditor(canvas)
    
    this.setupScrollHandler()
  }
  
  // 渲染流程
  public render() {
    // 1. 虚拟列表：只渲染可见行
    this.virtualList.renderVisibleRange()
    
    // 2. 懒加载：检查图片是否进入视口
    this.checkImagesInViewport()
    
    // 3. Worker：后台计算下一页
    this.prefetchNextPage()
  }
  
  // 检查视口内的图片
  private checkImagesInViewport() {
    const viewportTop = this.virtualList.scrollTop
    const viewportBottom = viewportTop + this.virtualList.viewportHeight
    
    for (const element of this.visibleElements) {
      if (element.type === 'image' && !element.loaded) {
        if (this.isElementInViewport(element, viewportTop, viewportBottom)) {
          this.loadImage(element)
        }
      }
    }
  }
  
  // 预取下一页数据
  private prefetchNextPage() {
    const lastVisibleRow = this.virtualList.visibleEndIndex
    const nextPageStart = this.findPageStart(lastVisibleRow + 1)
    
    // 在 Worker 中计算下一页
    const elements = this.elementList.slice(nextPageStart, nextPageStart + 1000)
    
    this.worker.postMessage({
      type: 'computeRows',
      payload: elements
    })
  }
  
  private findPageStart(rowIndex: number): number {
    // 找到下一页的起始元素索引
    let charCount = 0
    for (let i = 0; i < this.rowList.length; i++) {
      if (i === rowIndex) return charCount
      charCount += this.rowList[i].elements.length
    }
    return charCount
  }
}
```

---

## 12.8 性能监控

### 监控指标

```typescript
class PerformanceMonitor {
  private metrics: {
    renderTime: number[]
    layoutTime: number[]
    imageLoadTime: number[]
  } = {
    renderTime: [],
    layoutTime: [],
    imageLoadTime: []
  }
  
  public recordRender(time: number) {
    this.metrics.renderTime.push(time)
    this.cleanOldMetrics()
  }
  
  public recordLayout(time: number) {
    this.metrics.layoutTime.push(time)
    this.cleanOldMetrics()
  }
  
  public recordImageLoad(time: number) {
    this.metrics.imageLoadTime.push(time)
  }
  
  public getStats() {
    return {
      avgRenderTime: this.average(this.metrics.renderTime),
      avgLayoutTime: this.average(this.metrics.layoutTime),
      p95RenderTime: this.percentile(this.metrics.renderTime, 95),
      loadedImages: this.metrics.imageLoadTime.length
    }
  }
  
  private average(arr: number[]): number {
    return arr.length > 0 ? arr.reduce((a, b) => a + b, 0) / arr.length : 0
  }
  
  private percentile(arr: number[], p: number): number {
    if (arr.length === 0) return 0
    const sorted = [...arr].sort((a, b) => a - b)
    const index = Math.ceil((p / 100) * sorted.length) - 1
    return sorted[Math.max(0, index)]
  }
  
  private cleanOldMetrics() {
    // 只保留最近 100 个数据点
    const maxPoints = 100
    for (const key of Object.keys(this.metrics) as Array<keyof typeof this.metrics>) {
      if (this.metrics[key].length > maxPoints) {
        this.metrics[key] = this.metrics[key].slice(-maxPoints)
      }
    }
  }
}
```

---

## 12.9 本章总结

### 你学到了什么

1. **图片懒加载**：使用 IntersectionObserver 按需加载
2. **虚拟列表**：只渲染可见区域 + 缓冲区
3. **Web Worker**：将计算密集任务移至后台线程
4. **分片计算**：避免长任务阻塞主线程
5. **性能监控**：量化分析渲染性能

### 性能优化清单

| 优化点 | 方法 | 效果 |
|--------|------|------|
| 图片加载 | 懒加载 + 占位图 | 减少首屏时间 |
| 渲染范围 | 虚拟列表 | 渲染量 = O(可见行) |
| 排版计算 | Web Worker | 不阻塞 UI |
| 长任务 | 分片计算 | 保持响应性 |
| 预取 | 预测加载 | 减少等待 |

---

## 12.10 练习题

1. **基础练习**：实现"图片加载进度条"效果

2. **进阶练习**：实现"无限滚动"，滚动到底部时自动加载更多内容

3. **挑战练习**：实现"滚动锚定"，滚动后保持当前阅读位置不变

---

## 回顾：完整学习路径

```
第1章 → 第2章 → 第3章 → 第4章 → 第5章 → 第6章 → 第7章
  ↓        ↓        ↓        ↓        ↓        ↓        ↓
最小运行  数据模型   排版     光标     样式     撤销     命令
                                                    (← 初级可用水准)
  ↓                                                   ↓
第8章 → 第9章 → 第10章 → 第11章 → 第12章
  ↓        ↓        ↓        ↓        ↓
图片     表格     分页     增量     Worker
                                    (← 生产可用水准)
```

---

## 恭喜完成！

你已完成 Canvas Editor 从零构建的完整教程！

### 所学技能

1. **基础架构**：Canvas 绑定、事件处理、数据驱动
2. **排版系统**：文字换行、行高计算、分页
3. **编辑功能**：光标、选区、撤销重做
4. **样式系统**：字体、颜色、加粗、斜体
5. **元素支持**：图片浮动、表格
6. **性能优化**：增量渲染、懒加载、Web Worker

### 下一步

- 阅读实际项目代码：`src/editor/`
- 运行 Cypress 测试：`npm run cypress:open`
- 尝试扩展功能：超链接、代码块、LaTeX 公式...
- 贡献代码：提交 PR 到 GitHub

祝你编码愉快！ 🚀

---

[回到目录](./PROGRESSIVE-00-overview.md)
