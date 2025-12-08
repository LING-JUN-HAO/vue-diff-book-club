---
theme: seriph
colorSchema: light
lineNumbers: true
title: Vue.js 架構與設計(快速 diff & 渲染器實作渲染)
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

# Vue.js 架構與設計

## 快速 Diff & 組件的實現原理

---

# 快速 Diff 起源

- 借鏡了 ivi 和 inferno 框架，其速度在 js-framework-benchmark 中，比 Vue 2 還快。

其核心概念是使用文字預處理的思路，將文本對齊之後，再進行 Diff 的比較。

```text
welcome to Guangdong, i hope you have a great travel.
welcome to Beijing, i hope you have a great travel.
```

![benchmark 圖片](./benchmark-1.jpg) {.section-image}

---

# 快速 Diff 流程

<v-clicks every="1" depth="2">

1. 前置節點對比：從頭開始比對相同的節點
2. 後置節點對比：從尾端開始比對相同的節點
3. 純新增處理：處理只有新增節點的情況
4. 純刪除處理：處理只有刪除節點的情況
5. 複雜場景處理：當存在節點移動時，使用 diff 演算法
    - 建立 key → index 映射表
    - 遍歷舊節點建立 source 映射關係
    - 掛載新節點 & 透過最長遞增子序列最小化移動操作

</v-clicks>

<style scoped>
.slidev-layout ol li + li {
  margin-top: 0.75rem;
}

.slidev-layout ol li ul {
  margin-top: 0.75rem;
}
</style>

---

# 找出相同的前置節點

<div class="flex overflow-hidden gap-4">

```ts
  function patchKeyedChildren(n1, n2, container) {
    const newChildren = n2.children
    const oldChildren = n1.children
    // j 用於定義前置節點位置
    let j = 0
    let oldVNode = oldChildren[j]
    let newVNode = newChildren[j]

    // while 向右遍歷，直到 key 不相同
    while (oldVNode.key === newVNode.key) {
      // 調用 patch 函數更新
      patch(oldVNode, newVNode, container)
      j++
      oldVNode = oldChildren[j]
      newVNode = newChildren[j]
    }
    // ...
  }
```

![11-1 相同的前置節點跟後置節點.png](./11-1-F.jpg){.fix-dark}

</div>

<style scoped>
.slidev-layout p:has(> img){
  text-align: center;
  width: 50%;
}

.slidev-code-wrapper{
  width: 50%;
  overflow: auto;
}

img{
  object-fit: contain;
}
</style>

---

# 找出相同的後置節點

<div class="flex overflow-hidden gap-4">

```ts
  function patchKeyedChildren(n1, n2, container) {
    const newChildren = n2.children
    const oldChildren = n1.children
    // 前置節點處理(略過)
    
    // 後置節點處理
    let oldEnd = oldChildren.length - 1 // 舊子節點最後的索引值
    let newEnd = newChildren.length - 1 // 新子節點最後的索引值

    oldVNode = oldChildren[oldEnd]
    newVNode = newChildren[newEnd]
    
    // while 向左遍歷，直到遇到 key 不相同
    while (oldVNode.key === newVNode.key) {
      // 調用 patch 函數更新
      patch(oldVNode, newVNode, container)
      oldEnd--
      newEnd--
      oldVNode = oldChildren[oldEnd]
      newVNode = newChildren[newEnd]
    }
    // ...
  }
```

![11-1 相同的前置節點跟後置節點.png](./11-1-F.jpg){.fix-dark}

</div>

<style scoped>
.slidev-layout p:has(> img){
  text-align: center;
  width: 50%;
}

.slidev-code-wrapper{
  width: 50%;
  overflow: auto;
}

img{
  object-fit: contain;
}
</style>