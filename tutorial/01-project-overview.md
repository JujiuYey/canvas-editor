# Canvas-Editor 项目概览

> 学习日期：2026-03-27
> 学习目标：完整掌握项目架构，能独立实现并扩展

---

## 一、项目基本信息

- **包名**：`@hufe921/canvas-editor`
- **当前版本**：v0.9.129
- **技术栈**：TypeScript + Vite + Canvas API
- **构建产物**：UMD + ESM 双格式
- **许可**：MIT

这是一个基于 Canvas API 的富文本编辑器核心库，不依赖任何现成的富文本框架，所有排版、渲染、交互逻辑都是从头实现的。

---

## 二、顶层目录结构

```
canvas-editor/
├── src/                    # 源代码
│   ├── main.ts             # Demo 应用的入口
│   ├── editor/             # 【核心编辑器库】(1500+ 文件)
│   ├── plugins/            # 编辑器插件 (copy, markdown)
│   ├── components/         # Demo 页面 UI 组件
│   ├── style.css           # Demo 页面样式
│   └── mock.ts             # Demo 模拟数据
├── cypress/               # 端到端测试
│   └── e2e/                # 21 个测试文件，覆盖各种编辑器功能
├── docs/                   # VitePress 文档
├── scripts/                # 构建/发布脚本
├── package.json            # npm 配置
├── tsconfig.json           # TypeScript 配置
├── vite.config.ts          # Vite 构建配置
└── tutorial/               # 【学习文档存放目录】
```

---

## 三、核心库结构 `src/editor/`

这是项目的核心，发布到 npm 的包就是这里面的代码。

```
src/editor/
├── index.ts                # 【主入口】导出 Editor 类和所有公开类型
├── core/                   # 【核心子系统】20 个子目录 (~1500 文件)
├── interface/             # TypeScript 接口定义 (47 个文件)
├── dataset/                # 枚举 & 常量
└── utils/                  # 工具函数
```

### 核心子系统 `core/` 一览

| 子系统 | 所在目录 | 主要职责 |
|--------|---------|---------|
| **Draw** | `core/draw/` | 画布渲染引擎 (~97KB)，最核心的模块 |
| **Command** | `core/command/` | 命令模式门面，所有编辑器操作 (bold, undo 等) |
| **Event** | `core/event/` | 画布事件 & 全局事件处理 |
| **Observer** | `core/observer/` | 鼠标/滚动/选择/图片观察者 |
| **Position** | `core/position/` | 光标位置追踪 |
| **Range** | `core/range/` | 选区管理 |
| **Cursor** | `core/cursor/` | 光标渲染 |
| **History** | `core/history/` | 撤销/重做 |
| **Plugin** | `core/plugin/` | 插件系统 |
| **Override** | `core/override/` | 方法覆写/扩展机制 |
| **ContextMenu** | `core/contextmenu/` | 右键菜单 |
| **Shortcut** | `core/shortcut/` | 键盘快捷键 |
| **Zone** | `core/zone/` | 页眉/主内容/页脚分区管理 |
| **Actuator** | `core/actuator/` | 操作执行器 (含对话框/签名) |
| **Listener** | `core/listener/` | 变更回调系统 |
| **Register** | `core/register/` | 菜单和快捷键注册 |
| **Worker** | `core/worker/` | Web Workers (字数统计/目录/分组等) |
| **i18n** | `core/i18n/` | 国际化 (英语 + 中文) |

---

## 四、Draw 渲染引擎 `core/draw/`

这是整个项目最复杂、最核心的模块。

```
core/draw/
├── Draw.ts                 # 【核心】~97KB 的渲染引擎主文件
├── particle/               # 【粒子渲染器】每种元素类型有自己的渲染器
├── control/               # 表单控件渲染 (文本/数字/日期/下拉/复选/单选)
├── frame/                 # 页面装饰 (背景/页边距/页眉/页脚/水印/页码/边框)
├── richtext/              # 富文本装饰 (下划线/高亮/删除线)
├── interactive/           # 交互功能 (框选/分组/搜索)
└── graffiti/              # 自由绘图
```

