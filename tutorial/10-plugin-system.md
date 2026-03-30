# 教程 10：Plugin 插件系统 —— 功能的可扩展设计

> **前置知识**：建议先阅读教程 04（Draw 渲染引擎）和教程 07（Command 命令系统），理解编辑器的核心架构和命令接口。

---

## 目录

1. [概述：插件系统的设计哲学](#1-概述插件系统的设计哲学)
2. [核心类型：PluginFunction 与 Plugin 类](#2-核心类型pluginfunction-与-plugin-类)
3. [Editor 与插件的集成](#3-editor-与插件的集成)
4. [插件开发实战：版权复制插件](#4-插件开发实战版权复制插件)
5. [插件开发实战：Markdown 解析插件](#5-插件开发实战markdown-解析插件)
6. [插件的典型使用场景](#6-插件的典型使用场景)
7. [核心要点](#7-核心要点)

---

## 1. 概述：插件系统的设计哲学

插件系统是编辑器最简洁却最强大的扩展机制。它允许你在不修改核心代码的情况下，为编辑器添加新功能。

### 1.1 设计理念

这个编辑器的插件系统极度精简：

```typescript
// 插件就是一个函数
type PluginFunction<Options> = (editor: Editor, options?: Options) => void
```

没有复杂的生命周期、没有虚基类、没有依赖注入——就是一个普通的函数。

### 1.2 插件能做什么

插件可以：

- **扩展 Command**：添加新的命令方法
- **订阅 EventBus**：监听编辑器内部事件
- **覆盖现有方法**：修改默认行为
- **添加 DOM 元素**：插入自定义 UI

### 1.3 核心文件

| 文件 | 职责 |
|-----|------|
| `src/editor/interface/Plugin.ts` | `PluginFunction` 类型定义 |
| `src/editor/core/plugin/Plugin.ts` | `Plugin` 类，管理插件注册 |
| `src/editor/index.ts` | 编辑器入口，暴露 `use()` 方法 |
| `src/plugins/copy/index.ts` | 版权复制插件示例 |
| `src/plugins/markdown/index.ts` | Markdown 解析插件示例 |

---

## 2. 核心类型：PluginFunction 与 Plugin 类

### 2.1 PluginFunction 类型

```typescript
// src/editor/interface/Plugin.ts
export type PluginFunction<Options> = (editor: Editor, options?: Options) => void
```

这是一个简单的函数类型：
- 接收 `Editor` 实例作为第一个参数
- 可选的 `Options` 泛型参数用于配置
- 返回 `void`（即没有返回值）

### 2.2 Plugin 类

```typescript
// src/editor/core/plugin/Plugin.ts
export class Plugin {
  private editor: Editor

  constructor(editor: Editor) {
    this.editor = editor
  }

  public use<Options>(
    pluginFunction: PluginFunction<Options>,
    options?: Options
  ) {
    pluginFunction(this.editor, options)
  }
}
```

`Plugin` 类的唯一职责是：**将 `use()` 调用转发给插件函数**。

---

## 3. Editor 与插件的集成

### 3.1 编辑器暴露 use 方法

```typescript
// src/editor/index.ts
export class Editor {
  constructor() {
    // ...
    const plugin = new Plugin(this)
    this.use = plugin.use.bind(plugin)
  }

  public use: <Options>(
    pluginFunction: PluginFunction<Options>,
    options?: Options
  ) => void
}
```

用户通过 `editor.use(myPlugin)` 调用插件。

### 3.2 使用流程图

```
用户调用 editor.use(plugin)
    ↓
Editor.use → Plugin.use(pluginFunction)
    ↓
pluginFunction(editor, options)
    ↓
插件内部：editor.command.xxx = ...
           editor.eventBus.subscribe(...)
```

---

## 4. 插件开发实战：版权复制插件

版权复制插件是最简单的插件示例：在复制内容时自动附加版权信息。

### 4.1 完整代码

```typescript
// src/plugins/copy/index.ts
import { Editor } from '../../editor'
import { I18n } from '../../editor/core/i18n'

interface CopyPluginOptions {
  copyrightText: string  // 版权文字
}

export const copyPlugin = (
  editor: Editor,
  options?: CopyPluginOptions
): void => {
  // 默认版权文字
  const defaultCopyright = '来源：xxx'

  // 保存原始的 copy 命令
  const originalCopy = editor.command.executeCopy

  // 重写 copy 命令
  editor.command.executeCopy = (...args: any[]) => {
    // 调用原始 copy 获取选区内容
    const result = originalCopy.apply(editor.command, args)

    // 附加版权信息
    const copyright = options?.copyrightText || defaultCopyright
    const i18n = new I18n()

    // 将版权信息追加到剪贴板
    const clipboardData = window.localStorage.getItem('clipboardData')
    if (clipboardData) {
      const newClipboardData = clipboardData + '\n\n' + i18n.i18n(copyright)
      window.localStorage.setItem('clipboardData', newClipboardData)
    }

    return result
  }
}
```

### 4.2 使用方式

```typescript
// 使用默认配置
editor.use(copyPlugin)

// 自定义版权文字
editor.use(copyPlugin, {
  copyrightText: '© 2024 My Company'
})
```

### 4.3 核心模式

1. **保存原始方法**：`const originalCopy = editor.command.executeCopy`
2. **替换为新方法**：直接赋值覆盖
3. **调用原始方法**：使用 `apply` 或 `call` 转发调用

---

## 5. 插件开发实战：Markdown 解析插件

Markdown 插件展示了另一种模式：**添加全新的命令**。

### 5.1 完整代码

```typescript
// src/plugins/markdown/index.ts
import { Editor } from '../../editor'
import { formatElementList } from '../../editor/utils/formatElementList'
import { IElement } from '../../editor/interface/Element'

interface MarkdownPluginOptions {
  splitChar?: string  // 分隔符，默认 \n
}

export const markdownPlugin = (
  editor: Editor,
  options?: MarkdownPluginOptions
): void => {
  const splitChar = options?.splitChar || '\n'

  // 添加 executeMarkdown 命令
  editor.command.executeMarkdown = (markdownText: string) => {
    // 将 Markdown 文本转换为 IElement 数组
    const elementList: IElement[] = []

    // 按行分割
    const lines = markdownText.split(splitChar)

    for (const line of lines) {
      if (line.startsWith('# ')) {
        // 标题
        const titleElement: IElement = {
          type: 'title',
          value: line.substring(2),
          level: 1
        }
        elementList.push(titleElement)
      } else if (line.startsWith('## ')) {
        // 二级标题
        const titleElement: IElement = {
          type: 'title',
          value: line.substring(3),
          level: 2
        }
        elementList.push(titleElement)
      } else if (line.startsWith('- ')) {
        // 无序列表
        const listElement: IElement = {
          type: 'list',
          value: line.substring(2),
          listType: 'unordered'
        }
        elementList.push(listElement)
      } else {
        // 普通文本
        const textElement: IElement = {
          type: 'text',
          value: line
        }
        elementList.push(textElement)
      }
    }

    // 格式化元素列表
    const formattedElements = formatElementList(elementList)

    // 插入到编辑器
    editor.command.insertElementList(formattedElements)
  }
}
```

### 5.2 使用方式

```typescript
// 注册插件
editor.use(markdownPlugin)

// 调用新命令
editor.command.executeMarkdown(`
# 我的文档标题

这是第一段话。

## 列表

- 项目一
- 项目二
`)
```

### 5.3 核心模式

1. **直接添加属性**：`editor.command.executeMarkdown = ...`
2. **调用现有命令**：`editor.command.insertElementList(...)`
3. **使用编辑器内部工具**：如 `formatElementList()`

---

## 6. 插件的典型使用场景

### 6.1 扩展命令

```typescript
const myPlugin = (editor: Editor) => {
  editor.command.executeMyCommand = (params) => {
    // 实现新功能
  }
}
```

### 6.2 监听事件

```typescript
const eventPlugin = (editor: Editor) => {
  editor.eventBus.subscribe('onSelectionChange', (range) => {
    console.log('选区变化:', range)
  })
}
```

### 6.3 覆盖现有方法

```typescript
const overridePlugin = (editor: Editor) => {
  const original = editor.command.executeBold

  editor.command.executeBold = (...args) => {
    console.log('bold 被调用')
    return original.apply(editor.command, args)
  }
}
```

### 6.4 添加自定义 UI

```typescript
const uiPlugin = (editor: Editor) => {
  const container = editor.getContainer()

  const button = document.createElement('button')
  button.textContent = '自定义按钮'
  button.onclick = () => {
    editor.command.executeMyCommand()
  }

  container.appendChild(button)
}
```

---

## 7. 核心要点

1. **插件就是一个函数**——`PluginFunction<Options>` 类型简洁到极致。

2. **插件通过修改 editor 对象扩展功能**——添加 `command` 方法、订阅事件、覆盖现有行为。

3. **保存原始方法是覆盖的关键**——使用 `const original = editor.command.xxx` 保存，再通过 `apply/call` 调用。

4. **插件没有隔离机制**——所有插件共享同一个 `Editor` 实例，小心命名冲突。

5. **`options` 泛型提供配置能力**——通过泛型参数让插件支持可选配置。

6. **插件加载顺序很重要**——后加载的插件会覆盖先加载的同名方法。

7. **推荐使用 `apply` 而非直接调用**——`original.apply(editor.command, args)` 确保 `this` 上下文正确。

---

## 下一步

现在你已经理解了插件系统，整个编辑器架构的核心模块已经全部覆盖：

- 教程 04：Draw 渲染引擎
- 教程 05：Element 数据模型
- 教程 06：Position 与 Range 位置系统
- 教程 07：Command 命令系统
- 教程 08：Event 事件系统
- 教程 09：Particle 粒子系统
- 教程 10：Plugin 插件系统

**值得思考的问题**：

- 如果两个插件都定义了 `executeMyCommand`，会发生什么？
- 插件如何访问 Draw 的私有成员（如 `draw.getElementList()`）？
- 为什么版权复制插件要使用 `window.localStorage` 而不是剪贴板 API？
- 如何让插件支持禁用（类似 Command 的 `isIgnoreDisabledRule`）？

这些问题的答案需要进一步阅读源码或等待后续深入教程。
