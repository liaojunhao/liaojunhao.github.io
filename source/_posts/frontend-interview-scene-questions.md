---
title: 前端面试高频场景题精讲：10 个你必须掌握的设计问题
date: 2026-03-24 14:41:00
categories:
  - 前端面试系列
tags:
  - 面试
  - 前端
---

# 前端面试高频场景题精讲：10 个你必须掌握的设计问题

## 前言

场景题是前端面试中最能拉开差距的题型。它不考死记硬背，而是考察你在真实业务中的**系统设计能力、性能意识和工程经验**。本文精选 10 道高频场景题，从问题本质出发，给出完整的分析思路和代码实现。

---

## 如何实现一个准确的前端倒计时？

### 问题本质

`setInterval(fn, 1000)` 并不保证每 1000ms 准时执行。JS 是单线程的，遇到大量计算、页面渲染、长任务时，定时器会被延迟。页面切到后台，浏览器还会主动降频定时器。

**结果**：本该 1 秒减一次，实际可能 1.2 秒甚至 2 秒才执行一次，倒计时越来越慢。

### 正确设计思路

- **以时间戳差值为核心**，而不是依赖 `setInterval` 的执行次数
- `setInterval` 只作为 UI 刷新工具，每次执行时重新计算剩余时间

```javascript
function createCountdown(durationMs, onTick, onEnd) {
  const endTime = Date.now() + durationMs;

  const timer = setInterval(() => {
    const remain = endTime - Date.now();

    if (remain <= 0) {
      clearInterval(timer);
      onTick(0);
      onEnd?.();
      return;
    }

    onTick(remain);
  }, 200); // 刷新频率可以更高，保证精度

  return () => clearInterval(timer); // 返回清理函数
}

// 使用示例
const stop = createCountdown(
  60 * 1000,
  (remain) => {
    const seconds = Math.ceil(remain / 1000);
    document.getElementById("timer").textContent = `${seconds}秒`;
  },
  () => console.log("倒计时结束")
);
```

### 进阶：页面切后台后恢复

```javascript
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "visible") {
    // 页面重新可见，立即刷新一次，避免显示旧数据
    const remain = endTime - Date.now();
    onTick(Math.max(0, remain));
  }
});
```

---

## 如何设计精准的秒杀倒计时？

### 问题本质

普通倒计时的问题是依赖本地时间，而用户电脑时间可能不准（快几分钟或慢几分钟），导致倒计时提前或延迟结束。

### 正确设计思路

1. **以服务器时间为基准**，获取一次服务端当前时间
2. **计算本地时间与服务器时间的差值 `diff`**，后续用 `Date.now() + diff` 代替服务器时间
3. **防止前端篡改**：真正是否可支付由后端控制，前端倒计时只是展示

```javascript
async function initSeckillCountdown(activityStartTime) {
  // 1. 获取服务器时间
  const { serverTime } = await fetch("/api/server-time").then((r) => r.json());

  // 2. 计算时间差（本地时间 - 服务器时间）
  const diff = Date.now() - serverTime;

  // 3. 倒计时刷新
  const timer = setInterval(() => {
    // 用修正后的时间计算剩余时间
    const correctedNow = Date.now() - diff;
    const remain = activityStartTime - correctedNow;

    if (remain <= 0) {
      clearInterval(timer);
      showBuyButton(); // 展示购买按钮
      return;
    }

    updateCountdownUI(remain);
  }, 500);
}

function updateCountdownUI(remain) {
  const hours = Math.floor(remain / 3600000);
  const minutes = Math.floor((remain % 3600000) / 60000);
  const seconds = Math.floor((remain % 60000) / 1000);
  // 更新 DOM...
}
```

---

## Web 管理系统越用越慢，如何排查？

### 排查思路：先定位，再优化

```
越用越慢 → 三个方向
├── 网络层：接口响应变慢？
├── 前端层：JS 执行慢？内存泄漏？
└── 后端层：数据库查询慢？
```

### 使用 Chrome DevTools 定位

