---
type: concept
status: growing
topics: [algorithms, dynamic-programming, interview]
created: "2026-07-14"
---

# Dynamic Programming

## Summary

Dynamic Programming（DP）把問題拆成重疊的子問題，保存已計算的結果，以額外空間換取較少的重複運算。

## Why It Matters

當遞迴會反覆處理相同狀態時，單純暴力搜尋可能呈指數成長。DP 能把問題轉成有限狀態之間的轉移，常用於最佳化、計數與可達性問題。

## How It Works

適合 DP 的問題通常同時具備：

1. **Optimal substructure**：大問題的解可由子問題的解組合而成。
2. **Overlapping subproblems**：不同路徑會重複遇到相同子問題。

兩種常見做法：

- **Top-down + Memoization**：從目標開始遞迴，將已算過的狀態快取起來。
- **Bottom-up + Tabulation**：先計算最小狀態，再依賴關係填滿 DP table。

設計時先明確定義 State、Transition、Base case 與 Answer。若有額外限制，可增加維度；例如 `dp[i][j][w]` 表示走到 `(i, j)` 且使用成本 `w` 時的最佳分數。

## Example

```javascript
function fibonacci(n) {
  if (n <= 1) return n;

  const dp = [0, 1];
  for (let i = 2; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}
```

## Tradeoffs

- 多維狀態可以表達更多限制，但時間與空間複雜度會快速增加。
- Bottom-up 通常沒有遞迴堆疊成本，但狀態順序較難推導。
- Memoization 容易從暴力解逐步優化，但仍要留意遞迴深度。

## Related

- [[Graph Traversal - DFS and BFS]]
- [[Trie]]

## References

- [[2026-04-30 - Dynamic programming]]
- [LeetCode 3742](https://leetcode.com/problems/maximum-path-score-in-a-grid)

