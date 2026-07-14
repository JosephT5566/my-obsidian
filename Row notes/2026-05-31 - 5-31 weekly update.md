---
title: "5/31 weekly update"
date: 2026-05-31
tags: ["Interview 2026"]
source: "https://app.notion.com/p/36b6ed462a2680bbaf1cc4a484c93b5a"
notion_id: "36b6ed462a2680bbaf1cc4a484c93b5a"
---

## DFS vs BFS
### BFS 特徵
1. 從某個位置，可以走到很多下一步
2. 問能不能到達
3. 狀態會擴散，目前位置 → 下一批位置
4. 需要避免重複訪問
常見模板
```javascript
queue = [start]

while (queue.length) {
    curr = queue.shift()

    for (next of neighbors(curr)) {

        if (!visited[next]) {
            visited[next] = true
            queue.push(next)
        }
    }
}
```
### DFS 特徵
<mention-page url="https://app.notion.com/p/3506ed462a268087ae24d5620dfc571a"/>
1. 探索所有可能路徑
2. 做選擇 → 遞迴 → 回復狀態。Backtracking
3. 題目有「嘗試」
	1. try all possibilities
	2. all combinations
	3. all subsets
	4. every path
4. Tree traversal
5. Recursive structure 很明顯
常見模板
```javascript
function dfs(node) {

    // 終止條件
    if (...) return;

    // 做事
    ...

    // 遞迴下一層
    for (next of neighbors) {
        dfs(next);
    }
}
```
## 如何用AI面試
[一年被裁員兩次 我是怎麼在美國活下來的 AI可以幫忙找工作嗎](https://youtu.be/sqnM28p7H9E?si=PWtLbcZwczugoING)
- 建立一個 Stories bank 並套用 **STAR 原則**
	- 可以使用 **Whisperflow** 等工具，直接對著電腦說話來記錄想法
	- 針對常見的行為面試問題準備素材，包括：
		- 遇到的**最困難挑戰**以及如何克服它。
		- 如何與不同職能的人員（如**工程師**）進行溝通。
		- 如何在工作中使用 **AI 工具**。
		- 各種處理衝突或特定任務的經驗
	- **STAR 原則**
		- **S (Situation) 情境：** 描述當時發生的背景狀況。
		- **T (Task) 任務：** 說明當時你面臨的具體目標或挑戰。
		- **A (Action) 行動：** 詳細描述你採取了哪些具體動作來解決問題。
		- **R (Result) 結果：** 說明最終產出的正面效益或量化結果
	- 每當有新的面試通知時，可以從故事庫中挑選合適的故事，請 AI 根據該公司的 **JD** 與當前挑戰進行微調，讓故事與職位需求更匹配
- 使用錄音 ai 紀錄每次面試的過程並持續
	-  使用 **Granola **進行面試錄音。優點是本地錄製，不會有 AI 機器人出現在視訊會議中，並能在結束後提供完整的**面試逐字稿 (Transcription)**
	- 將面試逐字稿輸入 AI 工具（如 Claude），並下達**明確指令 (Prompt)** 要求 AI 提供直白、不誇獎、具體且可執行的改進建議
	- AI 能從面試對話中分析出你還需要補強的 Homework 或術語 (Terminology)

