---
title: "6/7 weekly update"
date: 2026-06-07
tags: ["Interview 2026"]
source: "https://app.notion.com/p/3796ed462a2680fabf14d2896aac26e9"
notion_id: "3796ed462a2680fabf14d2896aac26e9"
---

# Wedding Table Service
6/4（四）看了 IG 的短影片提到可以用 AI 做一個婚禮報到 app，貼給建銘後他覺得會有幫助，於是就開始動工，預計工作日一天
我的計劃是，為了快速上線，一樣是採用 Github page + google sheep 做為資料庫，可能快速搭配一個 Google apps script。考慮到可能多裝置會使用這個 app，要在多裝置之間更新，可能會再搭配 webhook or polling (per 5s)
但跟 ChatGPT 討論一下架構，GAS 可行，但要再掛上 Google cloud webhook 就會需要額外的測試
最後選擇的是 Firebase （Backend as a Service） 提供的 Firestore + Firebase hosting
- 前端： deploy 在 Firebase 託管
- 資料庫：NoSQL Firestore
- Auth: Firebase 直接提供了 Google auth
- Firestore 提供 listen 功能，能實現多個 app 同步，並幫我們省去了 webhook 的設置
	- app 能使用 Firebase SDK、並通過 Firebase 專案的驗證與 Rules
# SQL vs NoSQL
開始使用 Firestore 後才開始感受到 NoSQL
>
	SQL 是先設計資料關係，再開發功能。
	NoSQL 是先設計讀寫流程，再決定資料怎麼存。
