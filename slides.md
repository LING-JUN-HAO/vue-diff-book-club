---
theme: seriph
colorSchema: light
lineNumbers: true
selectable: true
contextMenu: true
title: Vue.js 架構與設計(快速 diff & 組件的實作原理)
class: text-center
author: Antonio
favicon: ./public/favicon.png
drawings:
  enabled: true
  persist: false
  presenterOnly: false
  syncAll: true
transition: slide-left
mdc: true
seoMeta:
  ogTitle: Vue.js 快速 diff & 組件的實作原理
  ogDescription: Vue.js 設計與實現第十一章與十二章重點整理
  ogImage: https://ling-jun-hao.github.io/vue-diff-book-club/OgImage.png
  ogUrl: https://ling-jun-hao.github.io/vue-diff-book-club/
  twitterCard: summary_large_image
  twitterTitle: Vue.js 快速 diff & 組件的實作原理
  twitterDescription: Vue.js 設計與實現第十一章與十二章重點整理
  twitterImage: https://ling-jun-hao.github.io/vue-diff-book-club/OgImage.png
  twitterSite: Antonio
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

![benchmark 測試結果](./public/benchmark-1.jpg) {.section-image}

<style scope>
.slidev-code-wrapper{
  width: 100%;
  overflow: clip;
}
</style>

---

# 快速 Diff 流程

<v-clicks every="1" depth="2">

1. 前置節點比對：從頭開始比對相同的節點，決定 j 索引位置。
2. 後置節點比對：從尾端開始比對相同的節點，決定 oldEnd 和 newEnd 索引位置。
3. 純新增處理：處理只有新增節點的情況
4. 純刪除處理：處理只有刪除節點的情況
5. 複雜場景處理：當存在節點移動時，使用 diff 演算法
    - 建立 key → index 映射表，判斷節點是否可以復用
    - 遍歷舊節點建立 source 映射關係 & 判斷節點是否需要移動
    - 掛載新節點 & 透過最長遞增子序列最小化移動操作

</v-clicks>

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

![相同的前置節點](./public/11-1-F.jpg)

</div>

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

![相同的後置節點](./public/11-1-B.jpg)

</div>

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

![純新增情形](./public/11-1-create.jpg)

</div>

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
			// 舊子節點比較完，僅需處理掛載節點
		}else if(j > newEnd && j <= oldEnd){
			// 新子節點比較完，僅需處理需卸載節點
			while( j <= oldEnd){
				unmount(oldChildren[j++])
			}
		}
  }
```

![純刪除情形](./public/11-1-delete.jpg)

</div>

---

# 處理複雜場景

在前置與後置比對階段，只會遇到「純新增」與「純刪除」的情況，不需要進行移動操作。但在更複雜的情形下，新舊子節點的順序可能發生變動，且無法一一對應。這時就需要透過**中段節點 diff**，綜合處理節點的復用、刪除、新增與移動。


<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  // 省略前置節點及後置節點的處理
  if (j > oldEnd && j <= newEnd) {
    // 舊子節點比較完，僅需處理掛載節點
  }else if(j > newEnd && j <= oldEnd){
    // 新子節點比較完，僅需處理需卸載節點
  }else{
    // 中段 diff 處理複雜場景
}
```

![複雜場景](./public/11-2-1.jpg)

</div>

---

# 中段 diff 處理流程 1 - 確認可以複用的舊子節點


- 雙層迴圈遍歷新舊子節點中段區域，找出可以複用的節點，並調用 patch 函數更新內容。

**時間複雜度 O(n x m) ⇒ O(n^ 2)**

<div class="flex overflow-hidden gap-4">

```ts
  function patchKeyedChildren(n1, n2, container) {
    // 省略前置節點及後置節點的處理
		if (j > oldEnd && j <= newEnd) {
			// 舊子節點比較完，僅需處理掛載節點
		}else if(j > newEnd && j <= oldEnd){
			// 新子節點比較完，僅需處理需卸載節點
		}else{
			// 中段 diff 比較處理區域		
		  // oldStart 和 newStart 分別為起始索引，即 j
		  const oldStart = j
		  const newStart = j
		
		  // 雙層迴圈找出可覆用的節點
		  for (let i = oldStart; i <= oldEnd; i++) {
		    const oldVNode = oldChildren[i]
		    for (let k = newStart; k <= newEnd; k++) {
		      const newVNode = newChildren[k]
		      if (oldVNode.key === newVNode.key) {
            // 更新覆用節點內容
		        patch(oldVNode, newVNode, container)
		      }
		    }
		  }
  }
```

