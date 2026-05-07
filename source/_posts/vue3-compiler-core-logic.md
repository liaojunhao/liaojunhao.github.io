---
title: Vue3 编译器核心逻辑：从 template 到 render 函数
date: 2026-05-07 10:30:00
categories:
  - JavaScript 系列
tags:
  - JavaScript
  - Vue
  - 前端
---

## 前言

Vue 的编译器（Compiler）是一个非常有趣却又常被忽视的模块。我们在 `.vue` 文件里写的 `<template>`，最终是怎么变成页面上的 DOM 的？这背后就是编译器的功劳。

本文基于 Vue 3.2 最新源码，剖析 Vue 编译器从 **template → render 函数** 的完整流程，揭示 AST、Transform、CodeGen 的核心原理。

---

## 一、什么是 DSL 编译器

### 1.1 通用编译器 vs DSL 编译器

编译器本质上是一种"翻译器"：**把 A 语言翻译成 B 语言**。

构建一个 Java 或 JavaScript 通用编译器是非常复杂的，涉及词法分析、语法分析、优化、目标代码生成等环节。

但 Vue 需要的不是通用编译器，而是一个**领域特定语言（DSL）编译器**：

> **把 template 模板，编译成 render 函数**

这个任务的复杂度比通用编译器低得多，但已经足够支撑 Vue 模板系统的核心能力。

### 1.2 编译器入口

在 Vue 源码 `packages/compiler-dom/src/index.ts` 中，`compile` 方法最终调用 `baseCompile`：

```typescript
// packages/compiler-core/src/compile.ts
export function baseCompile(
  template: string | RootNode,
  options: CompilerOptions = {}
): CodegenResult {
  // 1. parse：将模板字符串解析为 AST
  const ast = isString(template) ? baseParse(template, options) : template

  // 2. transform：将 AST 转换为 JavaScript AST
  transform(ast, extend({}, options, {
    prefixIdentifiers,
    nodeTransforms: [...nodeTransforms, ...(options.nodeTransforms || [])],
    directiveTransforms: extend({}, directiveTransforms, options.directiveTransforms || {})
  }))

  // 3. generate：根据 JavaScript AST 生成 render 函数
  return generate(ast, extend({}, options, { prefixIdentifiers }))
}
```

编译器只做三件事：**Parse → Transform → Generate**，简洁而优雅。

---

## 二、Parse：模板解析为 AST

### 2.1 什么是 AST

AST（Abstract Syntax Tree，抽象语法树）是一个用来**描述模板结构的 JavaScript 对象**。

以如下模板为例：

```html
<div v-if="isShow">
  <p class="title">hello world</p>
</div>
```

解析后生成的 AST 核心结构如下：

```javascript
{
  type: 0, // NodeTypes.ROOT — 根节点
  children: [
    {
      type: 1, // NodeTypes.ELEMENT — 元素节点
      tag: 'div',
      tagType: 0,
      props: [
        {
          type: 7, // NodeTypes.DIRECTIVE — 指令节点
          name: 'if',
          exp: {
            type: 4, // NodeTypes.SIMPLE_EXPRESSION — 简单表达式
            content: 'isShow',
            isStatic: false
          }
        }
      ],
      children: [
        {
          type: 1, // 嵌套的 p 元素
          tag: 'p',
          props: [
            {
              type: 6, // NodeTypes.ATTRIBUTE — 属性节点
              name: 'class',
              value: { type: 2, content: 'title' }
            }
          ],
          children: [
            { type: 2, content: 'hello world' } // 文本节点
          ]
        }
      ]
    }
  ]
}
```

### 2.2 AST 节点类型一览

Vue 编译器定义了以下几种核心节点类型：

| type | 名称 | 说明 |
|------|------|------|
| 0 | ROOT | 根节点 |
| 1 | ELEMENT | DOM 元素节点 |
| 2 | TEXT | 文本节点 |
| 6 | ATTRIBUTE | HTML 属性（如 `class`、`id`） |
| 7 | DIRECTIVE | 指令（如 `v-if`、`v-for`） |
| 4 | SIMPLE_EXPRESSION | 简单表达式 |

指令节点（type: 7）下的表达式类型更丰富，包括：
- `SIMPLE_EXPRESSION` — 简单表达式
- `JS_CALL_EXPRESSION` — JS 调用表达式
- `JS_OBJECT_EXPRESSION` — JS 对象表达式
- `JS_CONDITIONAL_EXPRESSION` — 条件表达式
- `JS_ARRAY_EXPRESSION` — 数组表达式
- `JS_FUNCTION_EXPRESSION` — 函数表达式

### 2.3 AST 的核心价值

AST 本质上只是一个**结构化的 JS 对象**，它做到了：

- 用 `type` 区分节点类型（元素、文本、指令……）
- 用 `children` 递归表达 DOM 树结构
- 用 `loc` 记录每个节点在源码中的位置（行、列、偏移量）
- 将 `v-if`、`v-for` 等指令也纳入 AST 统一管理

这样一来，模板的**所有信息都被结构化**了，后续处理只需要遍历和操作这个对象即可。

---

## 三、Transform：AST 转化为 JavaScript AST

### 3.1 为什么要转换？

AST 描述的是模板的结构，但还**不能直接用来生成 render 函数**。因为 render 函数本质上是 JavaScript 代码，不是 HTML 结构。

所以需要第二步：**将模板 AST 转换为 JavaScript AST**。

### 3.2 核心产物：codegenNode

转换后最关键的产物是 `codegenNode`（代码生成节点），它是 render 函数的具体"配方"。

还是上面的例子，转换后在根节点和元素节点上都多了一个 `codegenNode` 属性：

