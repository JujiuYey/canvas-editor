# 02 - 主入口与 Demo 应用

> 学习日期：2026-03-27
> 学习目标：理解从页面加载到编辑器运行的完整流程，掌握 UI 与 Editor 核心之间的连接机制

---

## 一、整体架构概览

在深入代码之前，先建立宏观认知。整个 Demo 应用（`src/main.ts`）的职责可以用一句话概括：

> **将 Editor 核心库嵌入到一个完整的富文本编辑器 Demo 页面中，通过 DOM 交互驱动 Editor 命令，同时将 Editor 的状态变化回显到工具栏 UI 上。**

其数据流向如下：

```
用户点击工具栏按钮
       ↓
main.ts 中的 DOM 事件处理器
       ↓
instance.command.executeXxx()  ← Editor 核心执行操作
       ↓
Editor 内部状态变更
       ↓
触发 listener 回调（rangeStyleChange / contentChange 等）
       ↓
main.ts 中的 listener 处理器更新工具栏 UI
```

---

## 二、页面结构（HTML + CSS）

### 2.1 页面布局

`index.html` 构建了一个典型的富文本编辑器页面，共分为四个主要区域：

```
┌─────────────────────────────────────┐
│           顶部工具栏 (menu)         │  高度 60px，固定定位 z-index:9
├──────────┬──────────────────────────┤
│          │                          │
│   目录    │      Canvas 编辑器       │
│  (catalog)│      (.editor)          │
│          │                          │
│          │                          │
│          │                          │
├──────────┴──────────────────────────┤
│           批注面板 (comment)         │
├─────────────────────────────────────┤
│           底部状态栏 (footer)        │  高度约 30px，固定定位
└─────────────────────────────────────┘
```

### 2.2 各区域职责

| 区域 | CSS 类名 | 职责 |
|------|---------|------|
| 顶部工具栏 | `.menu` | 承载所有格式操作按钮 |
| 目录面板 | `.catalog` | 侧边栏，显示文档标题目录 |
| 编辑区 | `.editor` | Editor 类的挂载容器，Canvas 渲染区域 |
| 批注面板 | `.comment` | 显示文档批注列表 |
| 底部状态栏 | `.footer` | 页码、缩放比例、字数统计、光标行列信息 |

### 2.3 工具栏按钮的 HTML 结构

工具栏中的每个按钮都是一个 `.menu-item` 下的 `<div>` 元素，遵循统一的命名规范：

```html
<!-- 单击操作按钮 -->
<div class="menu-item__bold">
  <i></i>  <!-- 用 CSS 图标背景色/形状表示功能 -->
</div>

<!-- 带下拉选项的按钮 -->
<div class="menu-item__font">
  <span class="select">微软雅黑</span>   <!-- 当前选中值 -->
  <div class="options">               <!-- 下拉选项面板 -->
    <ul>
      <li data-family="Microsoft YaHei">微软雅黑</li>
      <li data-family="Arial">Arial</li>
      ...
    </ul>
  </div>
</div>
```

命名约定：`.menu-item__<功能名>`，下划线后缀表示工具栏项的语义功能名（如 `undo`、`font`、`bold`）。每个选项元素通过 `data-*` 属性存储传给命令的参数（如 `data-size="16"`、`data-family="Arial"`）。

### 2.4 CSS 关键样式

`src/style.css` 中的几个关键样式：

- **工具栏固定定位**：`position: fixed; top: 0; height: 60px; z-index: 9;`
- **下拉选项默认隐藏**：`.options` 通过 `display: none` 隐藏，点击时添加 `.visible` 类切换显示
- **滚动条美化**：统一了 Webkit 滚动条样式
- **按钮禁用态**：工具栏按钮在只读模式下通过 `.disable` 类加灰透明

---

## 三、Mock 数据结构（`src/mock.ts`）

### 3.1 导出的数据

`mock.ts` 导出了三个东西：

```typescript
export const data: IElement[]       // 文档内容（主区域的元素数组）
export const commentList: IComment[] // 批注列表
export const options: IEditorOption  // 编辑器配置选项
```

