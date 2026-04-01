# 第8章：图片与浮动元素

> 实现图片插入和浮动布局

---

## 本章目标

```
学完本章后，你将能够：
1. 实现图片元素的存储和渲染
2. 实现图片的插入和删除
3. 理解浮动元素的布局逻辑
4. 处理图片加载的异步问题
```

---

## 8.1 图片元素的数据结构

```typescript
interface IElement {
  // 基础属性
  value: string
  
  // 图片属性
  type?: 'text' | 'image' | 'table'  // 元素类型
  imageUrl?: string    // 图片地址
  imageWidth?: number  // 图片宽度
  imageHeight?: number // 图片高度
  imageDisplay?: 'inline' | 'float-left' | 'float-right'  // 显示方式
  
  // 其他样式...
}
```

---

## 8.2 图片加载与缓存

```typescript
class ImageEditor {
  private imageCache: Map<string, HTMLImageElement> = new Map()
  
  // 加载图片
  private async loadImage(url: string): Promise<HTMLImageElement> {
    // 先检查缓存
    if (this.imageCache.has(url)) {
      return this.imageCache.get(url)!
    }
    
    return new Promise((resolve, reject) => {
      const img = new Image()
      img.crossOrigin = 'anonymous'
      img.onload = () => {
        this.imageCache.set(url, img)
        resolve(img)
      }
      img.onerror = () => {
        reject(new Error(`Failed to load image: ${url}`))
      }
      img.src = url
    })
  }
}
```

---

## 8.3 插入图片命令

```typescript
interface IImageOptions {
  url: string
  width?: number
  height?: number
  display?: 'inline' | 'float-left' | 'float-right'
}

class ImageEditor {
  public async executeInsertImage(options: IImageOptions) {
    this.saveHistory()
    
    const img = await this.loadImage(options.url)
    
    // 计算尺寸
    const maxWidth = this.options.width - this.options.padding * 2
    let width = options.width || img.width
    let height = options.height || img.height
    
    // 保持宽高比
    if (width > maxWidth) {
      const ratio = maxWidth / width
      width = maxWidth
      height = height * ratio
    }
    
    // 创建图片元素
    const imageElement: IElement = {
      type: 'image',
      value: '',
      imageUrl: options.url,
      imageWidth: width,
      imageHeight: height,
      imageDisplay: options.display || 'inline'
    }
    
    // 如果有选区，先删除选区内容
    if (this.startIndex !== this.endIndex) {
      this.deleteSelection()
    }
    
    // 插入图片
    this.elementList.splice(this.startIndex, 0, imageElement)
    this.startIndex++
    this.endIndex = this.startIndex
    
    this.computeRows()
    this.render()
  }
}
```

---

## 8.4 图片渲染

```typescript
class ImageEditor {
  private renderElement(element: IElement, x: number, y: number) {
    // 如果不是图片元素，绘制文字
    if (element.type !== 'image') {
      this.renderText(element, x, y)
      return
    }
    
    const img = this.imageCache.get(element.imageUrl!)
    if (!img) return
    
    // 绘制图片
    this.ctx.drawImage(
      img,
      x,
      y,
      element.imageWidth,
      element.imageHeight
    )
  }
}
```

---

## 8.5 浮动图片布局

### 浮动图片的特点

1. 不占用正常文档流
2. 文字环绕在图片周围
3. 需要计算"可用宽度"

### 实现思路

```typescript
class FloatImageEditor {
  // 计算行的可用宽度
  private computeRowAvailableWidth(row: IRow, elementIndex: number): number {
    const maxWidth = this.options.width - this.options.padding * 2
    
    // 统计当前行之前的浮动图片
    let floatLeftWidth = 0
    let floatRightWidth = 0
    
    for (let i = 0; i < elementIndex; i++) {
      const element = this.elementList[i]
      if (element.type === 'image' && element.imageDisplay === 'float-left') {
        floatLeftWidth = Math.max(floatLeftWidth, element.imageWidth || 0)
      }
      if (element.type === 'image' && element.imageDisplay === 'float-right') {
        floatRightWidth = Math.max(floatRightWidth, element.imageWidth || 0)
      }
    }
    
    return maxWidth - floatLeftWidth - floatRightWidth
  }
}
```

---

## 8.6 图片删除

```typescript
class ImageEditor {
  private handleBackspace() {
    // 检查光标前的元素
    const prevElement = this.elementList[this.startIndex - 1]
    
    if (prevElement && prevElement.type === 'image') {
      this.saveHistory()
      this.elementList.splice(this.startIndex - 1, 1)
      this.startIndex--
      this.endIndex = this.startIndex
      this.computeRows()
      this.render()
      return
    }
    
    // 普通删除逻辑...
  }
  
  // 删除图片
  public executeDeleteImage(index: number) {
    this.saveHistory()
    const element = this.elementList[index]
    
    if (element.type === 'image') {
      this.elementList.splice(index, 1)
      
      // 调整光标位置
      if (this.startIndex > index) {
        this.startIndex--
      }
      if (this.endIndex > index) {
        this.endIndex--
      }
      
      this.computeRows()
      this.render()
    }
  }
}
```

---

## 8.7 图片工具栏

```typescript
class ImageToolbar {
  public attachToEditor(editor: ImageEditor) {
    const imageButton = document.querySelector('#insert-image')
    
    imageButton.addEventListener('click', async () => {
      const url = prompt('请输入图片地址：')
      if (url) {
        await editor.executeInsertImage({ url })
      }
    })
  }
}
```

---

## 8.8 本章总结

### 你学到了什么

1. **图片数据结构**：用 type 区分元素类型
2. **图片加载**：异步加载 + 缓存
3. **图片渲染**：使用 Canvas drawImage
4. **浮动布局**：计算行的可用宽度

### 核心概念

| 概念 | 实现 |
|------|------|
| 图片元素 | `type: 'image'` |
| 图片加载 | `new Image()` + Promise |
| 图片缓存 | `Map<string, HTMLImageElement>` |
| 浮动布局 | 计算可用宽度 |

---

## 8.9 练习题

1. **基础练习**：实现"图片拖拽调整大小"

2. **进阶练习**：实现"图片题注"功能，在图片下方显示说明文字

3. **挑战练习**：实现"多图环绕"，同时有左侧和右侧图片

---

## 下一步

[第9章：表格基础](./PROGRESSIVE-09-table-basic.md)

我们将学习：
- 表格数据结构
- 单元格合并
- 表格绘制和编辑
