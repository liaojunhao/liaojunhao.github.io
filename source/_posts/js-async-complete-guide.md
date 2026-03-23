---
title: JavaScript 异步编程完全指南：从回调到 async/await
date: 2026-03-23 14:52:00
tags:
  - JavaScript
categories:
  - JavaScript 系列
---

# JavaScript 异步编程完全指南：从回调到 async/await

## 前言

异步编程是 JavaScript 的核心特性，也是最容易让初学者困惑的概念。从最初的回调函数，到 Promise，再到现代的 async/await，JavaScript 的异步编程模式经历了多次演进。本文将从**事件循环的底层机制**出发，系统地讲解 JavaScript 异步编程的全貌。

---

## 1. 同步 vs 异步：本质区别

### 1.1 同步执行

```javascript
function add(a, b) {
  return a + b;
}

const result = add(1, 2);
console.log(result); // 3 — 立即得到结果
```

同步代码**阻塞式执行**：每一行代码必须等待前一行完成才能执行。

### 1.2 异步执行

```javascript
function fetchData(callback) {
  setTimeout(() => {
    callback('data');
  }, 1000);
}

console.log('start');
fetchData((data) => {
  console.log(data); // 1秒后输出 'data'
});
console.log('end');

// 输出顺序：start → end → data
```

异步代码**非阻塞式执行**：某些操作可以在后台进行，不会阻塞主线程。

---

## 2. 事件循环（Event Loop）：异步的心脏

### 2.1 JavaScript 运行时模型

JavaScript 是**单线程**的，但通过事件循环机制实现了异步能力：

```
┌─────────────────────────────────────┐
│      JavaScript 引擎                 │
│  ┌──────────────────────────────┐   │
│  │   Call Stack（调用栈）        │   │
│  │  执行同步代码                  │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │   Heap（堆）                  │   │
│  │  存储对象和变量                │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
         ↓ 异步操作 ↓
┌─────────────────────────────────────┐
│      Web APIs / Node.js APIs         │
│  setTimeout, fetch, fs.readFile...   │
└─────────────────────────────────────┘
         ↓ 完成 ↓
┌─────────────────────────────────────┐
│   Callback Queue（回调队列）         │
│  Microtask Queue（微任务队列）       │
│  Macrotask Queue（宏任务队列）       │
└─────────────────────────────────────┘
         ↓ Event Loop ↓
    检查 Call Stack 是否为空
    是 → 从队列取任务执行
```

### 2.2 事件循环的执行顺序

```javascript
console.log('1. 同步代码');

setTimeout(() => {
  console.log('2. setTimeout（宏任务）');
}, 0);

Promise.resolve()
  .then(() => {
    console.log('3. Promise（微任务）');
  });

console.log('4. 同步代码');

// 输出顺序：
// 1. 同步代码
// 4. 同步代码
// 3. Promise（微任务）
// 2. setTimeout（宏任务）
```

**执行流程**：
1. 执行所有同步代码（1、4）
2. 同步代码执行完，检查微任务队列（3）
3. 微任务队列清空，执行宏任务（2）

### 2.3 微任务 vs 宏任务

| 类型 | 示例 | 优先级 |
|------|------|--------|
| **微任务** | Promise.then/catch/finally、MutationObserver、queueMicrotask | 高 |
| **宏任务** | setTimeout、setInterval、setImmediate、I/O、UI 渲染 | 低 |

```javascript
// 微任务优先级更高
console.log('start');

setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));

console.log('end');

// 输出：start → end → Promise → setTimeout
```

---

## 3. 回调函数（Callback）：异步的起点

### 3.1 基础回调

```javascript
function readFile(path, callback) {
  // 模拟异步读文件
  setTimeout(() => {
    callback(null, 'file content');
  }, 1000);
}

readFile('/path/to/file', (err, data) => {
  if (err) {
    console.error(err);
  } else {
    console.log(data);
  }
});
```

### 3.2 回调地狱（Callback Hell）

```javascript
// ❌ 嵌套过深，难以维护
readFile('/file1', (err, data1) => {
  if (!err) {
    readFile('/file2', (err, data2) => {
      if (!err) {
        readFile('/file3', (err, data3) => {
          if (!err) {
            console.log(data1, data2, data3);
          }
        });
      }
    });
  }
});
```

### 3.3 回调的问题

1. **可读性差**：嵌套深，难以理解流程
2. **错误处理困难**：每层都要单独处理错误
3. **无法复用**：回调逻辑难以提取和组合
4. **控制反转**：将控制权交给回调函数