### 3.2 文档内容 data 的构建过程

`data` 是一个 `IElement[]`（元素数组），按以下步骤构建：

**步骤 1 - 基础文本解析**

```typescript
const text = `主诉：\n发热三天，咳嗽五天...`  // 原始病历文本
```

通过 `splitText`（或手动 `split('')`）将字符串拆分为单个字符的 `IElement` 数组，然后根据字符位置标记特殊属性：

- **标题识别**：在特定位置插入 `type: ElementType.TITLE` 的标题元素
- **红色高亮**：`color: '#FF0000'` 用于"传染性疾病"等关键字
- **背景高亮**：`highlight: '#F2F27F'` 用于"血细胞比容"

```typescript
// 普通字符
{ value: '主', size: 16 }
// 红色字符
{ value: '传', color: '#FF0000', size: 16 }
// 高亮字符（带批注分组ID）
{ value: '血', highlight: '#F2F27F', groupIds: ['1'], size: 16 }
// 标题元素
{ type: ElementType.TITLE, level: TitleLevel.FIRST, valueList: [...] }
```

**步骤 2 - 插入各种特殊元素**

在字符数组的特定位置（通过 `elementList.splice(index, 0, ...)`）插入各种特殊元素：

```typescript
// 文本控件（带占位符、前缀、后缀）
{ type: ElementType.CONTROL, value: '', control: { type: ControlType.TEXT, placeholder: '其他补充', prefix: '{', postfix: '}' } }

// 下拉选择控件
{ type: ElementType.CONTROL, control: { type: ControlType.SELECT, valueSets: [{ value: '有', code: '98175' }, ...] } }

// 超链接
{ type: ElementType.HYPERLINK, valueList: [...], url: 'https://...' }

// 图片
{ type: ElementType.IMAGE, value: 'data:image/png;base64,...', width: 89, height: 32 }

// 表格（带 colspan/rowspan 支持）
{ type: ElementType.TABLE, colgroup: [...], trList: [...] }

// 复选框控件
{ type: ElementType.CONTROL, control: { type: ControlType.CHECKBOX, valueSets: [...] } }

// LaTeX 公式
{ type: ElementType.LATEX, value: '{E_k} = hv - {W_0}' }

// 日期控件
{ type: ElementType.CONTROL, control: { type: ControlType.DATE, dateFormat: 'yyyy-MM-dd hh:mm:ss' } }

// 标签（诊断标签）
{ type: ElementType.LABEL, value: '高血压', labelId: 'l1' }
```

### 3.3 批注数据结构

```typescript
interface IComment {
  id: string           // 对应元素上的 groupIds
  content: string      // 批注内容
  userName: string    // 评论人
  rangeText: string   // 被批注的原文
  createdDate: string // 创建时间
}

// 示例：
{ id: '1', content: '红细胞比容（HCT）是指...', userName: 'Hufe', rangeText: '血细胞比容', createdDate: '...' }
```

### 3.4 编辑器配置 options

```typescript
export const options: IEditorOption = {
  margins: [100, 120, 100, 120],        // 页边距 [上右下左]，单位 px
  watermark: { data: 'CANVAS-EDITOR', size: 120 },  // 水印文字
  pageNumber: { format: '第{pageNo}页/共{pageCount}页' }, // 页码格式
  placeholder: { data: '请输入正文' },  // 空占位符提示
  zone: { tipDisabled: false },         // 区域提示是否禁用
  maskMargin: [60, 0, 30, 0]            // 遮盖层边距（顶部菜单60px，底部工具栏30px）
}
```

这个 `options` 是 `IEditorOption` 接口的实例，在创建 Editor 时作为第二个参数传入，用于配置编辑器的初始状态和行为。

---

## 四、Editor 实例化流程

### 4.1 创建实例