| 工具            | 排查内容                    |
| --------------- | --------------------------- |
| **Network**     | 接口 TTFB、响应时间是否变长 |
| **Performance** | JS 执行耗时、渲染帧率       |
| **Memory**      | 内存是否持续增长（泄漏）    |
| **Lighthouse**  | 整体性能评分和优化建议      |

### 前端常见问题与解决方案

```javascript
// ❌ 问题 1：大列表一次性渲染
// 一次渲染 10000 条数据，DOM 节点过多
list.forEach((item) => {
  const el = document.createElement("div");
  el.textContent = item.name;
  container.appendChild(el);
});

// ✅ 解决：虚拟列表（只渲染可视区域）
// 使用 vue-virtual-scroller / react-window 等库

// ❌ 问题 2：事件监听未释放（内存泄漏）
function init() {
  window.addEventListener("resize", handleResize);
  // 组件销毁时忘记移除
}

// ✅ 解决：组件销毁时清理
onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
});

// ❌ 问题 3：定时器未清理
function startPolling() {
  setInterval(fetchData, 5000);
  // 每次进入页面都新增一个定时器
}

// ✅ 解决：保存引用并清理
let timer = null;
function startPolling() {
  if (timer) clearInterval(timer);
  timer = setInterval(fetchData, 5000);
}
```

---

## 后端返回几万条数据，前端表格如何展示？

### 核心原则：不一次性渲染所有 DOM

一次性渲染 5 万条数据会创建大量 DOM 节点，导致浏览器卡死。

### 方案一：虚拟列表（推荐）

只渲染可视区域内的行，滚动时动态替换内容：

```javascript
class VirtualList {
  constructor({ container, itemHeight, data, renderItem }) {
    this.container = container;
    this.itemHeight = itemHeight;
    this.data = data;
    this.renderItem = renderItem;
    this.init();
  }

  init() {
    const totalHeight = this.data.length * this.itemHeight;
    this.container.style.position = "relative";
    this.container.style.height = `${totalHeight}px`;

    this.visibleContainer = document.createElement("div");
    this.visibleContainer.style.position = "absolute";
    this.visibleContainer.style.width = "100%";
    this.container.appendChild(this.visibleContainer);

    this.container.parentElement.addEventListener("scroll", () =>
      this.render()
    );
    this.render();
  }

  render() {
    const scrollTop = this.container.parentElement.scrollTop;
    const viewHeight = this.container.parentElement.clientHeight;

    const startIndex = Math.floor(scrollTop / this.itemHeight);
    const endIndex = Math.min(
      startIndex + Math.ceil(viewHeight / this.itemHeight) + 1,
      this.data.length
    );

    this.visibleContainer.style.top = `${startIndex * this.itemHeight}px`;
    this.visibleContainer.innerHTML = "";

    for (let i = startIndex; i < endIndex; i++) {
      const el = this.renderItem(this.data[i], i);
      this.visibleContainer.appendChild(el);
    }
  }
}
```

### 方案二：Web Worker 处理数据

数据量大时，数据处理（排序、过滤）放到 Worker 中，避免阻塞主线程：

```javascript
// worker.js
self.onmessage = ({ data: { list, keyword } }) => {
  const result = list.filter((item) => item.name.includes(keyword));
  self.postMessage(result);
};

// main.js
const worker = new Worker("./worker.js");
worker.postMessage({ list: bigData, keyword: "搜索词" });
worker.onmessage = ({ data }) => {
  renderTable(data);
};
```

---

## H5 瀑布流在低端安卓机和弱网下如何优化？

### 优化策略全景

```
优化方向
├── 图片资源优化（最关键）
│   ├── WebP/AVIF 格式
│   ├── 多尺寸响应式图片（srcset）
│   └── 低清图占位（LQIP）
├── 网络加载优化
│   ├── IntersectionObserver 懒加载
│   ├── 分批请求（分页加载）
│   └── 请求失败自动重试
├── 渲染优化
│   ├── 虚拟瀑布流
│   └── 使用 transform 代替 top/left
└── 容错降级
    ├── 骨架屏占位
    └── 弱网自动切换简化模式
```

