# Canvas Editor 渐进式开发教程

> 本教程从零开始，手把手教你构建一个基于 Canvas 的富文本编辑器。

---

## 教程大纲

### 第一阶段：基础概念（第1-3章）
理解最小可运行编辑器的核心概念

| 章节 | 内容 | 目标 |
|------|------|------|
| [第1章：最小可运行编辑器](./PROGRESSIVE-01-minimal-editor.md) | 创建 Canvas、绑定事件、基本输入 | 运行起来，看到文字 |
| [第2章：元素与数据模型](./PROGRESSIVE-02-element-model.md) | 设计 Element 数据结构、文本数组 | 理解"数据驱动视图" |
| [第3章：排版初探](./PROGRESSIVE-03-typesetting.md) | 文本换行、行高计算 | 文字能自动换行 |

### 第二阶段：核心功能（第4-7章）
实现编辑器的基础编辑功能

| 章节 | 内容 | 目标 |
|------|------|------|
| [第4章：光标与选区](./PROGRESSIVE-04-cursor-selection.md) | 光标位置、选区管理 | 能选中文字 |
| [第5章：样式系统](./PROGRESSIVE-05-style-system.md) | 字体、颜色、加粗、斜体 | 支持富文本样式 |
| [第6章：撤销与重做](./PROGRESSIVE-06-undo-redo.md) | 历史记录栈 | 能 Ctrl+Z 撤销 |
| [第7章：命令系统](./PROGRESSIVE-07-command-system.md) | 命令模式封装 | 清晰的 API 接口 |

### 第三阶段：高级功能（第8-10章）
实现复杂编辑功能

| 章节 | 内容 | 目标 |
|------|------|------|
| [第8章：图片与浮动元素](./PROGRESSIVE-08-image-float.md) | 图片插入、位置浮动 | 支持图片 |
| [第9章：表格基础](./PROGRESSIVE-09-table-basic.md) | 表格绘制、单元格 | 支持表格 |
| [第10章：分页系统](./PROGRESSIVE-10-pagination.md) | 多页、分页符、页码 | 支持分页 |

### 第四阶段：性能与优化（第11-12章）
让编辑器在大文档下流畅运行

| 章节 | 内容 | 目标 |
|------|------|------|
| [第11章：增量渲染](./PROGRESSIVE-11-incremental-render.md) | 只重绘变化区域 | 输入不卡顿 |
| [第12章：懒加载与 Worker](./PROGRESSIVE-12-lazy-worker.md) | IntersectionObserver、Web Worker | 大文档优化 |

---

## 学习路径建议

```
第1章 → 第2章 → 第3章 → 第4章 → 第5章 → 第6章 → 第7章
  ↓        ↓        ↓        ↓        ↓        ↓        ↓
最小运行  数据模型   排版     光标     样式     撤销     命令
                                                    (← 达到初级可用水准)
  ↓                                                   ↓
第8章 → 第9章 → 第10章 → 第11章 → 第12章
  ↓        ↓        ↓        ↓        ↓
图片     表格     分页     增量     Worker
                                    (← 达到生产可用水准)
```

---

## 代码结构

每个章节都有对应的代码文件：

```
progressive-tutorial/
├── chapter-01/           # 第1章：最小可运行编辑器
│   ├── index.html
│   ├── editor.ts
│   └── main.ts
├── chapter-02/           # 第2章：元素与数据模型
│   └── ...
└── ...
```

---

## 开始之前

你需要具备：
- JavaScript/TypeScript 基础
- HTML5 Canvas 基础（会 basic 绘图即可）
- 了解 DOM 事件处理

你不需要：
- 了解设计模式（教程中会解释）
- 了解富文本编辑器（这是我们要学的）

---

## 准备好了吗？

[开始第1章：最小可运行编辑器](./PROGRESSIVE-01-minimal-editor.md)