```typescript
const container = document.querySelector<HTMLDivElement>('.editor')!
const instance = new Editor(
  container,   // DOM 容器，Editor 将把 Canvas 挂载到这里
  {             // 文档初始内容
    header: [   // 页眉区域内容
      { value: '第一人民医院', size: 32, rowFlex: RowFlex.CENTER },
      { value: '\n门诊病历', size: 18, rowFlex: RowFlex.CENTER },
      { value: '\n', type: ElementType.SEPARATOR }
    ],
    main: <IElement[]>data,  // 【主区域】传入 mock 数据
    footer: [   // 页脚区域内容
      { value: 'canvas-editor', size: 12 }
    ]
  },
  options  // 编辑器配置
)
```

Editor 构造函数接受三个参数：

1. **container**：`HTMLDivElement`，Editor 会在其中创建并管理 Canvas 元素
2. **document**：包含 `header`、`main`、`footer` 三个区域的元素数组，分别对应页眉、主内容、页脚
3. **options**：编辑器配置对象（页边距、水印、页码格式等）

### 4.2 全局挂载（供外部工具访问）

```typescript
// Cypress 测试访问
Reflect.set(window, 'editor', instance)
// canvas-editor-devtools 访问
Reflect.set(window, '__CANVAS_EDITOR_INSTANCE__', instance)
```

---

## 五、工具栏按钮与命令的连接机制

### 5.1 核心连接模式

每个工具栏按钮的连接模式完全一致：

```typescript
// 1. 通过 class 找到 DOM 元素
const boldDom = document.querySelector<HTMLDivElement>('.menu-item__bold')!

// 2. 给按钮添加 title 属性显示快捷键
boldDom.title = `加粗(${isApple ? '⌘' : 'Ctrl'}+B)`

// 3. 给按钮绑定 onclick
boldDom.onclick = function () {
  instance.command.executeBold()  // ← 命令调用
}
```

这就是整个连接机制的核心：**DOM 事件 -> Editor Command 调用**。

### 5.2 不同类型按钮的处理

#### 5.2.1 单参数命令

```typescript
// 字体
instance.command.executeFont('Microsoft YaHei')

// 字号
instance.command.executeSize(16)

// 文字颜色
instance.command.executeColor('#FF0000')

// 行对齐
instance.command.executeRowFlex(RowFlex.LEFT)
instance.command.executeRowFlex(RowFlex.CENTER)

// 标题级别
instance.command.executeTitle(TitleLevel.FIRST)
```

参数来源是 HTML 中的 `data-*` 属性：

```typescript
fontOptionDom.onclick = function (evt) {
  const li = evt.target as HTMLLIElement
  instance.command.executeFont(li.dataset.family!)  // ← 从 data-family 读取
}
```

#### 5.2.2 复杂参数命令（通过 Dialog 弹窗收集）

对于需要多个输入项的操作（如插入图片、表格、超链接），使用 `Dialog` 组件先弹出配置框，用户确认后再执行命令：

```typescript
// 超链接插入
new Dialog({
  title: '超链接',
  data: [
    { type: 'text', label: '文本', name: 'name', required: true, value: instance.command.getRangeText() },
    { type: 'text', label: '链接', name: 'url', required: true }
  ],
  onConfirm: payload => {
    const name = payload.find(p => p.name === 'name')?.value
    const url = payload.find(p => p.name === 'url')?.value
    instance.command.executeHyperlink({
      url,
      valueList: splitText(name).map(n => ({ value: n, size: 16 }))
    })
  }
})
```

#### 5.2.3 带即时 UI 反馈的命令

例如表格插入，用户在面板上 hover 时就要实时预览：

```typescript
// 表格插入面板 - 鼠标移动时动态计算行列数
tablePanel.onmousemove = function (evt) {
  const celSize = 16
  const rowMarginTop = 10
  const celMarginRight = 6
  colIndex = Math.ceil(offsetX / (celSize + celMarginRight)) || 1
  rowIndex = Math.ceil(offsetY / (celSize + rowMarginTop)) || 1
  // 动态高亮预览格
  tableCellList.forEach((tr, trIndex) => {
    tr.forEach((td, tdIndex) => {
      if (tdIndex < colIndex && trIndex < rowIndex) {
        td.classList.add('active')
      }
    })
  })
  setTableTitle(`${rowIndex}×${colIndex}`)  // 显示"3×2"这样的预览标题
}

// 点击时真正插入
tablePanel.onclick = function () {
  instance.command.executeInsertTable(rowIndex, colIndex)
  recoveryTable()  // 恢复面板初始状态
}
```