![雙層迴圈找可復用節點](./public/11-2-1.jpg)

</div>

<v-click>

> 隨著新舊子節點數量增加，效能會急劇下降。因此需要優化這個過程

</v-click>

---

# 中段 diff 處理流程 1 - 確認可複用的舊子節點(優化版)


- 使用**keyIndex 索引表**：將新子節點的 key 映射到索引位置，透過單層迴圈遍歷舊子節點，並使用索引表進行 O(1) 時間查找對應的新節點。

**時間複雜度降低成：O(n + m)  ⇒ O(n)**

<div class="flex overflow-hidden gap-4">

```ts
  function patchKeyedChildren(n1, n2, container) {
    // 省略前置節點及後置節點的處理
		if (j > oldEnd && j <= newEnd) {
			// 舊子節點比較完，僅需處理掛載節點
		}else if(j > newEnd && j <= oldEnd){
			// 新子節點比較完，僅需處理需卸載節點
		}else{
			// 中段 diff 比較處理區域		
		  // oldStart 和 newStart 分別為起始索引，即 j
		  const oldStart = j
		  const newStart = j
		
      const keyIndex = new Map()

      // 遍歷新子節點，建立索引表
      for (let i = newStart; i <= newEnd; i++) {
        keyIndex.set(newChildren[i].key, i)
      }

      // 遍歷舊子節點，通過索引表查找
      for (let i = oldStart; i <= oldEnd; i++) {
        const oldVNode = oldChildren[i]
        
        // O(1) 時間查找對應的新節點索引
        const k = keyIndex.get(oldVNode.key)

        if (typeof k !== undefined) {
          // 找到可覆用的節點
          const newVNode = newChildren[k]
          patch(oldVNode, newVNode, container)
        }
      }
  }
}
```

![keyIndex 優化舊子節點復用](./public/11-2-1.jpg)

</div>

> keyIndex Map：鍵是「新子節點的 key」，值是「新節點在新子節點陣列中的索引」

---

# 中段 diff 處理流程 2 - 遍歷舊節點建立 source 映射關係

前面已經能判斷哪些舊節點可以複用，接下來需要建立一個 source 陣列來記錄這些可複用節點的對應關係。

透過 source 陣列，我們可以追蹤哪些舊節點被複用、哪些需要刪除，以及哪些新節點需要新增。

<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  // 省略前置節點及後置節點的處理
  if (j > oldEnd && j <= newEnd) {
    // 舊子節點比較完，僅需處理掛載節點
  }else if(j > newEnd && j <= oldEnd){
    // 新子節點比較完，僅需處理需卸載節點
  }else{
      // 中段 diff 比較處理區域

      // == 新增 source 陣列建立映射關係(預設填滿 -1) ==
      const count = newEnd - j + 1
      const source = new Array(count)
      source.fill(-1)
        // == 新增 source 陣列建立映射關係 ==

      const oldStart = j
      const newStart = j
    
      const keyIndex = new Map()

      for (let i = newStart; i <= newEnd; i++) {
        keyIndex.set(newChildren[i].key, i)
      }

      for (let i = oldStart; i <= oldEnd; i++) {
        const oldVNode = oldChildren[i]
        const k = keyIndex.get(oldVNode.key)

        if (typeof k !== undefined) {
          // 找到可覆用的節點
          const newVNode = newChildren[k]
          patch(oldVNode, newVNode, container)
          // 更新 source 陣列映射關係
          source[k - newStart] = i
        }
      }
  }
}
```

![source 陣列紀錄新舊節點位置映射關係](./public/11-2-2.jpg)

</div>

> source 陣列：記錄新子節點對應的舊子節點索引位置（若無對應則為 -1）

---

# source 陣列應用 - 舊子節點卸載

- **找不到對應新節點**：遍歷舊子節點時，若某個舊節點在新節點中找不到匹配的 key，表示該節點需要被卸載。
- **已復用數量達到上限**：當已複用的節點數達到新子節點總數時，表示所有需要的節點都已找到，剩餘的舊節點都是多餘的，應直接卸載。

<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  // 省略前置節點及後置節點的處理
  if (j > oldEnd && j <= newEnd) {
    // 舊子節點比較完,僅需處理掛載節點
  } else if (j > newEnd && j <= oldEnd) {
    // 新子節點比較完,僅需處理需卸載節點
  } else {
    const count = newEnd - newStart + 1
    const source = new Array(count)
    source.fill(-1)
    
    const oldStart = j
    const newStart = j

    // ★ patched 用於記錄已複用的節點數量
    let patched = 0
    
    const keyIndex = new Map()
    for (let i = newStart; i <= newEnd; i++) {
      keyIndex.set(newChildren[i].key, i)
    }
    
    for (let i = oldStart; i <= oldEnd; i++) {
      const oldVNode = oldChildren[i]
      
      // ★ 檢查：是否還有新節點需要匹配
      if (patched < count) {
        const k = keyIndex.get(oldVNode.key)
        
        if (k !== undefined) {
          const newVNode = newChildren[k]
          patch(oldVNode, newVNode, container)
          patched++
          source[k - newStart] = i
        } else {
          // ★ 舊節點在新節點中找不到對應，需卸載
          unMount(oldVNode)
        }
      } else {
        // ★ 所有新節點都已處理完畢，剩餘的舊節點為多餘節點，直接卸載
        unMount(oldVNode)
      }
    }
  }
}
```

