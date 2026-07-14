---
type: concept
status: growing
topics: [algorithms, graph, interview]
created: "2026-07-14"
---

# Graph Traversal: DFS and BFS

## Summary

Depth-First Search（DFS）會沿著一條路徑深入後再回頭；Breadth-First Search（BFS）則逐層擴散。兩者都需要管理「下一個節點」與 `visited` 狀態，但適合的題型不同。

## Why It Matters

圖、樹、迷宮、連通性與路徑問題經常需要先判斷應該「深入嘗試」還是「逐層搜尋」。選錯方法不一定無法解題，但可能讓狀態管理或複雜度變得不必要地困難。

## How It Works

### DFS

- 適合列出所有路徑、排列、組合或子集合。
- 常用於 Tree traversal、連通性、Cycle detection 與 Topological sort。
- Backtracking 題型通常遵循「選擇 → 遞迴 → 復原狀態」。
- 圖搜尋要記錄 `visited`，避免循環；重複子問題可搭配 Memoization。
- 可用 Base case、當前處理、遞迴擴展、狀態復原與 Pruning 來設計。

### BFS

- 從起點逐層擴散，通常使用 Queue。
- 適合最短步數、最少操作次數或「第幾層首次抵達」的問題。
- 節點加入 Queue 時就應標記為已訪問，避免同一節點被重複排入。

## Example

```javascript
function dfs(node, visited) {
  if (visited.has(node)) return;
  visited.add(node);

  for (const next of neighbors(node)) {
    dfs(next, visited);
  }
}

function bfs(start) {
  const queue = [start];
  const visited = new Set([start]);

  while (queue.length) {
    const current = queue.shift();
    for (const next of neighbors(current)) {
      if (!visited.has(next)) {
        visited.add(next);
        queue.push(next);
      }
    }
  }
}
```

## Tradeoffs

- 遞迴 DFS 容易表達樹狀結構，但深度太大時可能 Stack overflow。
- BFS 需要保存整層節點，寬度很大時會使用較多記憶體。
- DFS 找到的第一條路通常不是最短路；無權重圖的 BFS 第一次抵達通常代表最少步數。

## Related

- [[Dynamic Programming]]
- [[Trie]]

## References

- [[2026-04-27 - 4-27 進辦公室]]
- [[2026-05-31 - 5-31 weekly update]]
- [LeetCode 1391](https://leetcode.com/problems/check-if-there-is-a-valid-path-in-a-grid)

