---
title: "6/14 weekly update"
date: 2026-06-14
tags: ["Interview 2026"]
source: "https://app.notion.com/p/37d6ed462a268072b274e2d13f50d3ec"
notion_id: "37d6ed462a268072b274e2d13f50d3ec"
---

# How to use the Codex remotely
<callout icon="⚠️" color="gray_bg">
	結果發現，即使筆電不蓋上，黑畫面經過一段時間後一樣會進入睡眠，因此可能要設置關閉睡眠
</callout>
考慮接下來會出國一週且應該不會攜帶電腦，如何持續遠端使用 mbp 中的 codex skill？
1. 使用 ChatGPT app 中的 Codex 功能，~~只要筆電不蓋上，即使是睡眠後似乎都能成功連線~~
2. 包裝成 Github action [https://github.com/JosephT5566/my-actions-runner](https://github.com/JosephT5566/my-actions-runner)
	1. 前提也是，只要筆電不蓋上，即使是睡眠後似乎都能成功呼叫
	2. 搭配 Github action 的 **self-hosted runner**，讓 action 呼叫電腦的 cli
	3. 也能配合 action **cron** 實現週期自動呼叫 (refer to mac `cron` or `launchd`)
	4. 缺點：有些 skill 只有在 Codex app 才能使用，像是 Chrome plugin。在 Codex cli 上面呼叫似乎不太穩定，會需要來回確認權限，透過 `Chrome-devtools-mcp` 才比較穩定
# ByteByteGo Takeaways
## 2. Back-of-the-envelope calculation
[System Design · Coding · Behavioral · Machine Learning Interviews](https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation)
<callout icon="💡" color="gray_bg">
	Assumptions → Calculations → Capacity → Architecture decisions
</callout>
- 目標是確認設計在數量級上合理，不是求精確答案。
- 面試官更重視拆解過程、假設和推理。
- 估算結果應該能影響架構決策，例如是否需要快取、分片或物件儲存。

1. Power of Two
	1. 熟悉 KB → MB → GB → TB → PB 的換算。
	2. 面試心算通常可以使用：
		1. 1 KB ≈ 10³ bytes
		2. 1 MB ≈ 10⁶ bytes
		3. 1 GB ≈ 10⁹ bytes
		4. 1 TB ≈ 10¹² bytes
		5. 1 PB ≈ 10¹⁵ bytes
	3. 最重要的是不要弄錯數量級。
	4. 所有計算都要標記單位，避免把 MB/day 和 GB/second 混在一起。
2. Latency Numbers
	1. 不必背每個數字，但要記住速度階層：
	2. CPU cache \< RAM \< SSD/disk \< datacenter network \< cross-region network
	3. 主要 takeaway：
		1. 記憶體比磁碟快很多，因此熱資料適合放快取。
		2. 隨機磁碟存取昂貴，應減少 disk seek。
		3. 網路呼叫不是免費的，尤其是跨區域請求。
		4. 多個依序執行的遠端呼叫會累積延遲。
		5. 壓縮通常比傳輸大量未壓縮資料便宜。
	4. 設計含義：使用 cache、batching、compression、CDN，以及減少 network round trips。
3. Availability Numbers
	1. 可用性每增加一個 9，容許的停機時間就大幅下降。
	2. 最值得記住：
		1. 99.9%   ≈ 每年 8.76 小時<br>99.99%  ≈ 每年 52.6 分鐘<br>99.999% ≈ 每年 5.26 分鐘
	3. 高可用性不是免費的，通常需要備援、複寫、自動故障轉移和跨區部署。
	4. 不要一律承諾 five nines；應根據產品需求和成本選擇 SLA。
	5. 系統總可用性會受到依賴服務影響。多個服務串聯可能降低整體可用性。
4. Twitter QPS Example
	1. 估算流程比答案重要：
	2. MAU → DAU → 每日操作量 → 平均 QPS → Peak QPS
	3. Takeaways：
		1. QPS = 每日請求量 ÷ 86,400
		2. 平均 QPS 不足以規劃容量，必須估算 Peak QPS。
		3. 範例假設尖峰是平均值的兩倍，但實際倍率應依產品特性決定。
		4. 必須分開估算不同操作，例如發文、讀取 timeline、搜尋和上傳媒體。
		5. 社群系統通常 read-heavy，總讀取流量可能遠高於發文 QPS。
5. Twitter Storage Example
	1. 通用流程：
	2. 每日資料筆數 × 每筆大小<br>→ 每日儲存量<br>→ 保存期限總容量
	3. Takeaways：
		1. 大型系統中，媒體通常比文字占用更多空間。
		2. 儲存估算必須包含 retention period。
		3. 文章算出的 55 PB 只是原始媒體容量。
		4. 真實需求還要加入：
			1. replication
			2. backups
			3. thumbnails/transcoding
			4. metadata與索引
			5. 預留容量
		5. 因此實際採購容量可能是原始估算的數倍。
6. Tips: 這是面試最實用的一節：
	1. 使用近似值：99,987 ÷ 9.1 可以簡化成 100,000 ÷ 10。
	2. 寫下假設：讓面試官可以檢查、修正並跟上推理。
	3. 標記單位：每一步都寫 requests/s、TB/day 等。
	4. 先平均、再尖峰：不要只算平均流量。
	5. 檢查結果：確認單位和數量級合理。
	6. 把結果連回設計：例如 55 PB 表示不能使用一般關聯式資料庫儲存媒體，應考慮 object storage。