---

## 4. Promise：异步的转折点

### 4.1 Promise 的三种状态

```javascript
const promise = new Promise((resolve, reject) => {
  // pending（待定）
  setTimeout(() => {
    resolve('success'); // → fulfilled（已兑现）
    // 或 reject('error'); → rejected（已拒绝）
  }, 1000);
});

promise
  .then(value => console.log(value))
  .catch(error => console.error(error));
```

**状态转移**：
- `pending` → `fulfilled`（通过 resolve）
- `pending` → `rejected`（通过 reject）
- 状态一旦改变，**不可逆转**

### 4.2 Promise 链式调用

```javascript
// ✅ 链式调用，易于理解
readFileAsPromise('/file1')
  .then(data1 => {
    console.log(data1);
    return readFileAsPromise('/file2');
  })
  .then(data2 => {
    console.log(data2);
    return readFileAsPromise('/file3');
  })
  .then(data3 => {
    console.log(data3);
  })
  .catch(error => {
    console.error('Error:', error);
  });
```

### 4.3 Promise 的常用方法

```javascript
// Promise.all — 所有 Promise 都成功才成功
Promise.all([p1, p2, p3])
  .then(([v1, v2, v3]) => console.log(v1, v2, v3))
  .catch(err => console.error(err));

// Promise.race — 最快的 Promise 决定结果
Promise.race([p1, p2, p3])
  .then(value => console.log('First:', value));

// Promise.allSettled — 等待所有 Promise 完成（无论成功失败）
Promise.allSettled([p1, p2, p3])
  .then(results => {
    results.forEach(r => {
      if (r.status === 'fulfilled') {
        console.log('Success:', r.value);
      } else {
        console.log('Error:', r.reason);
      }
    });
  });

// Promise.any — 至少一个成功就成功
Promise.any([p1, p2, p3])
  .then(value => console.log('At least one:', value))
  .catch(err => console.error('All failed:', err));
```

### 4.4 Promise 的常见陷阱

```javascript
// ❌ 陷阱 1：忘记 return
promise
  .then(data => {
    console.log(data);
    // 没有 return，下一个 then 收到 undefined
  })
  .then(data => {
    console.log(data); // undefined
  });

// ✅ 正确做法
promise
  .then(data => {
    console.log(data);
    return data; // 显式 return
  })
  .then(data => {
    console.log(data); // 正确的值
  });

// ❌ 陷阱 2：错误处理不当
promise
  .then(data => {
    throw new Error('oops');
  })
  .then(data => {
    // 这里不会执行
  })
  .catch(err => {
    console.error(err); // 捕获错误
  });

// ❌ 陷阱 3：Promise 中的异常被吞掉
new Promise((resolve, reject) => {
  throw new Error('error');
  // 自动转为 reject，但如果没有 catch，错误会被忽略
});
```

---

## 5. async/await：异步的终极形态

### 5.1 基础语法

```javascript
// async 函数总是返回 Promise
async function fetchData() {
  return 'data'; // 自动包装为 Promise.resolve('data')
}

fetchData().then(data => console.log(data)); // 'data'
```

### 5.2 await 的作用

```javascript
async function main() {
  // await 暂停执行，等待 Promise 完成
  const data = await fetchDataAsPromise();
  console.log(data); // 获得 Promise 的值
}

// 等价于
function main() {
  return fetchDataAsPromise()
    .then(data => {
      console.log(data);
    });
}
```

### 5.3 顺序执行 vs 并行执行

```javascript
// ❌ 顺序执行（低效）：总耗时 = 1000 + 1000 + 1000 = 3000ms
async function sequential() {
  const data1 = await fetch('/api/1'); // 1000ms
  const data2 = await fetch('/api/2'); // 1000ms
  const data3 = await fetch('/api/3'); // 1000ms
  return [data1, data2, data3];
}

// ✅ 并行执行（高效）：总耗时 = max(1000, 1000, 1000) = 1000ms
async function parallel() {
  const [data1, data2, data3] = await Promise.all([
    fetch('/api/1'),
    fetch('/api/2'),
    fetch('/api/3')
  ]);
  return [data1, data2, data3];
}
```

### 5.4 错误处理

