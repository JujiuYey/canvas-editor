# 教程 07：Command 命令系统 —— 操作的外露接口

> **前置知识**：建议先阅读教程 06（Position 与 Range 系统），理解选区和光标的工作原理。

---

## 目录

1. [概述：命令系统的设计哲学](#1-概述命令系统的设计哲学)
2. [三层架构：Command → CommandAdapt → Draw](#2-三层架构command--commandadapt--draw)
3. [Command 类：对外的薄 facade](#3-command-类对外的薄-facade)
4. [CommandAdapt 类：真正的实现者](#4-commandadapt-类真正的实现者)
5. [命令分类：140+ 个命令](#5-命令分类140-个命令)
6. [典型命令实现分析：bold 加粗](#6-典型命令实现分析bold-加粗)
7. [典型命令实现分析：insertElementList 插入元素](#7-典型命令实现分析insertelementlist-插入元素)
8. [只读/禁用检查模式](#8-只读禁用检查模式)
9. [render 在命令中的使用模式](#9-render-在命令中的使用模式)
10. [核心要点](#10-核心要点)

---

## 1. 概述：命令系统的设计哲学

Command 系统是编辑器对外暴露的操作接口。用户通过工具栏、快捷键、API 调用的任何操作，最终都通过 Command 执行。

### 1.1 设计目标

```
用户操作（工具栏/快捷键/API）
    ↓
Command.executeXXX()  ← 统一入口
    ↓
CommandAdapt.XXX()   ← 实际逻辑
    ↓
Draw.render()        ← 重新渲染
```

### 1.2 核心文件

| 文件 | 职责 |
|-----|------|
| `Command.ts` | facade，仅做方法转发 |
| `CommandAdapt.ts` | 真正的实现，包含所有命令逻辑 |

### 1.3 为什么需要 CommandAdapt？

`CommandAdapt` 是命令模式中的 **Adapter**，它：
- 持有 `Draw` 的引用
- 封装了所有对 Draw 的操作
- 防止 Draw 的内部实现暴露给外部

注释明确说明了这一点：

```typescript
// src/editor/core/command/Command.ts, line 3
// 通过CommandAdapt中转避免直接暴露编辑器上下文
```

---

## 2. 三层架构：Command → CommandAdapt → Draw

```
┌─────────────────────────────────────────┐
│          Command（对外接口）              │
│   editor.command.executeBold()          │
│   editor.command.executeInsertTable()   │
└─────────────────┬───────────────────────┘
                  │ 方法转发
                  ▼
┌─────────────────────────────────────────┐
│         CommandAdapt（实现层）            │
│   bold() { ... }                       │
│   insertTable() { ... }                │
│   - 持有 Draw、RangeManager、HistoryManager 等│
└─────────────────┬───────────────────────┘
                  │ 调用
                  ▼
┌─────────────────────────────────────────┐
│          Draw（渲染引擎）                │
│   render() { ... }                     │
│   spliceElementList() { ... }          │
│   - 实际修改数据并触发重绘               │
└─────────────────────────────────────────┘
```

---

## 3. Command 类：对外的薄 facade

`Command` 类（`src/editor/core/command/Command.ts`）只有构造器，没有任何独立逻辑。

### 3.1 结构

```typescript
// src/editor/core/command/Command.ts, lines 4-313
export class Command {
  // 140+ 个方法，全是方法转发
  public executeBold: CommandAdapt['bold']
  public executeItalic: CommandAdapt['italic']
  public executeFont: CommandAdapt['font']
  // ... 全部从 CommandAdapt 转发

  constructor(adapt: CommandAdapt) {
    // 每个方法都 bind 到 adapt
    this.executeBold = adapt.bold.bind(adapt)
    this.executeItalic = adapt.italic.bind(adapt)
    // ...
  }
}
```

### 3.2 使用方式

外部通过 `Editor.command` 访问：

```typescript
// 加粗
editor.command.executeBold()

// 插入表格
editor.command.executeInsertTable(3, 3)

// 撤销
editor.command.executeUndo()
```

---

## 4. CommandAdapt 类：真正的实现者

`CommandAdapt`（`src/editor/core/command/CommandAdapt.ts`）持有所有子系统的引用。

### 4.1 构造器

```typescript
// src/editor/core/command/CommandAdapt.ts, lines 155-168
export class CommandAdapt {
  private draw: Draw
  private range: RangeManager
  private position: Position
  private historyManager: HistoryManager
  private canvasEvent: CanvasEvent
  private options: DeepRequired<IEditorOption>
  private control: Control
  private workerManager: WorkerManager
  private searchManager: Search
  private i18n: I18n
  private zone: Zone
  private tableOperate: TableOperate

  constructor(draw: Draw) {
    this.draw = draw
    this.range = draw.getRange()
    this.position = draw.getPosition()
    this.historyManager = draw.getHistoryManager()
    this.canvasEvent = draw.getCanvasEvent()
    this.options = draw.getOptions()
    this.control = draw.getControl()
    this.workerManager = draw.getWorkerManager()
    this.searchManager = draw.getSearch()
    this.i18n = draw.getI18n()
    this.zone = draw.getZone()
    this.tableOperate = draw.getTableOperate()
  }
}
```

---

## 5. 命令分类：140+ 个命令

Command 系统包含 140+ 个命令，分为以下几类：

### 5.1 全局命令（16 个）

| 命令 | 方法 | 说明 |
|-----|------|------|
| 模式 | `executeMode` | 设置编辑器模式 |
| 剪贴板 | `executeCut/Copy/Paste` | 剪切/复制/粘贴 |
| 选择 | `executeSelectAll` | 全选 |
| 光标 | `executeSetRange/ReplaceRange/Blur/HideCursor` | 选区/光标操作 |
| 撤销/重做 | `executeUndo/Redo` | 历史记录 |
| 格式刷 | `executePainter/ApplyPainterStyle` | 格式刷 |
| 清除格式 | `executeFormat` | 清除选区格式 |
| 强制更新 | `executeForceUpdate` | 强制重绘 |

### 5.2 文字样式命令（12 个）

| 命令 | 方法 |
|-----|------|
| 字体 | `executeFont` |
| 字号 | `executeSize/SizeAdd/SizeMinus` |
| 加粗 | `executeBold` |
| 斜体 | `executeItalic` |
| 下划线 | `executeUnderline` |
| 删除线 | `executeStrikeout` |
| 上标 | `executeSuperscript` |
| 下标 | `executeSubscript` |
| 字体颜色 | `executeColor` |
| 背景色 | `executeHighlight` |

### 5.3 段落样式命令（4 个）

| 命令 | 方法 | 说明 |
|-----|------|------|
| 标题 | `executeTitle` | H1-H6 |
| 列表 | `executeList` | 有序/无序 |
| 对齐 | `executeRowFlex` | 左/中/右/两端 |
| 行间距 | `executeRowMargin` | 段落间距 |

### 5.4 表格命令（20 个）

| 命令 | 说明 |
|-----|------|
| `executeInsertTable` | 插入表格 |
| `executeInsertTableTopRow/BottomRow` | 插入行 |
| `executeInsertTableLeftCol/RightCol` | 插入列 |
| `executeDeleteTableRow/Col` | 删除行/列 |
| `executeDeleteTable` | 删除表格 |
| `executeMergeTableCell` | 合并单元格 |
| `executeCancelMergeTableCell` | 取消合并 |
| `executeSplitVerticalTableCell/HorizontalTableCell` | 拆分单元格 |
| `executeTableTdVerticalAlign` | 垂直对齐 |
| `executeTableBorderType/Color` | 表格边框 |
| `executeTableTdBorderType/SlashType/BackgroundColor` | 单元格样式 |

### 5.5 搜索与替换（3 个）

| 命令 | 方法 | 说明 |
|-----|------|------|
| 搜索 | `executeSearch` | 高亮搜索结果 |
| 上一个/下一个 | `executeSearchNavigatePre/Next` | 导航 |
| 替换 | `executeReplace` | 替换文本 |

### 5.6 插入命令（12 个）

| 命令 | 说明 |
|-----|------|
| `executeImage` | 插入图片 |
| `executeHyperlink/DeleteHyperlink/CancelHyperlink/EditHyperlink` | 超链接操作 |
| `executeSeparator` | 插入分隔线 |
| `executePageBreak` | 插入分页符 |
| `executeInsertElementList` | 通用插入 |
| `executeInsertControl` | 插入控件 |
| `executeInsertTitle` | 插入标题 |
| `executeInsertArea` | 插入区域 |

### 5.7 页面设置（7 个）

| 命令 | 说明 |
|-----|------|
| `executePageMode` | 页面/连续模式 |
| `executePageScale/ScaleAdd/ScaleMinus/ScaleRecovery` | 缩放 |
| `executePaperSize/Direction` | 纸张大小/方向 |
| `executeSetPaperMargin` | 页边距 |

### 5.8 控件命令（10+ 个）

| 命令 | 说明 |
|-----|------|
| `executeSetControlValue/ValueList` | 设置控件值 |
| `executeSetControlExtension/ExtensionList` | 设置扩展数据 |
| `executeSetControlProperties/PropertiesList` | 设置控件属性 |
| `executeSetControlHighlight` | 设置高亮 |
| `executeRemoveControl` | 删除控件 |
| `executeLocationControl` | 定位到控件 |
| `executeInsertControl` | 插入控件 |
| `executeJumpControl` | 跳转到下一个控件 |

### 5.9 获取命令（20+ 个）

| 命令 | 返回值 |
|-----|-------|
| `getValue/getHTML/getText` | 文档内容 |
| `getRange/getRangeText/getRangeContext` | 选区信息 |
| `getCursorPosition` | 光标位置 |
| `getOptions/getCatalog/getWordCount` | 配置/目录/字数 |
| `getElementById/getControlValue/getControlList` | 元素/控件查询 |
| `getImage/getPaperMargin/getLocale` | 图片/页边距/语言 |
| `getAreaValue/getTitleValue/getGroupIds` | 区域/标题/分组 |

---

## 6. 典型命令实现分析：bold 加粗

`bold` 命令展示了命令实现的典型模式：

```typescript
// src/editor/core/command/CommandAdapt.ts, lines 565-597
public bold(options?: IRichtextOption) {
  // 1. 检查只读/禁用状态
  const { isIgnoreDisabledRule = false } = options || {}
  const isDisabled =
    !isIgnoreDisabledRule &&
    (this.draw.isReadonly() || this.draw.isDisabled())
  if (isDisabled) return

  // 2. 获取选区元素
  const selection = this.range.getSelectionElementList()

  // 3. 有选区时：修改选中元素的样式
  if (selection?.length) {
    // 找到第一个非粗体元素，决定是加粗还是取消加粗
    const noBoldIndex = selection.findIndex(s => !s.bold)
    selection.forEach(el => {
      el.bold = !!~noBoldIndex  // 全部设置为相同的值
    })
    // 重新渲染，但不移动光标
    this.draw.render({ isSetCursor: false })
  } else {
    // 4. 无选区时：设置默认样式（影响后续输入）
    let isSubmitHistory = true
    const { endIndex } = this.range.getRange()
    const elementList = this.draw.getElementList()
    const enterElement = elementList[endIndex]

    // 设置默认样式
    this.range.setDefaultStyle({
      bold: enterElement.bold ? false : !this.range.getDefaultStyle()?.bold
    })

    // 如果在换行符上，直接修改换行符
    if (enterElement?.value === ZERO) {
      enterElement.bold = !enterElement.bold
    } else {
      // 不在换行符上，不提交历史（只是预览样式）
      isSubmitHistory = false
    }

    // 重新渲染
    this.draw.render({
      isSubmitHistory,
      curIndex: endIndex,
      isCompute: false
    })
  }
}
```

### 6.1 核心模式总结

```
有选区时：
  1. 修改选中元素的属性
  2. render({ isSetCursor: false })

无选区时：
  1. 设置默认样式（影响后续输入）
  2. 如果在换行符上，直接修改换行符
  3. render({ isSubmitHistory, curIndex, isCompute: false })
```

### 6.2 render 参数的含义

| 参数 | 说明 |
|-----|------|
| `isSubmitHistory` | 是否提交到历史记录栈 |
| `isSetCursor` | 是否设置光标位置 |
| `isCompute` | 是否重新计算排版（文字样式不需要） |
| `curIndex` | 光标位置 |

---

## 7. 典型命令实现分析：insertElementList 插入元素

`insertElementList` 是最常用的插入命令：

```typescript
// src/editor/core/command/CommandAdapt.ts, lines 1824-1846
public insertElementList(
  payload: IElement[],
  options: IInsertElementListOption = {}
) {
  // 1. 空检查
  if (!payload.length) return

  // 2. 只读/禁用检查
  const isDisabled = this.draw.isReadonly() || this.draw.isDisabled()
  if (isDisabled) return

  // 3. 解构选项
  const { isReplace = true, ignoreContextKeys } = options

  // 4. 如果不替换选区，先收缩选区
  if (!isReplace) {
    this.range.shrinkRange()
  }

  // 5. 克隆并格式化元素
  const cloneElementList = deepClone(payload)
  const { startIndex } = this.range.getRange()
  const elementList = this.draw.getElementList()

  // 6. 格式化上下文（字体、列表、表格等）
  formatElementContext(elementList, cloneElementList, startIndex, {
    ignoreContextKeys,
    isBreakWhenWrap: true,
    editorOptions: this.options
  })

  // 7. 实际插入
  this.draw.insertElementList(cloneElementList, options)
}
```

---

## 8. 只读/禁用检查模式

几乎每个命令开头都有类似的检查：

```typescript
// 模式 1：标准检查
const isDisabled = this.draw.isReadonly() || this.draw.isDisabled()
if (isDisabled) return

// 模式 2：只读检查（部分操作允许在禁用状态）
const isReadonly = this.draw.isReadonly()
if (isReadonly) return

// 模式 3：带选项的检查（用于内部调用）
const { isIgnoreDisabledRule = false } = options || {}
const isDisabled =
  !isIgnoreDisabledRule &&
  (this.draw.isReadonly() || this.draw.isDisabled())
if (isDisabled) return
```

---

## 9. render 在命令中的使用模式

### 9.1 修改样式后（不移动光标）

```typescript
this.draw.render({ isSetCursor: false })
```

### 9.2 修改内容后（移动光标）

```typescript
this.draw.render({ curIndex: newIndex })
```

### 9.3 不重新计算排版时

```typescript
this.draw.render({
  isCompute: false,
  isSetCursor: false,
  isSubmitHistory: false
})
```

### 9.4 完整参数示例

```typescript
// src/editor/core/command/CommandAdapt.ts, lines 960-966
public insertTable(row: number, col: number) {
  const isDisabled = this.draw.isReadonly() || this.draw.isDisabled()
  if (isDisabled) return
  const activeControl = this.control.getActiveControl()
  if (activeControl) return
  // 插入表格（内部会调用 render）
  this.tableOperate.insertTable(row, col)
}
```

---

## 10. 核心要点

1. **Command 是 facade**——只做方法转发，不含任何逻辑。

2. **CommandAdapt 是实现者**——持有 Draw 等所有子系统引用，包含 140+ 个命令的实现。

3. **命令遵循统一模式**——只读检查 → 获取选区 → 修改数据 → render()。

4. **有/无选区处理不同**：
   - 有选区：修改选中元素
   - 无选区：设置默认样式（影响后续输入）

5. **render 参数决定行为**：
   - `isSubmitHistory`：是否进入撤销栈
   - `isSetCursor`：是否移动光标
   - `isCompute`：是否重新计算排版

6. **命令与历史记录配合**——只有提交到历史记录的操作才能被撤销。

7. **isIgnoreDisabledRule 允许内部调用**——绕过禁用检查，用于实现其他命令。

---

## 下一步

现在你已经理解了 Command 命令系统，下一步可以学习 **Event 事件系统**（教程 08），了解键盘输入、鼠标点击等事件是如何捕获和处理的。

**值得思考的问题**：

- 当用户按下 `Ctrl+B` 时，是哪个事件处理器调用了 `executeBold`？
- 撤销/重做是如何与 render 流程配合的？
- `executePainter`（格式刷）和 `executeApplyPainterStyle`（应用格式刷）的区别是什么？
- 为什么有些命令调用 `isCompute: false`，有些不调用？

这些问题将在后续教程中一一解答。