![source 陣列紀錄新舊節點位置映射關係](./public/11-2-2.jpg)

</div>

> - Source 陣列長度是中段區域新子節點的數量(newEnd - newStart + 1)
> - Source 陣列紀錄的索引是中段區域的新子節點在舊子節點陣列中的位置(k - newStart)

---

#  中段 diff 處理流程 2 - 判斷節點是否需要移動

在處理節點移動前，需要先判斷是否存在需要移動的節點。若存在，則使用「最長遞增子序列(LIS)」演算法來優化移動操作，減少 DOM 操作次數。

<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  // 省略前置節點及後置節點的處理
  if (j > oldEnd && j <= newEnd) {
    // 舊子節點比較完,僅需處理掛載節點
  } else if (j > newEnd && j <= oldEnd) {
    // 新子節點比較完,僅需處理需卸載節點
  } else {
    const count = newEnd - newStart + 1
    const source = new Array(count)
    source.fill(-1)
    
    const oldStart = j
    const newStart = j
    let moved = false
    let pos = 0
    let patched = 0
    
    const keyIndex = new Map()
    for (let i = newStart; i <= newEnd; i++) {
      keyIndex.set(newChildren[i].key, i)
    }
    
    for (let i = oldStart; i <= oldEnd; i++) {
      const oldVNode = oldChildren[i]
      
      if (patched < count) {
        const k = keyIndex.get(oldVNode.key)
        
        if (typeof k !== undefined) {
          const newVNode = newChildren[k]
          patch(oldVNode, newVNode, container)
          patched++
          source[k - newStart] = i
          
          // ★ 判斷是否需要移動(跟簡單 diff 方法一樣，透過是否保持遞增性來判斷)
          if (k < pos) {
            moved = true
          } else {
            pos = k
          }
        } else {
          unMount(oldVNode)
        }
      } else {
        unMount(oldVNode)
      }
    }
  }
}
```

![source 陣列紀錄新舊節點位置映射關係](./public/11-2-2.jpg)

</div>

---

# 中段 diff 處理流程到目前為止存在的問題

前面已經完成舊子節點的處理：複用可複用的節點、卸載多餘的節點，並判斷是否需要移動

但如果有全新的節點需要新增呢？

**考慮以下情境：**


<div class="flex overflow-hidden gap-4">


![新子節點掛載的情形](./public/11-3-1.jpg)

<div class="flex flex-col gap-3">

  <v-click>  

  **當前情況不符合以下兩個原則**
  - j 沒有大於 newEnd，不屬於純刪除
  - j 沒有大於 oldEnd，不屬於純新增

  </v-click>

  <v-click>  

  **當我們處理完前面的步驟後：**
  - p-1 節點在前置比對中複用，位置不變
  - p-2 節點在中段處理中複用，索引遞增，不需移動

  </v-click>

  <v-click>  

  **★ 但我們遺漏了 p-3 節點！**

  </v-click>
</div>

</div>

<style>

.slidev-layout p:has(> img){
  margin: auto;
}

</style>

---

# 中段 diff 處理流程 3 - 掛載新節點

當遍歷新子節點的中段 diff 區域時，若 source 陣列中對應位置為 **-1**，代表該節點是全新的，需要執行掛載操作

<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  // 省略前置節點及後置節點的處理
  if (j > oldEnd && j <= newEnd) {
    // 舊子節點比較完,僅需處理掛載節點
  } else if (j > newEnd && j <= oldEnd) {
    // 新子節點比較完,僅需處理需卸載節點
  } else {
    const count = newEnd - newStart + 1
    const source = new Array(count)
    source.fill(-1)
    
    const oldStart = j
    const newStart = j
    let moved = false
    let pos = 0
    let patched = 0
    
    const keyIndex = new Map()
    for (let i = newStart; i <= newEnd; i++) {
      keyIndex.set(newChildren[i].key, i)
    }
    
    // 遍歷舊節點建立 source 映射關係 & 判斷節點是否需要移動
    for (let i = oldStart; i <= oldEnd; i++) {...}

    // 從後向前遍歷（只需要一次循環）
		for (let i = count - 1; i >= 0; i--) {
		  const pos = newStart + i
		  const newVNode = newChildren[pos]
		  const nextPos = pos + 1
		  const anchor = nextPos < newChildren.length 
		    ? newChildren[nextPos].el 
		    : null
		  
		  if (source[i] === -1) {
		    // 新增節點
		    patch(null, newVNode, container, anchor)
		  }
    }
  }
}
```