### Particle 粒子渲染器详解

每个粒子 (Particle) 负责渲染一种元素类型：

**文本类粒子**：
- `TextParticle.ts` — 普通文本
- `ListParticle.ts` — 有序/无序列表
- `HyperlinkParticle.ts` — 超链接
- `WhiteSpaceParticle.ts` — 空白字符

**块级粒子**：
- `ImageParticle.ts` — 图片
- `TableParticle.ts` — 表格
- `LaTexParticle.ts` — LaTeX 数学公式
- `BlockParticle.ts` — iframe 和视频
- `CheckboxParticle.ts` — 复选框
- `RadioParticle.ts` — 单选按钮
- `DateParticle.ts` — 日期选择器
- `SeparatorParticle.ts` — 分隔线
- `PageBreakParticle.ts` — 分页符
- `LineBreakParticle.ts` — 换行符
- `Previewer.ts` — 预览
- `LabelParticle.ts` — 标签
- `SubscriptParticle.ts` / `SuperscriptParticle.ts` — 下标/上标

---

## 五、Element 元素系统

编辑器用一套统一的 `IElement` 接口来描述所有内容。

### `interface/Element.ts` — 核心接口

所有元素共享的字段：
- `id` — 唯一标识
- `type` — 元素类型 (ElementType 枚举)
- `value` — 元素值
- `extension` — 扩展数据
- `font`, `size`, `bold`, `color` 等 — 样式属性
- `hide` — 隐藏规则
- `groupIds` — 分组 ID

### ElementType 枚举包含的元素类型

文本粒子、块级粒子、控件粒子、Frame 元素等。

---

## 六、Command 命令系统

编辑器通过命令模式对外暴露 API。

- `core/command/Command.ts` (~18KB) — 命令门面，暴露 `executeBold()`、`executeUndo()` 等方法
- `core/command/CommandAdapt.ts` (~81KB) — 命令适配器，连接 Command 和 Draw 上下文

用户通过 `editor.command.executeXxx()` 调用操作。

---

## 七、入口文件 `src/editor/index.ts`

这是发布包的主入口，导出了：
- `Editor` 类 — 主类，通过 `editor.command` 驱动所有操作
- 40+ 个类型/接口 — 供消费者使用
- 30+ 个枚举 — 定义各种常量

`Editor` 类的公开 API：
- `editor.command` — 命令门面
- `editor.listener` — 变更监听器
- `editor.eventBus` — 事件总线
- `editor.override` — 方法覆写
- `editor.register` — 注册菜单/快捷键
- `editor.use(plugin)` — 安装插件
- `editor.destroy()` — 销毁实例

---

## 八、构建配置

- **Vite** 作为构建工具
- 两种构建模式：
  - `lib` 模式 — 构建 npm 包 (ESM + UMD + TypeScript 声明)
  - `app` 模式 — 构建 Demo 应用

---

## 九、学习路线建议

基于这个项目，建议按以下顺序学习：

1. **项目概览**（本节）— 了解目录结构和核心概念
2. **Editor 主入口** — 理解 Editor 类如何组织各个子系统
3. **Draw 渲染引擎** — 理解画布如何被绘制
4. **Element 元素系统** — 理解数据模型
5. **Position & Range** — 理解位置和选区
6. **Command 命令系统** — 理解操作如何执行
7. **Event 事件系统** — 理解用户交互如何处理
8. **各粒子渲染器** — 理解每种元素如何渲染
9. **History & Observer** — 理解撤销/重做和观察者模式
10. **Plugin 系统** — 理解如何扩展编辑器
11. **Override 系统** — 理解如何覆写内部行为

---

## 十、思考与练习

1. 项目的核心渲染逻辑集中在哪个文件？有多大？
2. 命令系统和渲染系统是如何分离的？为什么要这样做？
3. 每种元素类型都有对应的 Particle，它们的设计模式是什么？
4. 尝试找出文字"你好"从输入到显示在画布上的完整流程涉及哪些文件？

---

*下一节我们将深入 Editor 主入口，理解 Editor 类是如何组织和管理各个子系统的。*