#### 5.2.4 特殊命令

**格式刷**（Painter）：通过 200ms 双击计时区分单次使用和连续使用：

```typescript
let isFirstClick = true
let painterTimeout: number
painterDom.onclick = function () {
  if (isFirstClick) {
    isFirstClick = false
    painterTimeout = window.setTimeout(() => {
      instance.command.executePainter({ isDblclick: false })  // 单击
      isFirstClick = true
    }, 200)
  } else {
    window.clearTimeout(painterTimeout)
  }
}
painterDom.ondblclick = function () {
  instance.command.executePainter({ isDblclick: true })  // 双击连续使用
  isFirstClick = true
}
```

---

## 六、事件监听系统（Listener）

这是最关键的双向通信机制。Editor 内部状态变化后，通过 `listener` 回调通知外部（main.ts），由 main.ts 更新工具栏 UI。

### 6.1 Listener 系统概览

```typescript
// main.ts 中注册监听器
instance.listener.rangeStyleChange = function(payload) { ... }
instance.listener.contentChange = function(payload) { ... }
instance.listener.saved = function(payload) { ... }
```

`listener` 对象上的每个属性都是一个回调函数，当对应事件发生时由 Editor 内部调用。

### 6.2 rangeStyleChange — 选区格式变化

这是最重要的监听器。当用户在编辑器中移动光标或选中文本时触发，payload 包含当前选区的完整格式信息。

**UI 更新流程**：

```typescript
instance.listener.rangeStyleChange = function (payload) {
  // 1. 字体样式：bold/italic/underline/strikeout
  payload.bold ? boldDom.classList.add('active') : boldDom.classList.remove('active')
  payload.italic ? italicDom.classList.add('active') : italicDom.classList.remove('active')
  payload.underline ? underlineDom.classList.add('active') : underlineDom.classList.remove('active')
  payload.strikeout ? strikeoutDom.classList.add('active') : strikeoutDom.classList.remove('active')

  // 2. 字体和字号：更新下拉框显示
  fontSelectDom.innerText = curFontDom.innerText
  sizeSelectDom.innerText = `${payload.size}`

  // 3. 颜色和高亮：更新颜色选择器
  if (payload.color) {
    colorDom.classList.add('active')
    colorControlDom.value = payload.color
    colorSpanDom.style.backgroundColor = payload.color
  }

  // 4. 上下标
  payload.type === ElementType.SUBSCRIPT ? subscriptDom.classList.add('active') : ...
  payload.type === ElementType.SUPERSCRIPT ? superscriptDom.classList.add('active') : ...

  // 5. 行对齐方式
  if (payload.rowFlex === 'right') rightDom.classList.add('active')
  else if (payload.rowFlex === 'center') centerDom.classList.add('active')
  else leftDom.classList.add('active')

  // 6. 行间距
  const curRowMarginDom = rowOptionDom.querySelector<HTMLLIElement>(
    `[data-rowmargin='${payload.rowMargin}']`
  )!
  curRowMarginDom.classList.add('active')

  // 7. 标题级别
  if (payload.level) { titleSelectDom.innerText = curTitleDom.innerText }
  else { titleSelectDom.innerText = '正文' }

  // 8. 撤销/重做按钮可用性
  payload.undo ? undoDom.classList.remove('no-allow') : undoDom.classList.add('no-allow')
  payload.redo ? redoDom.classList.remove('no-allow') : redoDom.classList.add('no-allow')

  // 9. 批注高亮
  if (payload.groupIds) {
    const [id] = payload.groupIds
    activeCommentDom.classList.add('active')
    scrollIntoView(commentDom, activeCommentDom)
  }

  // 10. 光标行列位置
  const rangeContext = instance.command.getRangeContext()
  document.querySelector<HTMLSpanElement>('.row-no')!.innerText = `${rangeContext.startRowNo + 1}`
  document.querySelector<HTMLSpanElement>('.col-no')!.innerText = `${rangeContext.startColNo + 1}`
}
```

