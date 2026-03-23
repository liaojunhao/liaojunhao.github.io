---
title: JavaScript 闭包深度解析：从词法作用域到内存管理
date: 2026-03-22 13:18:00
tags:
  - JavaScript
  - 前端
  - JS 系列
categories:
  - JavaScript 系列
---

# JavaScript 闭包深度解析：从词法作用域到内存管理

## 前言

闭包（Closure）是 JavaScript 中最核心、最常被面试考查、却也最容易被误解的概念之一。本文将从**词法作用域的底层机制**出发，逐步剖析闭包的形成原理、典型应用场景、常见陷阱，以及生产环境中的内存管理策略。

---

## 1. 闭包的定义与本质

MDN 对闭包的定义如下：

> A closure is the combination of a function bundled together (enclosed) with references to its surrounding state (the lexical environment).

翻译过来：**闭包 = 函数 + 该函数被创建时的词法环境的引用**。

当 JavaScript 引擎执行函数时，会为它创建一个**执行上下文（Execution Context）**，其中包含：

- **词法环境（Lexical Environment）**：记录当前作用域内的变量绑定
- **变量环境（Variable Environment）**：处理 `var` 声明
- **this 绑定**

闭包的本质在于：**内部函数创建时，会捕获（capture）其外层函数的词法环境，并在外层函数执行完毕后依然持有该引用**。

```javascript
function createScope() {
  const secret = 42; // 被内部函数捕获

  return function reveal() {
    return secret; // 即使 createScope 已返回，secret 仍可访问
  };
}

const fn = createScope();
console.log(fn()); // 42
```

正常情况下，`createScope` 执行完毕后，其执行上下文应该被销毁，`secret` 也应被垃圾回收。但由于返回的 `reveal` 函数持有对 `secret` 的引用，引擎不会释放这块内存——这就是闭包。

---

## 2. 词法作用域 vs 动态作用域

理解闭包的前提是理解 JavaScript 采用的是**词法作用域（Lexical Scoping）**。

### 2.1 词法作用域

函数的作用域在**代码编写时**就已经确定，而非运行时决定：

```javascript
const x = 'global';

function outer() {
  const x = 'outer';

  function inner() {
    console.log(x); // 'outer' — 在编写时就已经确定了作用域链
  }

  inner();
}

outer();
```

### 2.2 与动态作用域的对比

如果 JavaScript 采用动态作用域（如 Bash），`inner` 中的 `x` 会从**调用栈**中查找，结果可能完全不同。词法作用域保证了代码行为的**可预测性**，这也是闭包能够可靠工作的基础。

---

## 3. 闭包的形成机制

### 3.1 作用域链（Scope Chain）

当访问一个变量时，引擎会沿作用域链逐级向上查找：

```
inner 函数作用域 → outer 函数作用域 → 全局作用域
```

闭包使得这条链即使在外层函数执行完毕后也不会断裂。

### 3.2 [[Environment]] 内部槽

根据 ECMAScript 规范，每个函数对象都有一个内部槽 `[[Environment]]`，保存创建时的词法环境引用：

```javascript
function outer() {
  let a = 1;
  let b = 2;

  return function inner() {
    // inner.[[Environment]] → outer 的词法环境 { a: 1, b: 2 }
    console.log(a + b);
  };
}
```

### 3.3 不同创建方式的闭包

闭包不仅存在于 `return` 场景，任何将内部函数「逃逸」到外部作用域的方式都会形成闭包：

```javascript
// 回调函数
function delay(msg, ms) {
  setTimeout(() => console.log(msg), ms); // 箭头函数捕获 msg
}

// 事件监听
function setupButton(el) {
  let count = 0;
  el.addEventListener('click', () => {
    console.log(`Clicked ${++count} times`);
  });
}

// Promise
function fetchUser(id) {
  return fetch(`/api/users/${id}`)
    .then(res => res.json()) // 回调捕获 res
    .then(user => console.log(user.name)); // 回调捕获 user
}
```

---

## 4. 经典应用场景

### 4.1 模块模式（Module Pattern）

在 ES Modules 出现之前，闭包是实现模块化的核心手段：

```javascript
const UserModule = (function () {
  // 私有状态
  const cache = new Map();

  function validate(data) {
    return data && data.name && data.email;
  }

  // 公开接口
  return {
    get(id) {
      if (cache.has(id)) return cache.get(id);
      // 模拟异步获取
      return fetch(`/api/users/${id}`)
        .then(r => r.json())
        .then(user => {
          cache.set(id, user);
          return user;
        });
    },
    clear() {
      cache.clear();
    }
  };
})();
```

### 4.2 柯里化（Currying）

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function (...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
}

const add = curry((a, b, c) => a + b + c);
console.log(add(1)(2)(3));     // 6
console.log(add(1, 2)(3));     // 6
console.log(add(1)(2, 3));     // 6
```

### 4.3 偏函数应用（Partial Application）

```javascript
function bind(fn, context, ...presetArgs) {
  return function (...laterArgs) {
    return fn.apply(context, [...presetArgs, ...laterArgs]);
  };
}

// 使用示例：创建日志函数
const logError = bind(console.log, console, '[ERROR]');
const logInfo = bind(console.log, console, '[INFO]');