```javascript
{
  type: 0,
  children: [...],
  codegenNode: {
    type: 13,              // NodeTypes.VNODE_CALL
    tag: '"div"',          // 标签名（字符串字面量）
    children: {            // 子节点
      type: 2,             // NodeTypes.TEXT
      content: 'hello world'
    },
    isBlock: true,         // 是否为块级元素
    isComponent: false,    // 是否为组件
    disableTracking: false
  }
}
```

`codegenNode` 告诉代码生成器：当前节点要调用 `createElementBlock("div", null, ...)` 来创建 VNode。这就是从模板到 render 函数的关键桥梁。

### 3.3 Transform 的处理逻辑

Transform 阶段会注册一系列 `nodeTransforms`，每个 transform 负责特定节点类型的转换：

- `transformElement` — 处理元素节点，生成 VNode 调用
- `transformText` — 处理文本节点
- `transformIf` — 处理 `v-if`，注入条件分支逻辑
- `transformFor` — 处理 `v-for`，注入循环逻辑
- `transformOn` — 处理 `@click` 等事件绑定
- `bindDirectiveTransforms` — 处理各种指令的绑定

每个 transform 都是一个纯函数，接收 AST 节点，修改或替换节点的 `codegenNode`。Transform 链式执行，最终把整个模板 AST 改造成 JavaScript 可执行的形态。

---

## 四、Generate：生成 render 函数

### 4.1 render 函数长什么样

最终生成的 render 函数如下：

```javascript
function render(_ctx, _cache) {
  with (_ctx) {
    const { openBlock: _openBlock, createElementBlock: _createElementBlock } = _Vue
    return (_openBlock(), _createElementBlock("div", null, "hello world"))
  }
}
```

> **注意**：`with` 语句在现代代码中不推荐使用，Vue 编译器用它来简化模板中对响应式变量的访问。

### 4.2 `with (_ctx)` 的作用

```javascript
with (_ctx) {
  const { openBlock, createElementBlock } = _Vue
  return (openBlock(), createElementBlock("div", null, "hello world"))
}
```

`_ctx` 就是组件实例（this）。`with (_ctx)` 让 render 函数内部可以直接访问 `data`、`props` 等响应式变量，而不用写 `this.xxx`。

去掉 `with` 的等价写法：

```javascript
function render(_ctx, _cache) {
  const { openBlock, createElementBlock } = _ctx.$Vue
  return (openBlock(), createElementBlock("div", null, "hello world"))
}
```

### 4.3 openBlock 和 createElementBlock

这是 Vue3 引入的**Block（块）概念**的核心：

- **`openBlock()`** — 打开一个新的动态节点收集区，记录后续创建的所有动态节点
- **`createElementBlock("div", null, "hello world")`** — 创建一个块级 VNode

为什么要分两步？Vue3 会在运行时**跳过静态节点，只比较动态节点**。`openBlock` 开启了一个"动态节点收集器"，所有动态部分（`{{ message }}`、动态 class、动态指令等）都会被收集进来，后续 patch 时只比较这些动态节点，大幅提升 Diff 效率。

### 4.4 render 函数的执行结果

render 函数最终返回一个 VNode：

```javascript
// 等价于
h('div', 'hello world')

// 更精确的等价于
createVNode('div', null, 'hello world')

// 或 Vue3 优化版本
createElementBlock('div', null, 'hello world')
```

render 函数 → VNode → render(vnode, container) → 真实 DOM，这便是 Vue 模板渲染的完整链路。

---

## 五、完整编译流程回顾

```
┌─────────────┐
│  template   │
│  模板字符串  │
└──────┬──────┘
       │  ① Parse（baseParse）
       ▼
┌─────────────┐
│     AST     │   ← 模板的结构化表示
│ 抽象语法树   │
└──────┬──────┘
       │  ② Transform（nodeTransforms 链）
       ▼
┌──────────────────┐
│ JavaScript AST  │   ← 多了 codegenNode
│  + codegenNode  │
└──────┬──────────┘
       │  ③ Generate（codegen）
       ▼
┌─────────────┐
│ render 函数  │   ← 最终产物
└──────┬──────┘
       │  ④ runtime 执行
       ▼
┌─────────────┐
│ 真实 DOM    │
└─────────────┘
```

---

## 六、面试高频问题

**Q：Vue 编译器的 parse 用的是什么算法？**

A：Vue 使用**状态机**进行词法分析和语法解析。Vue3 的 `baseParse` 基于一个状态机逐字符扫描模板，根据当前状态和输入字符决定跳转逻辑，最终输出 AST。状态机比正则匹配更可控、不易出现歧义。

**Q：Vue2 和 Vue3 的编译策略有什么区别？**

A：Vue2 是**完整编译**，模板 → render 函数在构建时完成，运行时直接执行。Vue3 引入了**Block + 动态节点收集**机制，通过 `openBlock` 收集动态子节点，运行时只需比较动态部分，Diff 效率更高。同时 Vue3 支持**动态组件（Teleport、Suspense）**，编译策略也做了相应适配。

**Q：为什么需要 `with` 语句？**

A：`with (_ctx)` 省略了每个变量的 `this.` 前缀。如果不用 with，模板里的 `{{ message }}` 就要写成 `this.message`，代码膨胀且可读性差。Vue3 在运行时规避了 with 的严格模式限制（在 `with` 外做了 `with` 隔离），所以可以放心使用。

**Q：codegenNode 的 `type: 13` 是什么？**

A：`type: 13` 对应 `NodeTypes.VNODE_CALL`，表示"调用 createElementBlock / createVNode 创建 VNode"。还有 `type: 12`（`TEXT_CALL`，文本节点）和 `type: 14`（`COMMENT_CALL`，注释节点）等。

---

*本文基于 Vue 3.2.0 源码分析，源码路径：`packages/compiler-core/`*