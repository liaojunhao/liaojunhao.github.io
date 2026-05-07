---
title: Vue3 Diff 算法详解：从原理到源码实现
date: 2026-05-06 17:30:00
categories:
  - JavaScript 系列
tags:
  - JavaScript
  - Vue
  - 前端
---

## 前言

Diff 算法是 Virtual DOM 的核心，也是前端面试中的高频难点。本文基于 Vue 3.2 最新源码，手把手分析 Diff 的五大步骤，以及最长递增子序列在其中扮演的关键角色。

---

## 一、Diff 算法的触发场景

很多人以为「DOM 更新就会触发 Diff」，这是一个误区。

**Diff 只在"一组 DOM"发生变化时被触发**——即 `v-for` 循环渲染多个节点时，数据变化导致旧 VNode 数组需要更新为新 VNode 数组的场景。

```html
<template>
  <ul>
    <li v-for="item in arr" :key="item.id">
      {{ item.title }}
    </li>
  </ul>
  <button @click="onChange">修改</button>
</template>

<script>
export default {
  data() {
    return {
      arr: [
        { id: '1', title: 'a' },
        { id: '2', title: 'b' },
        { id: '3', title: 'c' }
      ]
    }
  },
  methods: {
    onChange() {
      this.arr[2] = { id: '4', title: 'd' }
      // 触发 Diff：一组 li 需要更新
    }
  }
}
</script>
```

---

## 二、key 的作用与 isSameVNodeType

### 2.1 为什么需要 key

上面的例子中，点击修改后，只有第三个 `li` 重新渲染了，前两个没有变化。Vue 是怎么知道的？

答案在于 `isSameVNodeType` 方法：

```typescript
// packages/runtime-core/src/vnode.ts
function isSameVNodeType(n1: VNode, n2: VNode): boolean {
  return n1.type === n2.type && n1.key === n2.key
}
```

- **type**：VNode 节点类型（`'li'`、`'div'`、`'comment'`、组件实例等）
- **key**：`v-for` 绑定的唯一标识

当 `type` 和 `key` 都相同时，两个 VNode 被认为是"相同的"，只需 `patch` 打补丁，无需重新创建 DOM。

### 2.2 key 的重要性

没有 key 或 key 不稳定，会导致 Vue 无法正确复用 DOM 节点，引发性能问题和状态错乱：

```html
<!-- ❌ 不推荐：用 index 作为 key -->
<li v-for="(item, index) in arr" :key="index">

<!-- ✅ 推荐：用唯一 id 作为 key -->
<li v-for="item in arr" :key="item.id">
```

---

## 三、Diff 算法的五大步骤

Vue3 的 `patchKeyedChildren` 将 Diff 分为 5 步：

| 步骤 | 名称 | 作用 |
|------|------|------|
| 1 | sync from start | 自前向后对比，相同节点直接 patch |
| 2 | sync from end | 自后向前对比，相同节点 patch |
| 3 | common sequence + mount | 新节点多于旧节点 → 挂载新增 |
| 4 | common sequence + unmount | 旧节点多于新节点 → 卸载多余 |
| 5 | unknown sequence | 乱序（最复杂，涉及移动） |

前四步处理"新、旧节点数量相等或简单增删"的场景，第五步处理**乱序**——这是真正考验算法功力的地方。

---

## 四、前四步详解

### 4.1 步骤一：sync from start（自前向后）

```typescript
while (i <= oldChildrenEnd && i <= newChildrenEnd) {
  const oldVNode = oldChildren[i]
  const newVNode = normalizeVNode(newChildren[i])
  
  if (isSameVNodeType(oldVNode, newVNode)) {
    patch(oldVNode, newVNode, container, null) // 相同 → 打补丁
  } else {
    break // 不同 → 停止
  }
  i++
}
```

从头部开始，依次对比相同下标的 VNode。相同则 patch，不同则退出。

### 4.2 步骤二：sync from end（自后向前）