logError('Something went wrong'); // [ERROR] Something went wrong
logInfo('Server started');       // [INFO] Server started
```

### 4.4 防抖与节流

```javascript
// 防抖（Debounce）
function debounce(fn, delay) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// 节流（Throttle）
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
```

### 4.5 迭代器模式

```javascript
function createIterator(array) {
  let index = 0;

  return {
    next() {
      if (index < array.length) {
        return { value: array[index++], done: false };
      }
      return { value: undefined, done: true };
    },
    [Symbol.iterator]() {
      return this;
    }
  };
}

const iter = createIterator([10, 20, 30]);
console.log(iter.next()); // { value: 10, done: false }
console.log(iter.next()); // { value: 20, done: false }
console.log(iter.next()); // { value: 30, done: false }
console.log(iter.next()); // { value: undefined, done: true }
```

---

## 5. 经典陷阱与解决方案

### 5.1 循环中的闭包（var 陷阱）

这是最经典的闭包面试题：

```javascript
// ❌ 错误：三次输出 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**根因分析**：`var` 声明的 `i` 具有函数作用域，三个回调共享同一个 `i`。当回调执行时，循环已经结束，`i` 的值为 3。

**解法一：`let`（推荐）**

```javascript
// ✅ 正确：输出 0, 1, 2
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

`let` 具有块级作用域，每次循环迭代都会创建一个独立的 `i` 绑定。

**解法二：IIFE 手动创建作用域**

```javascript
// ✅ 正确：通过 IIFE 捕获每次迭代的 i
for (var i = 0; i < 3; i++) {
  ((j) => {
    setTimeout(() => console.log(j), 0);
  })(i);
}
```

### 5.2 意外的共享引用

```javascript
function createFunctions() {
  const result = [];
  for (let i = 0; i < 3; i++) {
    result.push(() => i); // let 下每次迭代是独立绑定，无问题
  }
  return result;
}

const fns = createFunctions();
console.log(fns[0]()); // 0
console.log(fns[1]()); // 1
console.log(fns[2]()); // 2
```

但如果捕获的是**引用类型**，仍需注意：

```javascript
function createHandlers() {
  const handlers = [];
  const state = { count: 0 };

  for (let i = 0; i < 3; i++) {
    handlers.push(() => {
      state.count++;
      return state.count;
    });
  }
  return handlers;
}

const [h1, h2, h3] = createHandlers();
console.log(h1()); // 1
console.log(h2()); // 2
console.log(h3()); // 3 — 共享同一个 state 对象
```

---

## 6. 内存管理与性能考量

### 6.1 闭包与垃圾回收

V8 引擎采用**可达性（Reachability）**算法进行垃圾回收。闭包使得外层变量保持可达，因此不会被回收。

```javascript
function heavy() {
  const hugeArray = new Array(1000000).fill('data');

  return function use() {
    return hugeArray.length;
  };
}

const fn = heavy(); // hugeArray 常驻内存
fn = null; // 解除引用，hugeArray 可被回收
```

### 6.2 常见内存泄漏场景

```javascript
// 场景 1：未清理的事件监听
function bindEvent(el) {
  const bigData = loadBigData();
  el.addEventListener('click', () => {
    process(bigData);
  });
  // 忘记移除监听 → el、bigData 均无法回收
}
// 修复：el.removeEventListener('click', handler)

// 场景 2：闭包中的定时器
function startPolling() {
  const cache = new Map();
  setInterval(() => {
    // 如果不清除定时器，cache 永远不会被回收
  }, 1000);
}

// 场景 3：循环引用（现代引擎已能处理，但仍需注意）
function leak() {
  const obj = {};
  obj.self = obj; // 循环引用
  return () => obj;
}
```

### 6.3 性能优化建议

| 策略 | 说明 |
|------|------|
| 及时解引用 | 不再使用的闭包赋值为 `null` |
| 避免不必要闭包 | 能用块级作用域解决的，不要用闭包 |
| 控制捕获范围 | 只捕获需要的变量，不要捕获整个外部环境 |
| 使用 WeakMap | 对于引用类型，考虑使用 `WeakMap` 避免阻止垃圾回收 |

```javascript
// 使用 WeakMap 避免内存泄漏
const privateData = new WeakMap();

class User {
  constructor(name) {
    privateData.set(this, { name });
  }
  getName() {
    return privateData.get(this).name;
  }
}
```

---

## 7. 闭包在 React 中的实战

React Hooks 的大量模式本质上都是闭包的应用：

```javascript
// useEffect 闭包陷阱
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // ❌ 闭包捕获了 count = 0，每次都是 0 + 1 = 1
      // setCount(count + 1);
      // ✅ 使用函数式更新，避免闭包陷阱
      setCount(prev => prev + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

---

## 8. 总结

```
闭包的核心知识图谱：

                    ┌─────────────┐
                    │   闭包本质   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        词法作用域    [[Environment]]  作用域链
              │            │            │
              └────────────┼────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
      模块模式         函数柯里化       防抖节流
          │                │                │
          └────────────────┼────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
      内存管理         常见陷阱         React 实战
```

闭包不是什么黑魔法，它是 JavaScript 词法作用域机制的**必然产物**。理解它的本质，才能在日常开发和架构设计中真正驾驭它。

---

> **JS 系列持续更新中**，下一篇将深入解析 **JavaScript 原型链与继承机制**，敬请关注。