```javascript
// 图片懒加载（IntersectionObserver）
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;
        img.removeAttribute("data-src");
        observer.unobserve(img);
      }
    });
  },
  {
    rootMargin: "200px 0px", // 提前 200px 开始加载
  }
);

document.querySelectorAll("img[data-src]").forEach((img) => {
  observer.observe(img);
});

// 弱网检测降级
const connection = navigator.connection;
if (connection?.effectiveType === "2g" || connection?.saveData) {
  // 切换到简化模式：小图、无动画
  document.body.classList.add("lite-mode");
}
```

---

## 如何设计一个支持任意内容的单选框组件？

### 设计思路

使用 `RadioGroup + RadioItem` 组合组件，通过 `slot` 支持任意内容，用 `provide/inject` 传递状态：

```vue
<!-- RadioGroup.vue -->
<template>
  <div class="radio-group" role="radiogroup">
    <slot />
  </div>
</template>

<script setup>
import { provide, toRefs } from "vue";

const props = defineProps({
  modelValue: { type: [String, Number], required: true },
});
const emit = defineEmits(["update:modelValue"]);

provide("radioGroup", {
  ...toRefs(props),
  onChange: (val) => emit("update:modelValue", val),
});
</script>
```

```vue
<!-- RadioItem.vue -->
<template>
  <div
    class="radio-item"
    :class="{ 'is-checked': isChecked }"
    role="radio"
    :aria-checked="isChecked"
    :tabindex="0"
    @click="select"
    @keydown.space.prevent="select"
  >
    <span class="radio-indicator" />
    <slot />
  </div>
</template>

<script setup>
import { inject, computed } from "vue";

const props = defineProps({
  value: { type: [String, Number], required: true },
  disabled: { type: Boolean, default: false },
});

const { modelValue, onChange } = inject("radioGroup");
const isChecked = computed(() => modelValue.value === props.value);

const select = () => {
  if (!props.disabled) onChange(props.value);
};
</script>
```

---

## 如何排查网页白屏问题？

### 排查流程

```
白屏
├── 打开控制台，看 Console 有没有 JS 报错
│   └── 有报错 → 定位报错代码，修复
├── 看 Network，资源是否加载成功
│   ├── HTML/JS/CSS 404 → 检查部署配置、CDN
│   └── 接口 500/超时 → 后端问题，前端做容错
└── 看 DOM 结构，body 是否为空
    ├── 空 → JS 执行报错，框架未挂载
    └── 有内容但不显示 → CSS 问题（display:none、z-index 等）
```

### 工程化预防方案

```javascript
// 1. 全局错误监控
window.onerror = (message, source, lineno, colno, error) => {
  reportError({ type: "js_error", message, source, lineno, error });
};

window.addEventListener("unhandledrejection", (event) => {
  reportError({ type: "promise_error", reason: event.reason });
});

// 2. 资源加载失败监控
window.addEventListener(
  "error",
  (event) => {
    if (event.target !== window) {
      reportError({
        type: "resource_error",
        src: event.target.src || event.target.href,
      });
    }
  },
  true
);

// 3. React ErrorBoundary
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    reportError({ type: "render_error", error, info });
  }

  render() {
    if (this.state.hasError) {
      return <div>页面出错了，请刷新重试</div>;
    }
    return this.props.children;
  }
}
```

---

## 如何启动一个新的前端项目？

### 完整流程

**第一步：需求对齐**

- 产品形态（后台系统 / C 端 / 小程序）
- 用户量和性能要求（是否需要 SSR/SEO）
- 工期（决定技术方案保守程度）

**第二步：技术选型**

| 维度     | 选项                       |
| -------- | -------------------------- |
| 框架     | Vue 3 / React 18 / Next.js |
| 语言     | TypeScript（强烈推荐）     |
| UI 库    | Ant Design / Element Plus  |
| 构建工具 | Vite                       |
| 状态管理 | Pinia / Zustand            |

**第三步：工程化搭建**