```typescript
while (i <= oldChildrenEnd && i <= newChildrenEnd) {
  const oldVNode = oldChildren[oldChildrenEnd]
  const newVNode = normalizeVNode(newChildrenEnd)
  
  if (isSameVNodeType(oldVNode, newVNode)) {
    patch(oldVNode, newVNode, container, null)
  } else {
    break
  }
  oldChildrenEnd--
  newChildrenEnd--
}
```

逻辑与步骤一相反，从尾部开始对比。两者结合可以快速处理"首尾相同、中间变化"的场景。

### 4.3 步骤三：新节点多于旧节点

```typescript
if (i > oldChildrenEnd) {
  if (i <= newChildrenEnd) {
    const nextPos = newChildrenEnd + 1
    const anchor = nextPos < newChildrenLength 
      ? newChildren[nextPos].el 
      : parentAnchor
    
    while (i <= newChildrenEnd) {
      patch(null, normalizeVNode(newChildren[i]), container, anchor)
      i++
    }
  }
}
```

通过 `arr.push()` 添加新元素时，多出的节点位于尾部；通过 `arr.unshift()` 添加时，多出的节点位于头部。核心是找到正确的锚点（anchor）来插入新增 DOM。

### 4.4 步骤四：旧节点多于新节点

```typescript
else if (i > newChildrenEnd) {
  while (i <= oldChildrenEnd) {
    unmount(oldChildren[i]) // 直接卸载
    i++
  }
}
```

通过 `arr.pop()` 或 `arr.shift()` 删除元素时，只需逐个卸载多余节点。

---

## 五、最长递增子序列（核心前置知识）

第五步乱序 Diff 中，最长递增子序列（Longest Increasing Subsequence，简称 LIS）是关键优化手段。

### 5.1 什么是 LIS

在一个数值序列中，找出一个子序列，使其元素依次递增，且长度尽可能长。

```
旧节点: 1, 2, 3, 4, 5, 6
新节点: 1, 3, 2, 4, 6, 5

可能的递增子序列：
  1 → 3 → 6        (长度 3)
  1 → 2 → 4 → 6    (长度 4) ← 这就是最长递增子序列
```

### 5.2 LIS 在 Diff 中的作用

Diff 的本质操作是：**新增、删除、打补丁、移动**。

对于乱序场景，如果直接逐个移动，复杂度是 O(n)。但利用 LIS，我们可以**只移动"不在递增子序列中"的节点**，大幅减少移动次数：

```
递增子序列: 1 → 2 → 4 → 6（4个节点无需移动）
需要移动的: 3 和 5（仅需 2 次移动）

对比：若没有 LIS 策略，需要移动 4 次才能完成转换
```

### 5.3 Vue3 中 getSequence 的实现

Vue3 使用"贪心 + 二分查找"实现 LIS，时间复杂度 O(n log n)：

```typescript
// packages/runtime-core/src/renderer.ts
function getSequence(arr: number[]): number[] {
  const p = arr.slice()        // 回溯数组
  const result = [0]          // 存储下标
  let i, j, u, v, c
  
  for (i = 0; i < arr.length; i++) {
    const arrI = arr[i]
    if (arrI !== 0) {
      j = result[result.length - 1]
      
      // arr[j] < arrI → 找到更大的序列，直接加入
      if (arr[j] < arrI) {
        p[i] = j
        result.push(i)
        continue
      }
      
      // 二分查找：找最小的且大于 arrI 的元素位置
      u = 0
      v = result.length - 1
      while (u < v) {
        c = (u + v) >> 1
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          v = c
        }
      }
      
      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1]
        }
        result[u] = i
      }
    }
  }
  
  // 回溯得到最终下标
  u = result.length
  v = result[u - 1]
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  return result
}
```

**核心思想**：
- 遍历数组，维护一个递增的结果序列
- 用二分查找快速找到"第一个大于等于当前值"的位置
- 最终回溯得到完整的递增下标序列

---

## 六、第五步详解：乱序 Diff

这是整个 Diff 算法最复杂的部分，源码约 150 行，分三大块：

### 6.1 构建 keyToNewIndexMap