**这就是"回显"机制**：用户选中加粗文字，工具栏的加粗按钮自动高亮；用户取消加粗，按钮恢复原状。

### 6.3 contentChange — 内容变化

文档内容发生任何变化时触发（添加/删除文字、插入图片等），使用 `debounce` 防抖（200ms）：

```typescript
const handleContentChange = async function () {
  // 1. 更新字数统计
  const wordCount = await instance.command.getWordCount()
  document.querySelector<HTMLSpanElement>('.word-count')!.innerText = `${wordCount || 0}`

  // 2. 如果目录可见，更新目录
  if (isCatalogShow) {
    nextTick(() => { updateCatalog() })  // nextTick 等 DOM 更新周期结束
  }

  // 3. 更新批注面板
  nextTick(() => { updateComment() })
}
instance.listener.contentChange = debounce(handleContentChange, 200)
handleContentChange()  // 初始化时立即执行一次
```

注意 `async/await` + `getWordCount()`，因为字数统计是通过 Web Worker 异步计算的。

### 6.4 其他监听器

| 监听器名 | 触发时机 | UI 更新 |
|---------|---------|--------|
| `visiblePageNoListChange` | 可见页码列表变化 | 更新底部可见页码列表 |
| `pageSizeChange` | 总页数变化 | 更新底部页码 `1/N` |
| `intersectionPageNoChange` | 当前可见页码变化 | 更新底部当前页码 |
| `pageScaleChange` | 缩放比例变化 | 更新缩放百分比 |
| `controlChange` | 控件激活状态变化 | 禁用/启用某些工具栏按钮 |
| `pageModeChange` | 页面模式切换 | 更新分页/连页选中态 |
| `saved` | 文档保存时 | 打印日志 |

---

## 七、完整数据流分析

### 7.1 场景：用户点击"加粗"按钮

```
1. 用户点击 boldDom (.menu-item__bold)
          ↓
2. main.ts 的 boldDom.onclick 触发
          ↓
3. instance.command.executeBold() 被调用
          ↓
4. Editor 核心执行加粗逻辑：
   - 修改选区元素的 bold 属性
   - 触发 Draw 重绘
   - 触发 history 记录
          ↓
5. Editor 触发 listener.rangeStyleChange({ bold: true, ... })
          ↓
6. main.ts 的 rangeStyleChange 回调执行：
   - boldDom.classList.add('active')  ← 工具栏按钮高亮
   - 其他格式属性也被更新
          ↓
7. Canvas 重绘，加粗文字在画布上以粗体显示
```

### 7.2 场景：用户在编辑器中选择了一段文字

```
1. 用户在 Canvas 上拖动选中文本
          ↓
2. Canvas 事件系统捕获 mouseup，计算选区范围
          ↓
3. RangeManager 更新选区上下文（startElement、endElement、startColNo、endColNo）
          ↓
4. Draw 重绘选区（显示蓝色高亮背景）
          ↓
5. Editor 触发 listener.rangeStyleChange(payload)
          ↓
6. main.ts 读取 payload 中的格式信息：
   - payload.bold = true? → 加粗按钮高亮
   - payload.font = 'Arial'? → 字体下拉框显示 Arial
   - payload.size = 16? → 字号下拉框显示 小四
   - payload.undo = true? → 撤销按钮可用
```

### 7.3 场景：用户在编辑器中输入文字

```
1. 用户按下键盘
          ↓
2. GlobalEvent 捕获 keydown
          ↓
3. 触发对应的快捷键或输入处理
          ↓
4. TextParticle 处理文字输入：
   - 在当前位置插入新的 IElement
   - 更新后续元素的位置索引
          ↓
5. Draw 重绘（增量更新）
          ↓
6. listener.contentChange 触发
          ↓
7. main.ts 更新：
   - 字数统计（异步 Web Worker）
   - 目录（如有必要）
   - 批注面板
```

