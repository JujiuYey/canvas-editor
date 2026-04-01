# 教程 03：Editor 类 —— 画布编辑器的总指挥

> **前置知识**：建议先阅读教程 02（主入口），了解整体架构。

---

## 目录

1. [Editor 类是什么](#1-editor-类是什么)
2. [8 个公开属性](#2-8-个公开属性)
3. [构造器：启动所有子系统](#3-构造器启动所有子系统)
4. [命令模式：editor.command](#4-命令模式-editorcommand)
5. [适配器模式：CommandAdapt](#5-适配器模式-commandadapt)
6. [生命周期管理：destroy()](#6-生命周期管理-destroy)
7. [模块级别导出的内容](#7-模块级别导出的内容)
8. [整体串联](#8-整体串联)
9. [核心要点](#9-核心要点)

---

## 1. Editor 类是什么

`Editor` 类（`src/editor/index.ts`）是这个画布富文本编辑器**唯一的公开入口**。在应用中嵌入编辑器时，只需要实例化这一个类：

```typescript
import Editor from 'canvas-editor'

const container = document.getElementById('editor') as HTMLDivElement
const editor = new Editor(container, initialData, options)
```

可以把 `Editor` 想象成**乐团的指挥**——它自己不演奏任何乐器，但把所有的乐手（子系统）召集在一起，协调谁在什么时候演奏。它拥有编辑器的完整生命周期，创建所有核心子系统，将它们连接起来，并通过简洁的公开接口暴露出去。

---

## 2. 8 个公开属性

```typescript
// src/editor/index.ts, lines 79-86
export default class Editor {
  public command: Command           // 执行操作（加粗、插入文字等）
  public version: string            // 库版本号
  public listener: Listener         // 订阅变化回调
  public eventBus: EventBus<EventBusMap>  // 内部事件发布/订阅
  public override: Override        // 拦截粘贴/复制/拖放行为
  public register: Register        // 注册菜单、快捷键、i18n
  public destroy: () => void       // 清理函数
  public use: UsePlugin            // 安装插件
}
```

每个属性的作用：

| 属性 | 类型 | 作用 |
|---|---|---|
| `command` | `Command` | 所有编辑器操作的主要接口（格式化、插入、导航等） |
| `version` | `string` | 来自 `package.json` 的库版本 |
| `listener` | `Listener` | 基于回调的变化通知（内容变化、选区变化、页面大小变化等） |
| `eventBus` | `EventBus` | 精细化内部事件的发布/订阅系统 |
| `override` | `Override` | 允许插件/使用者拦截并覆盖粘贴、复制、拖放行为 |
| `register` | `Register` | 聚合了右键菜单、键盘快捷键和国际化翻译的注册接口 |
| `destroy` | `() => void` | 编辑器不再需要时，清理所有资源的函数 |
| `use` | `UsePlugin` | 插件安装函数 |

---

## 3. 构造器：启动所有子系统

构造器执行的是一个精心编排的 7 步初始化序列。

### 步骤 1：合并和标准化配置

```typescript
// src/editor/index.ts, lines 93-119
constructor(container, data, options = {}) {
  // 合并配置
  const editorOptions = mergeOption(options)      // 应用默认值
  data = deepClone(data)                          // 克隆，避免修改输入

  // 将数据拆分为三个区域：页眉、主体、页脚
  let headerElementList = []
  let mainElementList = []
  let footerElementList = []
  let graffitiData = []

  if (Array.isArray(data)) {
    // 简单 API：直接传入元素数组
    mainElementList = data
  } else {
    // 完整 API：IEditorData，分离的 header/main/footer
    headerElementList = data.header || []
    mainElementList = data.main
    footerElementList = data.footer || []
    graffitiData = data.graffiti || []
  }

  // 对所有区域的元素列表进行标准化（应用默认值、填补缺失属性）
  pageComponentData.forEach(elementList => {
    formatElementList(elementList, {
      editorOptions,
      isForceCompensation: true
    })
  })
```

**关键点**：编辑器支持两种数据输入格式——简单数组（快速上手）或完整 `IEditorData` 对象（含页眉/主体/页脚分区）。三个区域都通过 `formatElementList()` 进行标准化处理。

### 步骤 2：创建核心通信基础设施

```typescript
// src/editor/index.ts, lines 121-127
this.version = version
this.listener = new Listener()        // 回调式变化通知
this.eventBus = new EventBus<EventBusMap>()  // 精细化事件的发布/订阅
this.override = new Override()        // 插件钩子
```

这三个对象（`listener`、`eventBus`、`override`）构成了编辑器的**通信骨干**，在 `Draw` 之前创建，以便传入其中。

### 步骤 3：创建 Draw（渲染引擎）

```typescript
// src/editor/index.ts, lines 129-141
const draw = new Draw(
  container,
  editorOptions,
  {
    header: headerElementList,
    main: mainElementList,
    footer: footerElementList,
    graffiti: graffitiData
  },
  this.listener,      // 传入，让 Draw 可以触发回调
  this.eventBus,      // 传入，让 Draw 可以发出事件
  this.override       // 传入，让 Draw 可以检查被覆盖的行为
)
```

`Draw` 是核心渲染引擎（约 96KB）。在 `Draw` 内部，又实例化了更多的子系统：

- **粒子（Particle）**：`TextParticle`、`ImageParticle`、`TableParticle`、`HyperlinkParticle`、`LaTexParticle`、`CheckboxParticle` 等（每个负责绘制一种内容类型）
- **框架渲染器（Frame）**：`Header`、`Footer`、`Margin`、`Background`、`PageNumber`、`Watermark`、`LineNumber`
- **交互管理器**：`Search`、`Group`、`Area`
- **核心管理器**：`RangeManager`、`Position`、`HistoryManager`、`Zone`
- **观察者（Observer）**：`ScrollObserver`、`SelectionObserver`、`ImageObserver`、`MouseObserver`
- **Worker**：`WorkerManager`（异步操作，如字数统计、目录生成）

### 步骤 4：创建命令系统

```typescript
// src/editor/index.ts, lines 143
this.command = new Command(new CommandAdapt(draw))
```

### 步骤 5：创建右键菜单和快捷键系统

```typescript
// src/editor/index.ts, lines 145-153
const contextMenu = new ContextMenu(draw, this.command)
const shortcut = new Shortcut(draw, this.command)
this.register = new Register({
  contextMenu,
  shortcut,
  i18n: draw.getI18n()
})
```

### 步骤 6：绑定销毁函数

```typescript
// src/editor/index.ts, lines 155-160
this.destroy = () => {
  draw.destroy()
  shortcut.removeEvent()
  contextMenu.removeEvent()
  this.eventBus.dangerouslyClearAll()
}
```

### 步骤 7：初始化插件系统

```typescript
// src/editor/index.ts, lines 162-163
const plugin = new Plugin(this)
this.use = plugin.use.bind(plugin)
```

---

## 4. 命令模式：editor.command

`command` 属性是**编辑器程序化控制的主要接口**，暴露了 100 多个方法。

### 为什么这样设计？

CLAUDE.md 强调了关键架构决策：

> **命令与 Draw 分离**：命令通过 CommandAdapt 访问 Draw 功能，而不是直接访问。这样可以防止将 Draw 内部上下文暴露给外部使用者。

这意味着：
- 外部代码只看到扁平的、基于方法的 `Command` 接口
- 内部的 `Draw` 上下文被封装在 `CommandAdapt` 之后
- 即使内部架构变化，公开 API 保持稳定

### 命令方法分类

| 分类 | 示例 |
|---|---|
| **全局** | `executeCut`、`executeCopy`、`executePaste`、`executeSelectAll`、`executeUndo`、`executeRedo` |
| **文本格式** | `executeBold`、`executeItalic`、`executeUnderline`、`executeStrikeout`、`executeColor`、`executeHighlight` |
| **字体控制** | `executeFont`、`executeSize`、`executeSizeAdd`、`executeSizeMinus` |
| **结构** | `executeTitle`、`executeList`、`executeRowFlex`、`executeRowMargin` |
| **表格操作** | `executeInsertTable`、`executeDeleteTableRow`、`executeMergeTableCell`、`executeTableBorderColor` 等 |
| **媒体** | `executeImage`、`executeHyperlink`、`executeSeparator`、`executePageBreak` |
| **页面布局** | `executePageMode`、`executePageScale`、`executePaperSize`、`executeSetPaperMargin` |
| **控件** | `executeInsertControl`、`executeSetControlValue`、`executeLocationControl` |
| **Getter** | `getValue`、`getHTML`、`getText`、`getRange`、`getCatalog`、`getWordCount`、`getCursorPosition` |

### 命令 API 使用示例

```typescript
// 加粗文字
editor.command.executeBold()

// 插入图片
editor.command.executeImage({
  value: 'data:image/png;base64,...',
  width: 200,
  height: 150
})

// 获取当前文档内容
const result = editor.command.getValue()

// 导航到文档中的标题
editor.command.executeLocationCatalog(titleId)
```

### 命名约定

每个方法遵循两种模式之一：
- **`execute*`**：修改文档的变更操作（如 `executeBold`、`executeInsertTable`）
- **`get*`**：检索信息的只读操作（如 `getValue`、`getRange`）

---

## 5. 适配器模式：CommandAdapt

`CommandAdapt`（`src/editor/core/command/CommandAdapt.ts`）是公开 `Command` 外观与内部 `Draw` 上下文之间的桥梁。

### CommandAdapt 持有的引用

```typescript
// src/editor/core/command/CommandAdapt.ts, lines 141-153
export class CommandAdapt {
  private draw: Draw                    // 渲染引擎
  private range: RangeManager           // 选区管理
  private position: Position            // 光标/元素定位
  private historyManager: HistoryManager // 撤销/重做
  private canvasEvent: CanvasEvent       // 输入事件处理
  private options: DeepRequired<IEditorOption>
  private control: Control               // 表单控件（复选框等）
  private workerManager: WorkerManager   // 异步 Worker
  private searchManager: Search           // 搜索功能
  private i18n: I18n                    // 国际化
  private zone: Zone                     // 页眉/主体/页脚区域
  private tableOperate: TableOperate     // 表格操作
}
```

它在构造器中从 `Draw` 获取所有这些引用：

```typescript
constructor(draw: Draw) {
  this.draw = draw
  this.range = draw.getRange()
  this.position = draw.getPosition()
  this.historyManager = draw.getHistoryManager()
  this.canvasEvent = draw.getCanvasEvent()
  // ... 以此类推
}
```

### 命令如何绑定

`Command` 类从 `CommandAdapt` 中取出每个方法并绑定：

```typescript
// src/editor/core/command/Command.ts, lines 152-153
this.executeMode = adapt.mode.bind(adapt)
this.executeCut = adapt.cut.bind(adapt)
this.executeCopy = adapt.copy.bind(adapt)
// ... 100+ 个方法
```

这种绑定创建了一个稳定的、引用固定的公开 API。绑定确保方法内部的 `this` 始终指向 `CommandAdapt`。

### 具体示例：`executeBold` 的完整流程

```
editor.command.executeBold()
  -> Command.executeBold（从 CommandAdapt.bold 绑定）
  -> CommandAdapt.bold()
    1. 检查是否只读/禁用 -> 提前返回
    2. 通过 range.getSelectionElementList() 获取选中元素
    3. 有选区：切换所有选中元素的 bold
    4. 无选区：切换光标所在行的 bold + 更新默认样式
    5. 调用 draw.render() 以新状态重绘
```

---

## 6. 生命周期管理：destroy()

`destroy` 函数是 Editor API 的关键部分。编辑器从 DOM 中移除时必须调用它，以防止内存泄漏。

```typescript
// src/editor/index.ts, lines 155-160
this.destroy = () => {
  draw.destroy()              // 销毁所有 canvas 元素、观察者、Worker
  shortcut.removeEvent()      // 移除文档上的键盘事件监听
  contextMenu.removeEvent()    // 移除右键菜单事件监听
  this.eventBus.dangerouslyClearAll()  // 清除所有事件订阅
}
```

**务必调用 `editor.destroy()`**，否则会导致内存泄漏。`draw.destroy()` 会依次：
- 移除 IntersectionObserver（懒加载渲染）
- 终止 Web Worker
- 清空 canvas 上下文
- 移除所有 DOM 元素上的事件监听

---

## 7. 模块级别导出的内容

Editor 模块身兼两职——既是入口类，又是所有导出类型、枚举、工具函数和常量的命名空间。

```typescript
// src/editor/index.ts, lines 167-241

// 工具函数
export { splitText, createDomFromElementList, getElementListByHTML, getTextFromElementList }

// 常量
export { EDITOR_COMPONENT, LETTER_CLASS, INTERNAL_CONTEXT_MENU_KEY,
        INTERNAL_SHORTCUT_KEY, EDITOR_CLIPBOARD }

// 枚举（约 20+ 个）
export { Editor, RowFlex, VerticalAlign, EditorZone, EditorMode, ElementType,
         ControlType, EditorComponent, PageMode, RenderMode, ImageDisplay,
         Command, KeyMap, BlockType, PaperDirection, TableBorder, TdBorder,
         TdSlash, MaxHeightRatio, NumberType, TitleLevel, ListType, ListStyle,
         WordBreak, ControlIndentation, ControlComponent, BackgroundRepeat,
         BackgroundSize, TextDecorationStyle, LineNumberType, LocationPosition,
         AreaMode, ControlState, FlexDirection, WatermarkType }

// 类型
export type { IElement, IEditorData, IEditorOption, IEditorResult,
              IContextMenuContext, IRegisterContextMenu, IWatermark,
              INavigateInfo, IBlock, ILang, ICatalog, ICatalogItem,
              IRange, IRangeStyle, IBadge, IGetElementListByHTMLOption }
```

这样使用者可以一次性导入所有需要的内容：

```typescript
import Editor, { ElementType, EditorZone, IElement } from 'canvas-editor'
```

---

## 8. 整体串联

完整的初始化流程：

```
new Editor(container, data, options)
│
├─ mergeOption()           标准化用户配置与默认值
├─ deepClone()              克隆输入数据，避免修改原数据
├─ formatElementList()      为所有区域的元素应用默认值
│
├─ new Listener()            创建回调式通知系统
├─ new EventBus()           创建发布/订阅事件系统
├─ new Override()           创建插件钩子点
│
├─ new Draw()                创建渲染引擎
│   ├─ new Position()             光标/元素定位
│   ├─ new RangeManager()          选区管理
│   ├─ new HistoryManager()        撤销/重做栈
│   ├─ new CanvasEvent()          输入事件处理
│   ├─ new GlobalEvent()          文档级事件处理
│   ├─ new Cursor()               光标渲染
│   ├─ new WorkerManager()        Web Worker 异步任务
│   ├─ new [30+ Particle 类]      内容渲染器
│   ├─ new [10+ Frame 类]          布局装饰
│   ├─ new [4+ Observer 类]        DOM 观察
│   └─ new Actuator()             位置上下文变更处理
│
├─ new Command(new CommandAdapt(draw))
│   └─ 绑定 100+ 个 CommandAdapt 方法
│
├─ new ContextMenu(draw, command)
├─ new Shortcut(draw, command)
├─ new Register({ contextMenu, shortcut, i18n })
│
├─ new Plugin(this) -> this.use
│
└─ this.destroy = () => { ... }  清理函数
```

---

## 9. 核心要点

1. **Editor 是一个薄的编排层**——它自己不负责渲染或文档管理，而是创建并连接所有做这些事的子系统。

2. **`command` 是主要 API**——所有编辑器操作都通过 `editor.command.execute*()` 或 `editor.command.get*()`，很少需要直接访问内部子系统。

3. **三种通信模式共存**：
   - `listener`：回调式（简单，适合 `contentChange` 等常见场景）
   - `eventBus`：发布/订阅（强大，适合精细化事件订阅）
   - `override`：钩子式（用于拦截粘贴、复制、拖放操作）

4. **Draw 是核心**——它是最大的子系统（约 96KB），拥有所有渲染、文档状态和大部分管理器。其他一切都是委托给 Draw 或与 Draw 通信。

5. **CommandAdapt 是桥梁**——它封装了 Draw 内部细节，暴露了一个扁平的、基于方法的接口。这是保持公开 API 干净和稳定的关键设计决策。

6. **务必调用 `destroy()`**——Editor 创建了 DOM 元素、Web Worker、事件监听器和 IntersectionObserver。不调用 `destroy()` 会导致内存泄漏。

---

## 下一步

现在你已经理解了 Editor 如何编排所有子系统，下一步自然深入 **Draw 渲染引擎**（教程 04：渲染引擎）了解画布实际如何绘制。

**第四章内容预告**：
- Draw 如何管理多个 Canvas 页面
- computeRowList 排版算法的核心逻辑
- 粒子系统如何绘制文字、图片、表格等内容
- Frame 框架系统（页眉、页脚、水印）
- IntersectionObserver 懒加载渲染

**值得思考的问题**：
- `Draw.render()` 如何在一帧内协调 30+ 个粒子渲染器？
- `Listener` 回调系统与 `EventBus` 发布/订阅有何不同？
- `Override` 系统如何允许插件拦截粘贴操作？

这些问题将在后续教程中一一解答。
