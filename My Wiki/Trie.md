---
type: concept
status: growing
topics: [algorithms, data-structure, strings, interview]
created: "2026-07-14"
---

# Trie

## Summary

Trie（Prefix tree，字典樹）按字元或數字位元逐層建立路徑，讓具有相同前綴的資料共用節點。

## Why It Matters

Trie 適合 Prefix search、Autocomplete、Dictionary lookup 與最長共同前綴。它的查詢成本主要取決於輸入長度，而不是資料筆數。

## How It Works

- Root 不代表實際字元。
- 每個節點保存下一個字元到子節點的映射。
- 插入時沿著輸入逐字建立缺少的節點。
- 查詢時沿既有路徑前進；路徑中斷的位置就是可匹配的最長前綴。
- 若要判斷完整單字，節點還要保存 `isTerminal`。

## Example

```javascript
class TrieNode {
  constructor() {
    this.children = {};
    this.isTerminal = false;
  }
}

class Trie {
  constructor() {
    this.root = new TrieNode();
  }

  insert(value) {
    let current = this.root;
    for (const char of String(value)) {
      current.children[char] ??= new TrieNode();
      current = current.children[char];
    }
    current.isTerminal = true;
  }

  longestPrefix(value) {
    let current = this.root;
    let length = 0;
    for (const char of String(value)) {
      if (!current.children[char]) break;
      current = current.children[char];
      length++;
    }
    return length;
  }
}
```

## Tradeoffs

- 查詢與插入通常是 `O(L)`，其中 `L` 是輸入長度。
- 節點與子節點映射會增加記憶體開銷；資料稀疏時可能比 `Set` 更耗空間。
- `Set` 適合完整值查找，Trie 更適合共用前綴與前綴查詢。

## Related

- [[Graph Traversal - DFS and BFS]]
- [[Dynamic Programming]]

## References

- [[2026-05-24 - 5-24 weekly update]]
- [LeetCode 3043](https://leetcode.com/problems/find-the-length-of-the-longest-common-prefix/)