---

## 八、右键菜单和快捷键注册

### 8.1 右键菜单注册

通过 `instance.register.contextMenuList()` 注册右键菜单项：

```typescript
instance.register.contextMenuList([
  {
    name: '批注',
    when: payload => { ... },   // 条件函数，返回何时显示此菜单
    callback: (command: Command) => { ... }  // 点击后执行的命令
  },
  {
    name: '新增题注',
    when: payload => payload.startElement?.type === ElementType.IMAGE && !payload.startElement?.imgCaption,
    callback: (command: Command) => { command.executeSetImageCaption({ value }) }
  }
])
```

`when` 条件函数接收的 `payload` 包含丰富的上下文信息：`isReadonly`、`editorHasSelection`、`zone`、`startElement`、`options` 等。

### 8.2 快捷键注册

```typescript
instance.register.shortcutList([
  {
    key: KeyMap.P,
    mod: true,       // ⌘(Mac) 或 Ctrl(Windows)
    isGlobal: true,  // 是否全局生效
    callback: (command: Command) => { command.executePrint() }
  },
  {
    key: KeyMap.F,
    mod: true,
    isGlobal: true,
    callback: (command: Command) => { ... }  // 搜索框弹出
  }
])
```

全局快捷键在整个页面生效（即使焦点不在编辑器上），非全局的快捷键只在编辑器有焦点时生效。

---

## 九、底部状态栏

底部状态栏展示编辑器运行时的各种实时信息：

| 元素 | 类名 | 数据来源 |
|------|------|---------|
| 可见页码列表 | `.page-no-list` | `visiblePageNoListChange` listener |
| 当前页/总页数 | `.page-no` / `.page-size` | `intersectionPageNoChange` / `pageSizeChange` |
| 字数 | `.word-count` | `contentChange` 中的 `getWordCount()` |
| 光标行号 | `.row-no` | `rangeStyleChange` 中的 `getRangeContext()` |
| 光标列号 | `.col-no` | 同上 |
| 缩放比例 | `.page-scale-percentage` | `pageScaleChange` listener |

---

## 十、关键设计模式总结

### 10.1 命令模式（Command Pattern）

所有编辑器操作都通过 `instance.command.executeXxx()` 暴露，这是一种**命令模式**：
- 每个操作封装为一个命令对象
- 命令的接收者（Editor 核心）对调用者（main.ts）透明
- 便于实现撤销/重做（History）


### 10.2 观察者模式（Observer Pattern）

`listener` 系统是典型的**观察者模式**：
- Editor 是被观察者（Subject）
- main.ts 中的各个回调函数是观察者（Observer）
- Editor 状态变化时主动推送通知，观察者负责更新 UI

### 10.3 适配器模式

`command` 门面是对 Editor 内部 Draw 上下文的**适配器**：
- 外部消费者只看到简单的 `executeXxx()` 接口
- 内部实现处理了复杂的 Draw 上下文访问
- 隔离了内部实现细节

---

## 思考与练习

1. **流程追踪**：试着追踪从用户按下 `Ctrl+B` 到工具栏按钮高亮的完整调用链，涉及哪些文件？

2. **添加新功能**：如果要添加一个"首行缩进"按钮，需要修改哪几个文件？分别在哪些步骤中修改？

3. **Listener 机制**：如果 `rangeStyleChange` 不触发 `boldDom.classList.add('active')`，会有什么 UX 问题？

4. **Dialog 模式**：观察代码中 Dialog 弹窗的使用，找出哪些操作使用了弹窗，哪些没有。为什么有些操作不需要弹窗？

5. **异步处理**：为什么 `getWordCount()` 需要 `await`？这意味着字数统计是在哪里执行的？

---

*下一节我们将深入 Editor 主入口（`src/editor/index.ts`），理解 Editor 类本身是如何组织和管理各个子系统的。*