```bash
# 代码规范
npm install -D eslint prettier eslint-config-prettier
npm install -D husky lint-staged  # 提交前自动检查

# 目录结构
src/
├── api/          # 接口层，按模块拆分
├── components/   # 通用组件
├── views/        # 页面组件
├── hooks/        # 复用逻辑
├── stores/       # 状态管理
├── router/       # 路由 + 权限守卫
└── utils/        # 工具函数
```

**第四步：基础设施**

```typescript
// Axios 统一封装
const request = axios.create({ baseURL: import.meta.env.VITE_API_URL });

request.interceptors.request.use((config) => {
  config.headers.Authorization = `Bearer ${getToken()}`;
  return config;
});

request.interceptors.response.use(
  (res) => res.data,
  (error) => {
    if (error.response?.status === 401) logout();
    return Promise.reject(error);
  }
);
```

---

## 如何实现前端线上监控？

### 三步走：采集 → 上报 → 分析

**采集**

```javascript
class Monitor {
  init() {
    this.watchJsError();
    this.watchResourceError();
    this.watchPerformance();
    this.watchApiError();
  }

  watchJsError() {
    window.onerror = (msg, url, line, col, error) => {
      this.report({ type: "js_error", msg, url, line, stack: error?.stack });
    };
    window.addEventListener("unhandledrejection", (e) => {
      this.report({ type: "promise_error", reason: String(e.reason) });
    });
  }

  watchPerformance() {
    window.addEventListener("load", () => {
      const { domContentLoadedEventEnd, navigationStart, loadEventEnd } =
        performance.timing;
      this.report({
        type: "performance",
        dclTime: domContentLoadedEventEnd - navigationStart,
        loadTime: loadEventEnd - navigationStart,
      });
    });
  }

  report(data) {
    const payload = {
      ...data,
      url: location.href,
      userAgent: navigator.userAgent,
      timestamp: Date.now(),
    };
    // 使用 sendBeacon 保证页面关闭时也能上报
    navigator.sendBeacon("/api/monitor", JSON.stringify(payload));
  }
}
```

**排查线上报错**

1. 通过 source map 将压缩代码映射回源码
2. 按错误类型聚合，找出高频错误
3. 结合用户行为路径复现问题
4. 本地开发环境复现并修复

---

## 百万人同时抢一个商品，如何判断第一个？

### 核心在后端，前端只负责展示

前端能做的：

- 按钮防抖/防重复点击
- 请求发出后立即禁用按钮
- 展示排队/结果状态

```javascript
// 前端防重复提交
let isSubmitting = false;

async function handleBuy() {
  if (isSubmitting) return;
  isSubmitting = true;
  buyBtn.disabled = true;

  try {
    const result = await fetch("/api/seckill", { method: "POST" });
    const { success, message } = await result.json();
    showResult(success ? "抢购成功！" : message);
  } finally {
    isSubmitting = false;
    buyBtn.disabled = false;
  }
}
```

后端核心方案（Redis 原子操作）：

```bash
# Redis SETNX：只有第一个写入的用户才能成功
SETNX product:1:winner userId

# 返回 1 → 抢购成功
# 返回 0 → 已被抢走
```

---

## 总结

| 题目             | 核心考点                       |
| ---------------- | ------------------------------ |
| 准确倒计时       | 时间戳差值，不依赖定时器次数   |
| 秒杀倒计时       | 服务器时间校准，防篡改         |
| 系统越来越慢     | DevTools 定位，内存泄漏排查    |
| 几万条数据展示   | 虚拟列表，Web Worker           |
| 瀑布流低端机优化 | 懒加载，降级策略，虚拟列表     |
| 单选框组件设计   | slot + provide/inject 组合组件 |
| 白屏排查         | 系统化排查流程，ErrorBoundary  |
| 新项目启动       | 需求对齐，技术选型，工程化     |
| 前端监控         | 采集、上报、source map 还原    |
| 百万并发抢购     | 前端防重，后端 Redis 原子操作  |

场景题没有标准答案，面试官考察的是你的**思考过程**：能否快速定位问题本质，给出合理的设计方案，并考虑到边界情况和工程实践。

---

> **前端面试系列持续更新中**，下一篇将深入讲解 **前端性能优化完全指南**，敬请关注。