### 資料思維
SQL:
會先定義關聯，再設計語法讓 app 調用，資料通常會正規化。<br>我們會讓多個 Tables 負責不同的功能，查詢時透過 JOIN 來組合，管理時有 foreign key 確保 table 之間的資料一致
NoSQL:
則是從 app 出發思考畫面怎麼讀，API 怎麼查，然後設計資料欄位 (path)<br>相較於 SQL 在查詢時會用 JOIN 都組合好，NoSQL 不鼓勵在讀取時才去做關聯，而是在寫入時就把單一次查詢的資料都寫入
### 適合場景
SQL:
訂單系統、報表分析<br>需要 JOIN，需要 Group by, 報表，涉及金流
NoSQL:
聊天室、即時協作 (Google docs, Notion, Figma)、即時同步資料 (遊戲狀態、線上名單)<br>Query Pattern 一致，即時同步很重要，單一 Document 就能表達資料
> 所以我們更會思考，在這種使用情境下，什麼資料庫會更適合
NoSQL 在設計上資料重複，不 `JOIN`，弱化 Transaction，透過限制資料模型與查詢能力，避免大量跨節點協調，跨節點成本降得比較低。相較於 SQL 經常性的 JOIN，或是 Sharding 的規劃，在發生跨節點時，協調成本更高。（想像在交易執行時還要維持一致）
### 維護成本
小型專案：Firestore 通常比較輕鬆。
中大型專案：PostgreSQL 通常比較穩。因為當需求開始出現：搜尋、篩選、統計、報表、權限、稽核，SQL 會越來越舒服。
NoSQL 反而會開始出現：資料重複、資料同步問題、Aggregation 困難
### Migration Examples
**Discord** 早期訊息儲存從 MongoDB 轉到 Cassandra，後來又把大型 Cassandra messages cluster 遷到 ScyllaDB。原因不是「NoSQL 不好」，而是訊息量到 trillions 等級後，維護成本、hot partition、latency、on-call toil 都變成問題。
**Spotify** 曾把使用者資料從 PostgreSQL 遷到 Cassandra，原因：數億使用者、全球同步、高寫入量、高可用性。他們發現：User Profile 其實幾乎沒有 JOIN。常見操作只是：
```plain text
Get User
Update User
```
# Firebase rules
主要邏輯：<br>match xxx 定義針對 document path / collection path，allow xxx 當要進行對應操作時要檢驗後續的邏輯，後面再帶上 method 回傳是否允許執行
```javascript
match /weddings/{weddingId}/guests/{guestId} {
  allow read: if canRead(weddingId);
  allow create, update: if canEdit(weddingId);
}
```
Process:
1. request 進來
2. Firestore 找到符合的 match path
3. 看這次操作是 read/create/update/delete
4. 執行 allow if 條件
5. true → 允許<br>false → 拒絕
# ByteByteGo Takeaways
## 1. Scale From Zero To Millions Of Users
[System Design · Coding · Behavioral · Machine Learning Interviews](https://bytebytego.com/courses/system-design-interview/scale-from-zero-to-millions-of-users)
1. 單一伺服器設定 (Single server setup)
	1. 起步時最簡單的架構，將所有元件（Web 應用程式、資料庫、快取等）全都跑在**同一台伺服器**上。
	2. 使用者透過 DNS 解析網域名稱取得 IP 地址後，直接發送 HTTP 請求給這台伺服器，伺服器再回傳 HTML 或 JSON 回應。
2. 資料庫 (Database)
	1. 當單一伺服器不夠用時，**將網頁流量（Web Tier）與資料庫（Data Tier）分離**到不同的伺服器上，以便它們能各自獨立擴展。
	2. **資料庫選擇**：
		1. **關聯式資料庫 (RDBMS/SQL)**：如 MySQL、PostgreSQL，歷史悠久且支援強大的 <span color="red">Join 操作</span>。大多數為最佳選擇。
		2. **非關聯式資料庫 (NoSQL)**：如 DynamoDB、Cassandra，適合超低延遲、資料無結構、需要處理海量資料或單純 serialize and deserialize data (JSON, XML, YAML, etc.) 的場景。<br>分為 four categories: key-value stores, graph stores, column stores, and document stores.<br>無 Join 操作
3. 垂直擴展 vs 水平擴展 (Vertical scaling vs horizontal scaling)
	1. 垂直擴展（Scale up）：幫原本的伺服器升級更強的 CPU 和記憶體。流量不高時可以考慮垂直。但有硬體極限，且缺乏容錯機制（單點故障，SPOF）。
	2. 水平擴展（Scale out）：透過增加更多台伺服器來分擔流量，對大型應用而言是更理想的選擇。可以處理 failover （故障轉移） 和 redundancy （冗餘）
4. 負載平衡器 (Load balancer)
	1. 為了解決水平擴展後的流量分配與伺服器故障問題
	2. 使用者直接連線到負載平衡器的 public IP，負載平衡器再透過 Private IP 將流量**平均分配**給後端的 Web 伺服器。若有伺服器死機，流量會自動導向其他健康伺服器。
5. 資料庫複製 (Database replication)
	1. 解決資料庫的單點故障與讀寫瓶頸。
	2. 主資料庫（Master）只負責處理 insert, delete, or update 操作；從資料庫（Slave）複製主庫資料，只負責處理「讀取」操作。
	3. 讀取情境較多，所以 Slave 的數量也會較多
	4. 提升效能（讀寫分離）、提高可靠性與高可用性，可以實現並行讀取。若 Slave 故障，讀取會暫時由 Master 或其他 Slave 分擔；若 Master 故障，則會將一台 Slave 提升為新的 Master。（新的 master 的資料可能不會是最新，會需要再跑 data recovery scripts，可能會需要跟其他 slaves 互動，實際情況會更複雜點）
6. 快取 (Cache) 與 快取層 (Cache tier)
	1. 快取是一塊記憶體暫存區，用來儲存高成本的回應或頻繁存取的資料，避免一直去向慢速的資料庫查詢（通常採用 Read-through 策略）。
	2. 需要決定何時使用快取（讀多寫少）、制定過期策略（Expiration policy）、維和資料一致性（Consistency）、避免單點故障（Mitigating failures），以及選擇汰換機制（Eviction Policy，如 LRU、LFU、FIFO）。
7. 內容傳遞網路 (Content delivery network - CDN)
	1. **概念**：分散在世界各地的伺服器網路，用來**快取靜態內容**（如圖片、影片、CSS、JS 檔案）。
	2. **機制**：當使用者造訪網站，會由地理位置最近的 CDN 伺服器提供靜態檔案，大幅縮短載入時間。
	3. **考量**：需要注意流量成本、設定合理的過期時間（TTL）、準備 CDN 故障的退路（Fallback），以及必要時主動讓檔案失效（Invalidating files，如透過 API 或版本號變更 `image.png?v=2`）。
8. 無狀態網頁層 (Stateless web tier)
	1. **概念**：為了能自由地水平擴展 Web 伺服器，必須將用戶的「狀態」（例如 Session 連線狀態）移出 Web 伺服器。
	2. **狀態式 vs 無狀態**：狀態式架構需要將特定用戶綁定在特定伺服器上（Sticky sessions），難以擴展；無狀態架構則將狀態集中存在一個**共享的資料儲存區**（如 NoSQL、Redis），任何一台 Web 伺服器都能處理任何請求，進而能輕鬆做到**自動擴展（Autoscaling）**。
9. 資料中心 (Data centers)
	1. **概念**：當業務擴展到國際，需要支援多個地理位置獨立的資料中心。
	2. **機制**：透過 GeoDNS（地理位置 DNS）將使用者引導到最近的資料中心。若某個資料中心發生故障，100% 的流量會自動切換到另一個健康的資料中心。
	3. **挑戰**：需解決流量重導向、多中心間的資料同步（Data synchronization），以及跨區域的測試與自動化部署。
10. 訊息佇列 (Message queue)
	1. **概念**：一個存在於記憶體中、支援**非同步通訊**的耐用元件。
	2. **架構**：生產者（Producers）建立訊息並發送到佇列；消費者（Consumers）連線到佇列並執行對應的任務。
	3. **優點**：達到系統**解耦（Decoupling）**。例如照片裁切/模糊等耗時任務，Web 伺服器只需丟進佇列（發布），再由後端的 Worker 異步處理（消費），雙方能各自獨立擴展。
11. 日誌、指標、自動化 (Logging, metrics, automation)
	1. **日誌 (Logging)**：集中化監控錯誤日誌，以便快速定位系統問題。
	2. **指標 (Metrics)**：收集主機層級（CPU/記憶體）、架構層級（資料庫/快取效能）及業務層級（DAU/留存率）的數據來掌握系統健康度。
	3. **自動化 (Automation)**：利用 CI/CD 工具自動化建置、測試與部署，提升開發效率。
12. 資料庫擴展 (Database scaling)
	1. **垂直擴展**：買更貴、硬體更好的伺服器，但有硬體極限、高成本與單點故障風險。
	2. **水平擴展（分片 - Sharding）**：將大資料庫拆分成更小、更好管理的**分片（Shards）**。通常透過一個分片鍵（Sharding key，例如 `user_id % 4`）來決定資料存在哪一個分片。
	3. **分片挑戰**：資料量暴增時需要重新分片（Resharding）、名人熱點問題（Celebrity problem，某些分片流量過高）、以及跨分片難以執行 Join 操作（需透過反正規化 De-normalization 解決）。