![新子節點掛載的情形](./public/11-3-1.jpg)

</div>

> anchor 是 null，將 p-3 插入新子節點陣列的最後面

---

# 如何減少 DOM 移動？認識 LIS 最長遞增子序列

LIS 最長遞增子序列：在陣列中找出最長的遞增子序列，元素需保持原順序但不必連續。

這邊介紹比較直觀的動態規劃方法。
- 時間複雜度：O(n^2)

<div class="flex overflow-hidden gap-4">

![LIS最長遞增子序列](./public/LIS最長遞增子序列.jpg)

![LIS 比對結果](./public/LIS比對結果.jpg)

</div>

> 最終輸出：[2, 3, 7, 18]，長度為 4（可能有多組解）

[資料來源](https://medium.com/%E6%8A%80%E8%A1%93%E7%AD%86%E8%A8%98/leetcode-%E8%A7%A3%E9%A1%8C%E7%B4%80%E9%8C%84-300-longest-increasing-subsequence-f160358db4d1)

---

# 中段 diff 處理流程 3 - 利用 LIS 最小化移動操作

目的：在遍歷新子節點時，保持 LIS 中的節點位置不動，僅移動其他節點到正確位置。


<div class="flex overflow-hidden gap-4">

```ts
function patchKeyedChildren(n1, n2, container) {
  // 省略前置節點及後置節點的處理
  if (j > oldEnd && j <= newEnd) {
    // 舊子節點比較完,僅需處理掛載節點
  } else if (j > newEnd && j <= oldEnd) {
    // 新子節點比較完,僅需處理需卸載節點
  } else {
    const count = newEnd - newStart + 1
    const source = new Array(count)
    source.fill(-1)
    
    const oldStart = j
    const newStart = j
    let moved = false
    let pos = 0
    let patched = 0
    
    const keyIndex = new Map()
    for (let i = newStart; i <= newEnd; i++) {
      keyIndex.set(newChildren[i].key, i)
    }
    for (let i = oldStart; i <= oldEnd; i++) {...}

    // ★ 透過 source 陣列取得 LIS 序列
    const seq = moved ? getSequence(source) : []
		let s = seq.length - 1
    // 從後向前遍歷（只需要一次循環）
		for (let i = count - 1; i >= 0; i--) {
		  const pos = newStart + i
		  const newVNode = newChildren[pos]
		  const nextPos = pos + 1
		  const anchor = nextPos < newChildren.length 
		    ? newChildren[nextPos].el 
		    : null
		  
		  if (source[i] === -1) {
		    patch(null, newVNode, container, anchor)
		  }else if(moved){
        // 判斷當前新子節點索引是否在 LIS 中
        if (s < 0 || i !== seq[s]) {
          // 是移動位置而不是更新(更新在前面遍歷舊節點時已處理)
          insert(newVNode.el, container, anchor)
        }else{
          // 存在子序列中，指標向前移動
          s--
        }
      }
    }
  }
}
```

![最長遞增子序列比對情形](./public/11-3-2.jpg)

</div>

---

# 渲染組件

---
layout: center
---

# Thank you for listening.