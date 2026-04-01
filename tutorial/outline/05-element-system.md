# 教程 05：Element 系统 —— 文档内容的数据模型

> **前置知识**：建议先阅读教程 04（Draw 渲染引擎），理解元素如何在 Canvas 上渲染。

---

## 目录

1. [IElement 接口：文档的原子单位](#1-ielement-接口文档的原子单位)
2. [ElementType 枚举：元素的类型家族](#2-elementtype-枚举元素的类型家族)
3. [基础属性：id、value、extension](#3-基础属性idvalueextension)
4. [样式属性：文字与布局的视觉呈现](#4-样式属性文字与布局的视觉呈现)
5. [规则属性：hide 与条件渲染](#5-规则属性hide-与条件渲染)
6. [分组属性：groupIds 批注协作](#6-分组属性groupids-批注协作)
7. [复杂元素类型：表格、列表、标题等](#7-复杂元素类型表格列表标题等)
8. [元素与粒子的对应关系](#8-元素与粒子的对应关系)
9. [元素工具函数](#9-元素工具函数)
10. [核心要点](#10-核心要点)

---

## 1. IElement 接口：文档的原子单位

`IElement` 是编辑器中最核心的接口之一，它定义了文档内容的**最小数据单元**。

### 1.1 接口位置

```
src/editor/interface/Element.ts
src/editor/dataset/enum/Element.ts    ← 元素类型枚举
src/editor/dataset/enum/ElementStyle.ts ← 样式键枚举
```

### 1.2 IElement 的组合结构

`IElement` 不是单一接口，而是一个**交叉类型**（Intersection Type），由多个子接口组合而成：

```typescript
// src/editor/interface/Element.ts, lines 195-213
export type IElement = IElementBasic &
  IElementStyle &
  IElementRule &
  IElementGroup &
  ITable &
  IHyperlinkElement &
  ISuperscriptSubscript &
  ISeparator &
  IControlElement &
  ICheckboxElement &
  IRadioElement &
  ILaTexElement &
  IDateElement &
  IImageElement &
  IBlockElement &
  ITitleElement &
  IListElement &
  IAreaElement &
  ILabelElement
```

这种设计的好处是：**按功能分类的接口可以独立演进，互不干扰**。

### 1.3 子接口一览

| 子接口 | 包含属性 | 用途 |
|-------|---------|------|
| `IElementBasic` | id, type, value, extension | 基本信息 |
| `IElementStyle` | font, size, bold, color... | 视觉样式 |
| `IElementRule` | hide | 渲染规则 |
| `IElementGroup` | groupIds | 批注分组 |
| `ITable` | 表格行列信息 | 表格数据 |
| `IHyperlinkElement` | url, valueList | 超链接 |
| `ISuperscriptSubscript` | actualSize | 上标/下标 |
| `ISeparator` | dashArray, lineWidth | 分隔线 |
| `IControlElement` | control, controlId | 表单控件 |
| `ICheckboxElement` | checkbox | 复选框 |
| `IRadioElement` | radio | 单选框 |
| `ILaTexElement` | laTexSVG | LaTeX 公式 |
| `IDateElement` | dateFormat, dateId | 日期控件 |
| `IImageElement` | 图片相关属性 | 图片 |
| `IBlockElement` | block | 块级元素 |
| `ITitleElement` | valueList, level | 标题 |
| `IListElement` | valueList, listType | 列表 |
| `IAreaElement` | valueList, areaId | 区域 |
| `ILabelElement` | labelId, label | 标签 |

---

## 2. ElementType 枚举：元素的类型家族

每种内容类型对应一个 `ElementType`：

```typescript
// src/editor/dataset/enum/Element.ts
export enum ElementType {
  TEXT = 'text',           // 普通文本
  IMAGE = 'image',         // 图片
  TABLE = 'table',         // 表格
  HYPERLINK = 'hyperlink', // 超链接
  SUPERSCRIPT = 'superscript', // 上标
  SUBSCRIPT = 'subscript',    // 下标
  SEPARATOR = 'separator',    // 分隔线
  PAGE_BREAK = 'pageBreak',   // 分页符
  CONTROL = 'control',        // 表单控件（容器）
  AREA = 'area',              // 区域
  CHECKBOX = 'checkbox',      // 复选框
  RADIO = 'radio',            // 单选框
  LATEX = 'latex',            // LaTeX 公式
  TAB = 'tab',                // Tab 字符
  DATE = 'date',              // 日期控件
  BLOCK = 'block',            // 块级元素
  TITLE = 'title',            // 标题
  LIST = 'list',              // 列表
  LABEL = 'label'             // 标签
}
```

### 2.1 元素分类

这些类型可以分成三类：

```
块级元素（Block）
├── IMAGE      图片
├── TABLE      表格
├── LA TEX     LaTeX 公式
├── BLOCK      块级元素
├── TITLE      标题
├── LIST       列表
├── CHECKBOX   复选框
├── RADIO      单选框
├── DATE       日期控件
└── SEPARATOR  分隔线

内联元素（Inline）
├── TEXT       普通文本
├── HYPERLINK  超链接
├── SUPERSCRIPT 上标
├── SUBSCRIPT  下标
└── LABEL      标签

结构元素（Structure）
├── CONTROL    控件容器
├── AREA       区域
├── PAGE_BREAK 分页符
└── TAB        Tab 字符
```

---

## 3. 基础属性：id、value、extension

### 3.1 IElementBasic

```typescript
// src/editor/interface/Element.ts, lines 19-25
export interface IElementBasic {
  id?: string           // 元素唯一标识
  type?: ElementType    // 元素类型
  value: string         // 元素值（文本内容）
  extension?: unknown   // 扩展数据
  externalId?: string   // 外部关联ID
}
```

### 3.2 属性说明

| 属性 | 类型 | 说明 |
|-----|------|------|
| `id` | string | 元素的唯一标识符，用于精确定位和操作 |
| `type` | ElementType | 元素类型，决定如何渲染和处理 |
| `value` | string | 元素的核心值——文本内容、图片路径、URL 等 |
| `extension` | unknown | 扩展数据，用于存储类型特定的其他信息 |
| `externalId` | string | 关联外部系统的 ID，如数据库记录 |

### 3.3 示例

```typescript
// 文本元素
const textElement: IElement = {
  id: 'elem_001',
  type: ElementType.TEXT,
  value: 'Hello World',
  font: 'Arial',
  size: 16,
  bold: true,
  color: '#333333'
}

// 图片元素
const imageElement: IElement = {
  id: 'elem_002',
  type: ElementType.IMAGE,
  value: 'https://example.com/image.png',
  width: 300,
  height: 200
}

// 超链接元素
const linkElement: IElement = {
  id: 'elem_003',
  type: ElementType.HYPERLINK,
  value: '点击访问',
  url: 'https://example.com'
}
```

---

## 4. 样式属性：文字与布局的视觉呈现

### 4.1 IElementStyle

```typescript
// src/editor/interface/Element.ts, lines 27-42
export interface IElementStyle {
  font?: string              // 字体名称
  size?: number              // 字体大小
  width?: number             // 元素宽度
  height?: number            // 元素高度
  bold?: boolean             // 加粗
  color?: string             // 文字颜色
  highlight?: string         // 高亮背景色
  italic?: boolean           // 斜体
  underline?: boolean        // 下划线
  strikeout?: boolean        // 删除线
  rowFlex?: RowFlex          // 行对齐方式
  rowMargin?: number         // 行间距
  letterSpacing?: number     // 字间距
  textDecoration?: ITextDecoration  // 文字装饰（复杂下划线）
}
```

### 4.2 样式键枚举

`ElementStyleKey` 枚举定义了标准样式键：

```typescript
// src/editor/dataset/enum/ElementStyle.ts
export enum ElementStyleKey {
  font = 'font',
  size = 'size',
  width = 'width',
  height = 'height',
  bold = 'bold',
  color = 'color',
  highlight = 'highlight',
  italic = 'italic',
  underline = 'underline',
  strikeout = 'strikeout'
}
```

### 4.3 RowFlex 行对齐

```typescript
// 对齐方式
enum RowFlex {
  LEFT = 'left',           // 左对齐
  CENTER = 'center',       // 居中
  RIGHT = 'right',         // 右对齐
  BOTH = 'both'            // 两端对齐
}
```

### 4.4 样式示例

```typescript
// 粗体红色文本
const boldRedText: IElement = {
  value: 'Important',
  bold: true,
  color: '#FF0000'
}

// 带下划线和删除线的文本
const decoratedText: IElement = {
  value: 'Modified text',
  underline: true,
  strikeout: true,
  highlight: '#FFFF00'
}

// 右对齐文本块
const rightAligned: IElement = {
  value: 'Right aligned',
  rowFlex: RowFlex.RIGHT
}
```

---

## 5. 规则属性：hide 与条件渲染

### 5.1 IElementRule

```typescript
// src/editor/interface/Element.ts, lines 44-46
export interface IElementRule {
  hide?: boolean  // 是否隐藏
}
```

### 5.2 隐藏规则的作用

`hide` 属性控制元素是否渲染：

```
hide = false/undefined → 正常渲染
hide = true            → 不渲染，但保留在数据中
```

这与 `display: none` 类似，但作用于数据层而非渲染层。

### 5.3 应用场景

- **条件显示**：根据用户权限或上下文隐藏内容
- **草稿模式**：暂时隐藏某些元素
- **修订模式**：显示/隐藏删除的内容

---

## 6. 分组属性：groupIds 批注协作

### 6.1 IElementGroup

```typescript
// src/editor/interface/Element.ts, lines 48-50
export interface IElementGroup {
  groupIds?: string[]  // 所属分组 ID 列表
}
```

### 6.2 分组机制

一个元素可以属于多个分组（`groupIds` 是数组）：

```typescript
// 同时属于"技术文档"和"待审核"分组
const element: IElement = {
  value: 'Some content',
  groupIds: ['tech-doc', 'pending-review']
}
```

### 6.3 分组高亮

Draw 的 `group` 系统会根据 `groupIds` 渲染批注高亮：

```typescript
// src/editor/core/draw/interactive/Group.ts
// 遍历所有分组，渲染对应的高亮颜色
for (const groupId of groupIds) {
  this.renderGroupHighlight(ctx, element, groupId)
}
```

---

## 7. 复杂元素类型：表格、列表、标题等

### 7.1 表格元素 ITable

```typescript
// src/editor/interface/Element.ts, lines 67-90
export interface ITableAttr {
  colgroup?: IColgroup[]    // 列定义
  trList?: ITr[]           // 行列表
  borderType?: TableBorder // 边框类型
  borderColor?: string     // 边框颜色
  borderWidth?: number     // 边框宽度
  borderExternalWidth?: number
  translateX?: number       // 水平偏移
}

export interface ITableElement {
  tdId?: string     // 单元格 ID
  trId?: string     // 行 ID
  tableId?: string  // 表格 ID
  conceptId?: string
  pagingId?: string  // 分页拆分 ID
  pagingIndex?: number  // 分页拆分索引
}
```

### 7.2 标题元素 ITitleElement

```typescript
// src/editor/interface/Element.ts, lines 52-57
export interface ITitleElement {
  valueList?: IElement[]  // 标题内容（嵌套元素）
  level?: TitleLevel       // 标题级别（1-6）
  titleId?: string         // 标题 ID
  title?: ITitle           // 标题样式配置
}
```

标题级别枚举：

```typescript
enum TitleLevel {
  H1 = 1,
  H2 = 2,
  H3 = 3,
  H4 = 4,
  H5 = 5,
  H6 = 6
}
```

### 7.3 列表元素 IListElement

```typescript
// src/editor/interface/Element.ts, lines 59-65
export interface IListElement {
  valueList?: IElement[]    // 列表项内容
  listType?: ListType       // 列表类型
  listStyle?: ListStyle     // 列表样式
  listId?: string           // 列表 ID
  listWrap?: boolean         // 列表是否换行
}
```

列表类型和样式：

```typescript
enum ListType {
  ORDERED = 'ordered',    // 有序列表（1. 2. 3.）
  UNORDERED = 'unordered' // 无序列表（• • •）
}

enum ListStyle {
  NUMBER = 'number',           // 数字
  LOWER_LETTER = 'lowerLetter', // 小写字母
  UPPER_LETTER = 'upperLetter', // 大写字母
  LOWER_ROMAN = 'lowerRoman',   // 小写罗马数字
  UPPER_ROMAN = 'upperRoman',   // 大写罗马数字
  DISC = 'disc',               // 实心圆
  CIRCLE = 'circle',           // 空心圆
  SQUARE = 'square'             // 方块
}
```

### 7.4 超链接元素 IHyperlinkElement

```typescript
// src/editor/interface/Element.ts, lines 92-96
export interface IHyperlinkElement {
  valueList?: IElement[]   // 链接文本（支持多元素）
  url?: string             // 链接地址
  hyperlinkId?: string     // 链接 ID
}
```

### 7.5 控件元素 IControlElement

```typescript
// src/editor/interface/Element.ts, lines 107-111
export interface IControlElement {
  control?: IControl        // 控件定义
  controlId?: string        // 控件 ID
  controlComponent?: ControlComponent  // 控件组件类型
}
```

控件组件类型：

```typescript
enum ControlComponent {
  INPUT = 'input',           // 输入框
  TEXTAREA = 'textarea',      // 文本域
  SELECT = 'select',          // 下拉框
  DATE = 'date',              // 日期选择器
  DATETIME = 'datetime',      // 日期时间选择器
  TIME = 'time',              // 时间选择器
  UPLOAD = 'upload'           // 文件上传
}
```

### 7.6 图片元素 IImageElement

```typescript
// src/editor/interface/Element.ts, lines 161-172
export interface IImageBasic {
  imgDisplay?: ImageDisplay      // 显示模式
  imgFloatPosition?: {            // 浮动位置
    x: number
    y: number
    pageNo?: number
  }
  imgCrop?: IImageCrop            // 裁剪区域
  imgCaption?: IImageCaption       // 图片说明
}

export interface IImageRule {
  imgToolDisabled?: boolean       // 是否禁用图片工具
  imgPreviewDisabled?: boolean    // 是否禁用预览
}

export type IImageElement = IImageBasic & IImageRule
```

---

## 8. 元素与粒子的对应关系

元素类型决定了使用哪个 Particle 进行渲染：

```typescript
// src/editor/core/draw/Draw.ts, lines 2449-2477
private getParticleByElement(element: IElement): IParticle {
  switch (element.type) {
    case ElementType.TEXT:
    case ElementType.CONTROL:
      return this.textParticle

    case ElementType.IMAGE:
      return this.imageParticle

    case ElementType.TABLE:
      return this.tableParticle

    case ElementType.HYPERLINK:
      return this.hyperlinkParticle

    case ElementType.LATEX:
      return this.laTexParticle

    case ElementType.LIST:
      return this.listParticle

    case ElementType.SEPARATOR:
      return this.separatorParticle

    case ElementType.PAGE_BREAK:
      return this.pageBreakParticle

    case ElementType.CHECKBOX:
      return this.checkboxParticle

    case ElementType.RADIO:
      return this.radioParticle

    case ElementType.DATE:
      return this.dateParticle

    case ElementType.LABEL:
      return this.labelParticle

    case ElementType.SUPERSCRIPT:
      return this.superscriptParticle

    case ElementType.SUBSCRIPT:
      return this.subscriptParticle

    case ElementType.BLOCK:
      return this.blockParticle

    case ElementType.LINE_BREAK:
      return this.lineBreakParticle

    case ElementType.WHITE_SPACE:
      return this.whiteSpaceParticle

    default:
      return this.textParticle
  }
}
```

### 8.1 对应关系表

| ElementType | Particle | 绘制内容 |
|------------|----------|---------|
| TEXT | textParticle | 普通文本 |
| IMAGE | imageParticle | 图片 |
| TABLE | tableParticle | 表格 |
| HYPERLINK | hyperlinkParticle | 超链接 |
| LATEX | laTexParticle | LaTeX 公式 |
| LIST | listParticle | 列表标记 |
| SEPARATOR | separatorParticle | 分隔线 |
| PAGE_BREAK | pageBreakParticle | 分页符 |
| CHECKBOX | checkboxParticle | 复选框 |
| RADIO | radioParticle | 单选框 |
| DATE | dateParticle | 日期控件 |
| LABEL | labelParticle | 标签 |
| SUPERSCRIPT | superscriptParticle | 上标 |
| SUBSCRIPT | subscriptParticle | 下标 |
| BLOCK | blockParticle | 块级元素 |
| CONTROL | textParticle | 控件容器（用文本粒子渲染内容）|

---

## 9. 元素工具函数

编辑器提供了多个元素操作的工具函数。

### 9.1 元素格式化 formatElementList

将元素列表规范化，设置默认值：

```typescript
// src/editor/utils/formatElementList.ts
export function formatElementList(
  elementList: IElement[],
  defaultStyle?: IElementStyle
): IElement[]
```

主要功能：
- 填充默认值（font, size, color 等）
- 生成唯一 ID（如果没有）
- 过滤无效元素
- 应用默认样式

### 9.2 元素查找 getElementById

根据 ID 查找元素：

```typescript
// src/editor/utils/getElementById.ts
export function getElementById(
  elementList: IElement[],
  id: string
): IElement | null
```

### 9.3 元素更新 updateElementById

根据 ID 更新元素属性：

```typescript
// src/editor/utils/updateElementById.ts
export function updateElementById(
  elementList: IElement[],
  id: string,
  properties: Partial<IElement>
): boolean
```

### 9.4 元素删除 deleteElementById

根据 ID 删除元素：

```typescript
// src/editor/utils/deleteElementById.ts
export function deleteElementById(
  elementList: IElement[],
  id: string
): boolean
```

### 9.5 元素插入 insertElementList

插入元素列表到指定位置：

```typescript
// src/editor/utils/insertElementList.ts
insertElementList(
  elementList: IElement[],
  insertIndex: number,
  newElements: IElement[],
  options?: IInsertElementListOption
): void
```

选项：
```typescript
interface IInsertElementListOption {
  isReplace?: boolean        // 是否替换现有元素
  isSubmitHistory?: boolean  // 是否提交历史记录
  ignoreContextKeys?: Array<keyof IElement>  // 忽略的上下文键
}
```

---

## 10. 核心要点

1. **IElement 是交叉类型**——由多个功能接口组合而成，按职责分离。

2. **ElementType 决定渲染方式**——Draw 通过 `getParticleByElement()` 找到对应的 Particle。

3. **value 是核心数据**——不同元素类型，`value` 的含义不同（文本内容、图片路径、URL 等）。

4. **extension 提供扩展性**——用于存储类型特定但未标准化的数据。

5. **groupIds 支持批注协作**——一个元素可属于多个分组，实现复杂的协作标注。

6. **hide 控制条件渲染**——元素数据保留，但不在画布上渲染。

7. **表格和列表是复杂元素**——它们内部嵌套 `valueList`，形成树状结构。

---

## 下一步

现在你已经理解了 Element 数据模型，下一步可以学习 **Position 与 Range 系统**（教程 06），了解编辑器如何追踪光标位置和选区范围。

**值得思考的问题**：

- 表格元素内部有 `trList`（行列表），这些行元素是否也是 `IElement` 类型？
- 当用户选中文本时，Range 是如何关联到具体的 `IElement` 列表的？
- 元素的 `id` 是在什么时候生成的？如何保证唯一性？
- 列表的 `listType` 和 `listStyle` 是如何影响 `listParticle` 的渲染逻辑的？

这些问题将在后续教程中一一解答。