```javascript
// 方式 1：try/catch
async function main() {
  try {
    const data = await fetchData();
    console.log(data);
  } catch (error) {
    console.error('Error:', error);
  }
}

// 方式 2：.catch()
async function main() {
  const data = await fetchData().catch(err => {
    console.error('Error:', err);
    return null; // 提供默认值
  });
  console.log(data);
}

// 方式 3：链式 catch
async function main() {
  return fetchData()
    .then(data => console.log(data))
    .catch(err => console.error(err));
}
```

### 5.5 async/await 的陷阱

```javascript
// ❌ 陷阱 1：在循环中顺序 await（低效）
async function processItems(items) {
  for (const item of items) {
    await processItem(item); // 顺序执行
  }
}

// ✅ 正确做法：并行处理
async function processItems(items) {
  await Promise.all(items.map(item => processItem(item)));
}

// ❌ 陷阱 2：忘记 await
async function main() {
  const promise = fetchData(); // 没有 await，返回 Promise
  console.log(promise); // Promise { <pending> }
}

// ✅ 正确做法
async function main() {
  const data = await fetchData();
  console.log(data); // 实际数据
}

// ❌ 陷阱 3：async 函数中的错误不会自动捕获
async function main() {
  throw new Error('oops');
  // 错误被包装在返回的 Promise 中
}

main(); // 如果没有 .catch()，错误会被忽略

// ✅ 正确做法
main().catch(err => console.error(err));
```

---

## 6. 实战场景

### 6.1 并发控制（限制并发数）

```javascript
async function concurrentMap(items, fn, limit = 3) {
  const results = [];
  const executing = [];

  for (const [index, item] of items.entries()) {
    const promise = Promise.resolve()
      .then(() => fn(item))
      .then(result => {
        results[index] = result;
      });

    executing.push(promise);

    if (executing.length >= limit) {
      await Promise.race(executing);
      executing.splice(executing.findIndex(p => p === promise), 1);
    }
  }

  await Promise.all(executing);
  return results;
}

// 使用示例
const urls = ['/api/1', '/api/2', '/api/3', '/api/4', '/api/5'];
const results = await concurrentMap(urls, fetch, 3); // 最多同时 3 个请求
```

### 6.2 重试机制

```javascript
async function retryAsync(fn, maxRetries = 3, delay = 1000) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, delay * (i + 1)));
    }
  }
}

// 使用示例
const data = await retryAsync(
  () => fetch('/api/unstable'),
  3,
  1000
);
```

### 6.3 超时控制

```javascript
function withTimeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);
}

// 使用示例
try {
  const data = await withTimeout(fetch('/api/slow'), 5000);
} catch (error) {
  console.error('Request timeout or failed:', error);
}
```

### 6.4 顺序执行多个异步操作

```javascript
// 方式 1：reduce
async function sequence(tasks) {
  return tasks.reduce(
    (promise, task) => promise.then(results => 
      task().then(result => [...results, result])
    ),
    Promise.resolve([])
  );
}

// 方式 2：for...of（更清晰）
async function sequence(tasks) {
  const results = [];
  for (const task of tasks) {
    results.push(await task());
  }
  return results;
}

// 使用示例
const results = await sequence([
  () => fetchUser(1),
  () => fetchUser(2),
  () => fetchUser(3)
]);
```

---

## 7. 异步编程的最佳实践

| 原则 | 说明 |
|------|------|
| **优先使用 async/await** | 比 Promise 链更易读，更接近同步代码 |
| **并行优于顺序** | 无依赖关系的异步操作应该并行执行 |
| **正确处理错误** | 使用 try/catch 或 .catch()，不要忽略错误 |
| **避免回调地狱** | 使用 Promise 或 async/await 替代深层嵌套 |
| **理解事件循环** | 了解微任务和宏任务的执行顺序 |
| **控制并发数** | 大量异步操作时，限制并发数避免资源耗尽 |
| **设置超时** | 网络请求应该有超时机制 |
| **实现重试机制** | 不稳定的操作应该支持重试 |

---

## 8. 总结

```
异步编程演进路线：

回调函数 → Promise → async/await
  ↓         ↓         ↓
难维护    易链式    最优雅
嵌套深    错误处理  同步风格
        更清晰    最易读
```

JavaScript 的异步编程从回调函数开始，经过 Promise 的改进，最终在 async/await 中达到了最优形态。理解事件循环、微任务和宏任务的执行机制，是掌握异步编程的关键。

在实际开发中，优先使用 async/await，合理利用 Promise.all 进行并行处理，正确处理错误，才能写出高效、可维护的异步代码。

---

> **JS 系列持续更新中**，下一篇将深入解析 **JavaScript 原型链与继承机制**，敬请关注。
