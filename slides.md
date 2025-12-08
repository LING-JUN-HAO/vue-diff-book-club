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

![benchmark 圖片](./public/benchmark-1.jpg) {.section-image}

---

# 快速 Diff 流程

<v-clicks every="1" depth="2">

1. 前置節點比對：從頭開始比對相同的節點，決定 j 索引位置。
2. 後置節點比對：從尾端開始比對相同的節點，決定 oldEnd 和 newEnd 索引位置。
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

![11-1 相同的前置節點.png](./public/11-1-F.jpg){.fix-dark}

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

![11-2 相同的後置節點.png](./public/11-1-B.jpg){.fix-dark}

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

# 處理純新增節點的情形

1. **j > oldEnd**：代表預處理過程已處理完全部的舊子節點
2. **newEnd ≥ j**：代表新子節點中，有未被處理的子節點需要掛載

<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  const newChildren = n2.children
  const oldChildren = n1.children
  // 省略前置節點及後置節點的處理
  // 純新增節點條件
  if (j > oldEnd && j <= newEnd) {
        // 錨點索引
        const anchorIndex = newEnd + 1
        // 錨點元素
        const anchor = anchorIndex < newChildren.length ? newChildren[anchorIndex].el : null
        // 使用 while 循環，調用 patch 逐個掛載節點
        while (j <= newEnd) {
          patch(null, newChildren[j++], container, anchor)
        }
  }
}
```

![11-1 純新增情形.png](./public/11-1-create.jpg){.fix-dark}

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

# 處理純刪除節點的情形

1. **j > newEnd**：代表預處理過程已處理完全部的新子節點
2. **oldEnd ≥ j**：代表舊子節點中，有需要被卸載處理的子節點

<div class="flex overflow-hidden gap-4">

```ts
  function patchKeyedChildren(n1, n2, container) {
    const newChildren = n2.children
    const oldChildren = n1.children
    // 省略前置節點及後置節點的處理
    // 純移除節點條件
		if (j > oldEnd && j <= newEnd) {
			// 新增節點
		}else if(j > newEnd && j <= oldEnd){
			// 卸載節點
			while( j <= oldEnd){
				unmount(oldChildren[j++])
			}
		}
  }
```

![11-1 純新增情形.png](./public/11-1-delete.jpg){.fix-dark}

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