---
title: "Dynamic programming"
date: 2026-04-30
tags: ["Interview 2026"]
source: "https://app.notion.com/p/3546ed462a26807aa2e6c391be4e8d4b"
notion_id: "3546ed462a26807aa2e6c391be4e8d4b"
---

# Dynamic programming
將一個大問題拆解成許多「重疊」的小問題，並將這些小問題的答案儲存起來，避免重複計算。
## **核心概念**
記憶與重複利用
DP 最經典的特徵就是 **「空間換取時間」**。如果一個問題在遞迴過程中會反覆計算相同的子問題，DP 就會把第一次算的結果存進表格（陣列或物件）中。
**費波那契數列 (Fibonacci)**<br>當你計算 `F(5)` 時，你需要算 `F(4)` 和 `F(3)`；但算 `F(4)` 時又要再算一次 `F(3)`。<br>• **普通遞迴**：重複計算 `F(3)`，效率極低。<br>• **DP**：算出 `F(3)` 後存起來，下次直接拿來用。
## 判斷一個問題是否能用 DP
通常一個問題要具備以下兩個特性，才適合用 DP：
1. **最佳子結構 (Optimal Substructure)**：<br>大問題的最佳解，可以由小問題的最佳解組合而成。
2. **重疊子問題 (Overlapping Subproblems)**：<br>在解決大問題的過程中，會不斷遇到一模一樣的小問題。
## 解決 DP 問題的兩種方法
在實作上，通常有兩種思考方向：
- **由上而下 (Top-Down) + 備忘錄 (Memoization)**：<br>從大問題開始，用遞迴去拆解。如果遇到算過的，就從記憶庫拿出來。
- **由下而上 (Bottom-Up) + 表格法 (Tabulation)**：<br>從小問題開始算起，依序填表，直到算出最終目標。這通常是用 `for` 迴圈來達成，也是面試中最常要求的寫法。
## Leetcodes
### Leetcode 3742: Maximum Path Score in a Grid
[Maximum Path Score in a Grid - LeetCode](https://leetcode.com/problems/maximum-path-score-in-a-grid)
思考路徑：
1. 遍歷迷宮：
	1. 想像從左上角 `(0, 0)` 走到右下角 `(m-1, n-1)`。
	2. 決策點：在每一個格子，你只有**兩種**選擇：從**上面**來，或是從**左邊**來。
2. 評估每一步的代價與收穫
	1. 目前累積花了多少 Cost？（花費 `w`）
	2. 這個狀況下拿到的最高分是多少？（存下這個最高分）
3. 看上面來的最高分 VS 看左邊來的最高分
4. 挑選分數較高者 + 當前格子分數 → 更新我們的表格紀錄
規劃程式碼
1. 建立「**資料庫**」 (DP Table)
	1. 宣告一個三維陣列，用來記錄所有狀態的最大分數。
	2. 維度定義：`dp[i][j][w]` 代表走到格子 `(i, j)` 且花費 `w` 點 Cost 時的最大 Score。
	3. 初始值：全部填入 `-1`（代表該狀態還沒走到、或者不可達）。
	```javascript
// 建立 m x n x (k+1) 的三維陣列
const dp: number[][][] = Array.from({ length: m }, () =>
  Array.from({ length: n }, () => new Array(k + 1).fill(-1))
);
	```
2. 設定起點 (0, 0)
	```javascript
const firstCost = grid[0][0] > 0 ? 1 : 0;
if (firstCost <= k) {
  dp[0][0][firstCost] = grid[0][0];
}
	```
3. 雙層迴圈遍歷格子（填表）
	1. 我們依序走訪每一個格子。針對每個格子，計算當前的 Cost 增量（costInc），並檢查是從上方還是左方過來。
	```javascript
for (let i = 0; i < m; i++) {
  for (let j = 0; j < n; j++) {
    const costInc = grid[i][j] > 0 ? 1 : 0;

    // 走訪所有可能的累積 Cost, k: 得分上限
    for (let w = costInc; w <= k; w++) {
      let bestFromPrev = -1;

      // 檢查上方
      if (i > 0) {
        bestFromPrev = Math.max(bestFromPrev, dp[i - 1][j][w - costInc]);
      }
      // 檢查左方
      if (j > 0) {
        bestFromPrev = Math.max(bestFromPrev, dp[i][j - 1][w - costInc]);
      }

      // 如果前一步有路可達，更新目前格子的狀態
      if (bestFromPrev !== -1) {
        dp[i][j][w] = Math.max(dp[i][j][w], bestFromPrev + grid[i][j]);
      }
    }
  }
}
	```
4. 統計最終結果
	1. 當我們走完右下角 (m-1, n-1) 後，檢查所有符合條件的 Cost，找出最大分數。
	```javascript
let result = -1;
for (let w = 0; w <= k; w++) {
  result = Math.max(result, dp[m - 1][n - 1][w]);
}

return result;
	```