```typescript
const keyToNewIndexMap = new Map()
for (i = s2; i <= e2; i++) {
  const nextChild = normalizeVNode(newChildren[i])
  if (nextChild.key != null) {
    keyToNewIndexMap.set(nextChild.key, i)
  }
}
```

建立一个 `Map<key, newIndex>`，快速查找新节点在哪个位置。

### 6.2 遍历旧节点，patch 或 unmount

```typescript
const newIndexToOldIndexMap = new Array(toBePatched).fill(0)

for (i = s1; i <= e1; i++) {
  const prevChild = oldChildren[i]
  
  // 超出新节点数量 → 直接卸载
  if (patched >= toBePatched) {
    unmount(prevChild, ...)
    continue
  }
  
  // 通过 key 查找新节点位置
  let newIndex = keyToNewIndexMap.get(prevChild.key)
  
  // 没找到 → 旧节点没有对应的新节点，卸载
  if (newIndex === undefined) {
    unmount(prevChild, ...)
  } else {
    // 记录：新节点 index 对应的旧节点 index（+1，0 代表新增）
    newIndexToOldIndexMap[newIndex - s2] = i + 1
    
    // 判断是否需要移动
    if (newIndex >= maxNewIndexSoFar) {
      maxNewIndexSoFar = newIndex
    } else {
      moved = true // 出现了"逆序"，需要移动
    }
    
    patch(prevChild, newChildren[newIndex], ...)
    patched++
  }
}
```

核心：遍历旧节点，找到对应的新节点位置，记录到 `newIndexToOldIndexMap`，并判断是否需要移动。

### 6.3 处理移动与挂载

```typescript
const increasingNewIndexSequence = moved 
  ? getSequence(newIndexToOldIndexMap) 
  : []

j = increasingNewIndexSequence.length - 1

for (i = toBePatched - 1; i >= 0; i--) {
  const nextIndex = s2 + i
  const nextChild = newChildren[nextIndex]
  const anchor = nextIndex + 1 < l2 
    ? newChildren[nextIndex + 1].el 
    : parentAnchor
  
  // 新节点没有对应的旧节点 → 挂载
  if (newIndexToOldIndexMap[i] === 0) {
    patch(null, nextChild, container, anchor, ...)
  }
  // 需要移动
  else if (moved) {
    // 不在最长递增子序列中 → 移动
    if (j < 0 || i !== increasingNewIndexSequence[j]) {
      move(nextChild, container, anchor, MoveType.REORDER)
    } else {
      j-- // 在递增子序列中，不移动
    }
  }
}
```

**为什么倒序处理？** 倒序可以用已处理好的节点作为锚点（anchor），避免每次都查找插入位置。

---

## 七、完整流程总结

```
旧: [A, B, C, D, E]    新: [A, C, B, F, D]
```

**步骤一**：A vs A → patch，B vs C → 不同，停止
**步骤二**：E vs D → 不同，停止
**步骤三**：无新节点多
**步骤四**：无旧节点多
**步骤五**（乱序）：
1. 构建 keyToNewIndexMap
2. 遍历旧节点对比新节点位置
3. 计算 LIS，确定最小移动次数
4. 移动 + 挂载

**移动次数 = 新节点数 - LIS 长度**

---

## 八、面试高频问题

**Q：为什么 Vue3 推荐用唯一 id 作为 key，而不用 index？**

A：用 index 作为 key 时，如果数据中间插入或删除，index 对应的节点会变化，导致 Vue 无法正确复用 DOM，所有节点都需要重新渲染，严重的会引发状态错乱。用唯一 id 可以确保 VNode 身份稳定，精准复用。

**Q：Diff 算法的时间复杂度是多少？**

A：Vue3 Diff 整体是 O(n)。前四步是 O(n)，第五步乱序因为有 LIS 优化，从暴力 O(n²) 降到 O(n log n)。

**Q：没有 key 会怎样？**

A：Vue 会退化为"就地复用"策略，按位置逐一对比，无法正确复用节点，行为不可控，容易出 bug。

---

*本文基于 Vue 3.2.0 源码分析，如有问题欢迎交流探讨。*
