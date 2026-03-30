# 教程 08：Event 事件系统 —— 输入捕获与处理

> **前置知识**：建议先阅读教程 07（Command 命令系统），理解命令是如何执行操作的。

---

## 目录

1. [概述：事件系统的架构](#1-概述事件系统的架构)
2. [事件系统核心组件](#2-事件系统核心组件)
3. [CanvasEvent：画布事件的主入口](#3-canvasevent画布事件的主入口)
4. [GlobalEvent：全局窗口事件](#4- globalevent全局窗口事件)
5. [事件处理器详解：mousedown](#5-事件处理器详解mousedown)
6. [事件处理器详解：keydown](#6-事件处理器详解keydown)
7. [事件处理器详解：input](#7-事件处理器详解input)
8. [事件处理器详解：paste](#8-事件处理器详解paste)
9. [EventBus：发布订阅模式](#9- eventbus发布订阅模式)
10. [选区拖拽与剪贴板](#10-选区拖拽与剪贴板)
11. [核心要点](#11-核心要点)

---

## 1. 概述：事件系统的架构

编辑器的事件系统负责捕获用户的输入（键盘、鼠标、触摸），并将其转换为对文档的操作。

### 1.1 事件流向

```
用户操作（键盘/鼠标/触摸）
    ↓
浏览器事件（click/mousedown/keydown/input...）
    ↓
CanvasEvent / GlobalEvent 捕获
    ↓
事件处理器（handlers/*）处理
    ↓
CommandAdapt 执行命令
    ↓
Draw.render() 重新渲染
```

### 1.2 核心文件结构

```
src/editor/core/event/
├── CanvasEvent.ts          ← 画布事件主入口
├── GlobalEvent.ts           ← 全局窗口事件
├── eventbus/
│   └── EventBus.ts         ← 发布订阅模式
└── handlers/
    ├── mousedown.ts        ← 鼠标按下
    ├── mouseup.ts          ← 鼠标释放
    ├── mousemove.ts        ← 鼠标移动
    ├── mouseleave.ts       ← 鼠标离开
    ├── click.ts            ← 单击（包含双击、三击）
    ├── keydown/            ← 键盘按下
    │   ├── index.ts        ← 键盘事件分发
    │   ├── backspace.ts    ← 退格键
    │   ├── delete.ts       ← 删除键
    │   ├── enter.ts       ← 回车键
    │   ├── left.ts        ← 左箭头
    │   ├── right.ts       ← 右箭头
    │   ├── updown.ts      ← 上下箭头
    │   ├── home.ts        ← Home 键
    │   ├── end.ts         ← End 键
    │   └── tab.ts         ← Tab 键
    ├── input.ts           ← 文本输入
    ├── composition.ts     ← IME 输入
    ├── cut.ts             ← 剪切
    ├── copy.ts            ← 复制
    ├── paste.ts           ← 粘贴
    └── drop.ts           ← 拖拽放置
```

---

## 2. 事件系统核心组件

### 2.1 CanvasEvent

`CanvasEvent`（`src/editor/core/event/CanvasEvent.ts`）是画布事件的主入口：

```typescript
// src/editor/core/event/CanvasEvent.ts, lines 32-68
export class CanvasEvent {
  public isAllowSelection: boolean    // 是否允许选区
  public isComposing: boolean        // 是否正在 IME 输入
  public compositionInfo: ICompositionInfo | null  // IME 输入缓存
  public isAllowDrag: boolean         // 是否允许拖拽
  public isAllowDrop: boolean        // 是否允许放置
  public cacheRange: IRange | null   // 选区缓存（用于拖拽）
  public cacheElementList: IElement[] | null  // 元素列表缓存
  public cachePositionList: IElementPosition[] | null  // 位置列表缓存
  public mouseDownStartPosition: ICurrentPosition | null  // 鼠标按下位置

  private draw: Draw
  private pageContainer: HTMLDivElement
  private pageList: HTMLCanvasElement[]
  private range: RangeManager
  private position: Position
}
```

### 2.2 GlobalEvent

`GlobalEvent`（`src/editor/core/event/GlobalEvent.ts`）处理窗口级别的全局事件：

```typescript
// src/editor/core/event/GlobalEvent.ts, lines 16-45
export class GlobalEvent {
  // 监听的事件
  // - window blur：窗口失去焦点
  // - document mousedown：鼠标按下（全局）
  // - document mouseup：鼠标释放（全局）
  // - document wheel：滚轮（用于缩放）
  // - document visibilitychange：页面可见性变化
  // - DPR change：设备像素比变化
}
```

---

## 3. CanvasEvent：画布事件的主入口

### 3.1 事件注册

```typescript
// src/editor/core/event/CanvasEvent.ts, lines 74-87
public register() {
  this.pageContainer.addEventListener('click', this.click.bind(this))
  this.pageContainer.addEventListener('mousedown', this.mousedown.bind(this))
  this.pageContainer.addEventListener('mouseup', this.mouseup.bind(this))
  this.pageContainer.addEventListener('mouseleave', this.mouseleave.bind(this))
  this.pageContainer.addEventListener('mousemove', this.mousemove.bind(this))
  this.pageContainer.addEventListener('dblclick', this.dblclick.bind(this))
  this.pageContainer.addEventListener('dragover', this.dragover.bind(this))
  this.pageContainer.addEventListener('drop', this.drop.bind(this))
  threeClick(this.pageContainer, this.threeClick.bind(this))  // 三击
}
```

### 3.2 事件转发模式

CanvasEvent 使用转发模式，将事件分发给具体的处理器：

```typescript
// src/editor/core/event/CanvasEvent.ts
public mousedown(evt: MouseEvent) {
  mousedown(evt, this)  // 转发到处理器
}

public keydown(evt: KeyboardEvent) {
  keydown(evt, this)  // 转发到处理器
}

public input(data: string) {
  input(data, this)  // 转发到处理器
}
```

---

## 4. GlobalEvent：全局窗口事件

GlobalEvent 监听的是窗口和文档级别的事件：

```typescript
// src/editor/core/event/GlobalEvent.ts, lines 52-59
private addEvent() {
  window.addEventListener('blur', this.clearSideEffect)
  document.addEventListener('mousedown', this.clearSideEffect)
  document.addEventListener('mouseup', this.setCanvasEventAbility)
  document.addEventListener('wheel', this.setPageScale, { passive: false })
  document.addEventListener('visibilitychange', this._handleVisibilityChange)
  // ...
}
```

### 4.1 副作用清除 clearSideEffect

当用户点击编辑器外部时，清除所有浮动 UI：

```typescript
// src/editor/core/event/GlobalEvent.ts, lines 73-103
public clearSideEffect = (evt: Event) => {
  // 检查是否点击在编辑器内部
  const innerEditorDom = findParent(target, isPageNode, true)
  if (innerEditorDom) return  // 在内部，不处理

  // 清除所有浮动 UI
  this.cursor.recoveryCursor()              // 隐藏光标
  this.range.recoveryRangeStyle()            // 恢复选区样式
  this.previewer.clearResizer()             // 清除预览器
  this.tableTool.dispose()                  // 隐藏表格工具
  this.hyperlinkParticle.clearHyperlinkPopup()  // 隐藏超链接弹窗
  this.control.destroyControl()             // 销毁控件
  this.dateParticle.clearDatePicker()        // 隐藏日期选择器
  this.imageParticle.destroyFloatImage()     // 销毁浮动图片
}
```

### 4.2 缩放处理 setPageScale

```typescript
// src/editor/core/event/GlobalEvent.ts, lines 124-150
public setPageScale = (evt: WheelEvent) => {
  // 仅在按下 Ctrl 键时生效
  if (!evt.ctrlKey) return
  evt.preventDefault()
  if (evt.deltaY < 0) {
    // 放大
    this.draw.setPageScale(nextScale / 10)
  } else {
    // 缩小
    this.draw.setPageScale(nextScale / 10)
  }
}
```

---

## 5. 事件处理器详解：mousedown

`mousedown` 是最复杂的事件处理器之一，负责：
- 位置计算
- 选区设置
- Shift + 点击扩选
- 各种元素的命中检测

### 5.1 完整流程

```typescript
// src/editor/core/event/handlers/mousedown.ts, lines 64-262
export function mousedown(evt: MouseEvent, host: CanvasEvent) {
  const draw = host.getDraw()
  const rangeManager = draw.getRange()
  const position = draw.getPosition()

  // 1. 右键点击时，如果有选区则忽略
  if (evt.button === MouseEventButton.RIGHT && range.isCrossRowCol) {
    return
  }

  // 2. 检查是否在选区内部按下（可能触发拖拽）
  if (!host.isAllowDrag && !isReadonly && range.startIndex !== range.endIndex) {
    const isPointInRange = rangeManager.getIsPointInRange(evt.offsetX, evt.offsetY)
    if (isPointInRange) {
      setRangeCache(host)  // 缓存选区信息，准备拖拽
      return
    }
  }

  // 3. 计算点击位置
  const positionResult = position.adjustPositionContext({
    x: evt.offsetX,
    y: evt.offsetY
  })

  // 4. 解构位置结果
  const { index, isDirectHit, isCheckbox, isRadio, isImage, isLabel, isTable } = positionResult

  // 5. Shift + 点击扩选
  if (evt.shiftKey) {
    // 扩展选区到点击位置
    if (newPositionContext.tdId === oldPositionContext.tdId) {
      // 在同一单元格内，可以扩选
    }
  }

  // 6. 设置选区
  rangeManager.setRange(startIndex, endIndex)

  // 7. 处理特殊元素点击
  if (isDirectHitCheckbox) {
    hitCheckbox(curElement, draw)  // 切换复选框状态
  } else if (isDirectHitRadio) {
    hitRadio(curElement, draw)  // 切换单选框状态
  }

  // 8. 预览工具（图片调整大小手柄）
  if (isDirectHitImage) {
    previewer.drawResizer(curElement, positionList[curIndex], options)
  }

  // 9. 表格工具
  if (isTable && !isReadonly) {
    tableTool.render()  // 显示表格工具
  }

  // 10. 超链接
  if (curElement.type === ElementType.HYPERLINK) {
    if (isMod(evt)) {
      hyperlinkParticle.openHyperlink(curElement)  // Ctrl+点击打开链接
    } else {
      hyperlinkParticle.drawHyperlinkPopup(curElement)  // 显示链接弹窗
    }
  }

  // 11. 日期控件
  if (curElement.type === ElementType.DATE && !isReadonly) {
    dateParticle.renderDatePicker(curElement)  // 显示日期选择器
  }
}
```

### 5.2 命中检测结果

`adjustPositionContext` 返回的命中检测结果：

```typescript
interface ICurrentPosition {
  index: number            // 元素索引
  isDirectHit: boolean     // 是否精确命中元素
  isCheckbox: boolean     // 是否命中复选框
  isRadio: boolean        // 是否命中单选框
  isImage: boolean       // 是否命中图片
  isLabel: boolean       // 是否命中标签
  isTable: boolean       // 是否命中表格
  tdIndex?: number        // 单元格索引
  trIndex?: number        // 行索引
  // ...
}
```

---

## 6. 事件处理器详解：keydown

`keydown` 处理器负责所有键盘快捷键的处理。

### 6.1 键盘事件分发

```typescript
// src/editor/core/event/handlers/keydown/index.ts, lines 15-85
export function keydown(evt: KeyboardEvent, host: CanvasEvent) {
  if (host.isComposing) return  // IME 输入期间忽略

  if (evt.key === KeyMap.Backspace) {
    backspace(evt, host)
  } else if (evt.key === KeyMap.Delete) {
    del(evt, host)
  } else if (evt.key === KeyMap.Enter) {
    enter(evt, host)
  } else if (evt.key === KeyMap.Left) {
    if (isMod(evt)) {
      home(evt, host)  // Mac: Cmd+Left = Home
    } else {
      left(evt, host)
    }
  } else if (evt.key === KeyMap.Right) {
    if (isMod(evt)) {
      end(evt, host)  // Mac: Cmd+Right = End
    } else {
      right(evt, host)
    }
  } else if (isMod(evt) && evt.key.toLocaleLowerCase() === KeyMap.Z) {
    draw.getHistoryManager().undo()  // Ctrl+Z 撤销
    evt.preventDefault()
  } else if (isMod(evt) && evt.key.toLocaleLowerCase() === KeyMap.Y) {
    draw.getHistoryManager().redo()  // Ctrl+Y 重做
    evt.preventDefault()
  } else if (isMod(evt) && evt.key.toLocaleLowerCase() === KeyMap.C) {
    host.copy()  // Ctrl+C 复制
    evt.preventDefault()
  } else if (isMod(evt) && evt.key.toLocaleLowerCase() === KeyMap.X) {
    host.cut()  // Ctrl+X 剪切
    evt.preventDefault()
  } else if (isMod(evt) && evt.key.toLocaleLowerCase() === KeyMap.A) {
    host.selectAll()  // Ctrl+A 全选
    evt.preventDefault()
  } else if (isMod(evt) && evt.key.toLocaleLowerCase() === KeyMap.S) {
    // Ctrl+S 保存
    listener.saved(draw.getValue())
    evt.preventDefault()
  } else if (evt.key === KeyMap.ESC) {
    host.clearPainterStyle()  // ESC 取消格式刷
    evt.preventDefault()
  }
}
```

### 6.2 修饰键检测 isMod

```typescript
// src/editor/utils/hotkey.ts
export function isMod(evt: KeyboardEvent): boolean {
  const isMac = navigator.platform.toUpperCase().indexOf('MAC') >= 0
  return isMac ? evt.metaKey : evt.ctrlKey
}
```

---

## 7. 事件处理器详解：input

`input` 处理器负责文本输入。

### 7.1 输入流程

```typescript
// src/editor/core/event/handlers/input.ts, lines 13-120
export function input(data: string, host: CanvasEvent) {
  const draw = host.getDraw()
  if (draw.isReadonly() || draw.isDisabled()) return

  // 1. 移除正在合成的输入（IME 输入）
  removeComposingInput(host)

  // 2. 检查是否能输入
  const rangeManager = draw.getRange()
  if (!rangeManager.getIsCanInput()) return

  // 3. 获取默认样式
  const defaultStyle = rangeManager.getDefaultStyle() || host.compositionInfo?.defaultStyle

  // 4. 将文本拆分为元素数组
  const text = data.replaceAll(`\n`, ZERO)  // 换行符转为零宽字符
  const inputData = splitText(text).map(value => {
    const newElement: IElement = { value }
    // 复制光标位置的样式
    EDITOR_ELEMENT_COPY_ATTR.forEach(attr => {
      if (copyElement[attr] !== undefined) {
        newElement[attr] = copyElement[attr]
      }
    })
    return newElement
  })

  // 5. 如果有选区，先删除选区内容
  if (startIndex !== endIndex) {
    draw.spliceElementList(elementList, start, endIndex - startIndex)
  }

  // 6. 格式化上下文
  formatElementContext(elementList, inputData, startIndex, {
    editorOptions: draw.getOptions()
  })

  // 7. 插入新元素
  draw.spliceElementList(elementList, start, 0, inputData)

  // 8. 设置光标位置
  rangeManager.setRange(curIndex, curIndex)
  draw.render({
    curIndex,
    isSubmitHistory: !isComposing
  })
}
```

### 7.2 IME 输入支持

IME（输入法编辑器）输入需要特殊处理：

```typescript
// 合成开始
compositionstart() {
  host.isComposing = true
  // 缓存当前状态
}

// 合成结束
compositionend(evt: CompositionEvent) {
  host.isComposing = false
  // 将合成的文本作为普通输入处理
  host.input(evt.data)
}
```

---

## 8. 事件处理器详解：paste

`paste` 处理器负责粘贴操作。

### 8.1 粘贴来源

编辑器支持三种粘贴来源：

```typescript
// src/editor/core/event/handlers/paste.ts

// 1. 从剪贴板事件粘贴（Ctrl+V / Cmd+V）
pasteByEvent(host, evt: ClipboardEvent)

// 2. 通过 API 粘贴（editor.command.executePaste）
pasteByApi(host, options?: IPasteOption)

// 3. 内部粘贴（编辑器内部的复制数据）
pasteElement(host, elementList)
```

### 8.2 粘贴流程

```typescript
// pasteByEvent 的处理逻辑
export function pasteByEvent(host: CanvasEvent, evt: ClipboardEvent) {
  // 1. 检查自定义覆盖
  const overrideResult = paste(evt)
  if (overrideResult?.preventDefault) return

  // 2. 优先使用内部剪贴板数据
  if (editorClipboardData) {
    pasteElement(host, editorClipboardData.elementList)
    return
  }

  // 3. 从系统剪贴板提取数据
  for (const item of clipboardData.items) {
    if (item.kind === 'string') {
      if (item.type === 'text/html') {
        // HTML 粘贴
        pasteHTML(host, htmlText)
      } else if (item.type === 'text/plain') {
        // 纯文本粘贴
        host.input(plainText)
      }
    } else if (item.kind === 'file') {
      if (item.type.includes('image')) {
        // 图片粘贴
        pasteImage(host, file)
      }
    }
  }
}
```

---

## 9. EventBus：发布订阅模式

`EventBus`（`src/editor/core/event/eventbus/EventBus.ts`）实现发布订阅模式，用于解耦组件通信。

### 9.1 接口

```typescript
// src/editor/core/event/eventbus/EventBus.ts
export class EventBus<EventMap> {
  private eventHub: Map<string, Set<Function>>

  // 订阅
  public on<K extends keyof EventMap>(eventName: K, callback: EventMap[K]) {
    const eventSet = this.eventHub.get(eventName) || new Set()
    eventSet.add(callback)
    this.eventHub.set(eventName, eventSet)
  }

  // 发布
  public emit<K extends keyof EventMap>(eventName: K, payload?: ...) {
    const callBackSet = this.eventHub.get(eventName)
    callBackSet.forEach(callBack => callBack(payload))
  }

  // 取消订阅
  public off<K extends keyof EventMap>(eventName: K, callback: EventMap[K])

  // 检查是否订阅
  public isSubscribe<K extends keyof EventMap>(eventName: K): boolean
}
```

### 9.2 使用示例

```typescript
// 订阅保存事件
eventBus.on('saved', (value: IEditorResult) => {
  console.log('Document saved:', value)
})

// 订阅选区样式变化
eventBus.on('rangeStyleChange', (style: IRangeStyle) => {
  toolbar.updateStyle(style)
})

// 发布事件
eventBus.emit('saved', draw.getValue())
```

### 9.3 常用事件

| 事件名 | 触发时机 | 用途 |
|-------|--------|------|
| `saved` | Ctrl+S 保存时 | 通知外部保存 |
| `rangeStyleChange` | 选区样式变化时 | 更新工具栏状态 |
| `contentChange` | 内容变化时 | 通知外部内容已修改 |
| `pageSizeChange` | 页数变化时 | 更新页码显示 |
| `imageMousedown` | 图片点击时 | 触发外部图片预览 |
| `labelMousedown` | 标签点击时 | 触发外部标签处理 |
| `positionContextChange` | 位置上下文变化时 | 通知外部位置变化 |

---

## 10. 选区拖拽与剪贴板

### 10.1 选区拖拽流程

当用户在选区内部按下鼠标时，可能触发拖拽：

```typescript
// src/editor/core/event/handlers/mousedown.ts, lines 78-89
if (!host.isAllowDrag) {
  if (!isReadonly && range.startIndex !== range.endIndex) {
    const isPointInRange = rangeManager.getIsPointInRange(evt.offsetX, evt.offsetY)
    if (isPointInRange) {
      setRangeCache(host)  // 缓存选区信息
      return  // 等待后续 mousemove/mouseup 处理拖拽
    }
  }
}
```

### 10.2 拖拽缓存

```typescript
// src/editor/core/event/handlers/mousedown.ts, lines 16-26
export function setRangeCache(host: CanvasEvent) {
  host.cacheRange = deepClone(rangeManager.getRange())
  host.cacheElementList = draw.getElementList()
  host.cachePositionList = position.getPositionList()
  host.cachePositionContext = position.getPositionContext()
}
```

---

## 11. 核心要点

1. **事件分发模式**——CanvasEvent 作为主入口，将事件转发给具体处理器。

2. **mousedown 最复杂**——它负责位置计算、命中检测、选区设置、特殊元素处理。

3. **keydown 分发中心**——根据 `evt.key` 分发到不同的快捷键处理器。

4. **input 处理文本输入**——将输入文本拆分为元素数组，复制样式，然后插入文档。

5. **IME 输入支持**——通过 `compositionstart/compositionend` 事件处理中文、日文等输入法输入。

6. **paste 支持多种来源**——剪贴板事件、系统剪贴板 API、内部剪贴板数据。

7. **EventBus 解耦组件**——通过发布订阅模式实现组件间通信。

8. **GlobalEvent 处理窗口事件**——blur、visibilitychange、wheel 等全局事件。

---

## 下一步

现在你已经理解了 Event 事件系统，下一步可以学习 **Particle 粒子系统**（教程 09），了解文字、图片、表格等不同内容是如何在 Canvas 上绘制的。

**值得思考的问题**：

- 当用户三击（threeClick）时，为什么选择整个段落而不是整个文档？
- IME 输入时，`compositionInfo` 缓存了什么信息？
- 为什么粘贴图片时使用 `FileReader` 而不是直接使用 Blob？
- 拖拽选区时，`cacheRange` 和 `cacheElementList` 的作用是什么？

这些问题将在后续教程中一一解答。
