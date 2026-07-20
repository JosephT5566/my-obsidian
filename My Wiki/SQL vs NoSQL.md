---
type: concept
status: growing
topics: [database, sql, nosql, system-design]
created: "2026-07-14"
---

# SQL vs NoSQL

## Summary

SQL database 通常先定義資料關係與一致性限制；NoSQL database 更常從 Access pattern 出發設計資料形狀。選擇重點不是新舊，而是 Transaction、Query、Scale、Consistency 與維護需求。

在 PostgreSQL 內部也存在類似選擇：[[PostgreSQL Relational Data Modeling]] 比較 normalized table、historical snapshot 與 JSONB，說明彈性如何換取 database guarantees 與查詢便利性。

## Why It Matters

資料模型一旦和產品的讀寫模式不合，後續會出現複雜 Join、重複資料同步、昂貴跨節點協調或難以產生報表等問題。

## How It Works

### SQL

- 常使用 Normalization、Foreign key 與 Join。
- 適合交易正確性、複雜篩選、Aggregation、報表、權限與稽核。
- PostgreSQL、MySQL 等 RDBMS 通常提供成熟的 ACID transaction。

### NoSQL

- 包含 Key-value、Document、Wide-column 與 Graph 等不同模型。
- 常依 Query pattern 去 Denormalize 資料，讓單次讀取取得畫面需要的內容。
- 適合固定存取模式、即時同步、高寫入量或容易以單一 Document 表達的資料。
- 避免 Join 不代表沒有成本；重複資料會帶來同步與一致性問題。

## Example

即時協作名單只需要 `Get room` 與 `Update participant`，Document database 可能很自然。訂單、金流與跨多張表的報表通常更適合 SQL。

## Tradeoffs

- 小型即時 App 使用 Firestore 等 BaaS 可快速完成 Auth、Realtime listener 與 Hosting。
- 需求成長到搜尋、統計、報表、稽核與複雜權限後，SQL 往往更容易維護。
- SQL 也能水平擴展，NoSQL 也能支援 Transaction；不要只用「能不能擴展」做二分判斷。
- 大規模系統的選擇還要考慮 Hot partition、Latency、On-call toil 與 Migration cost。

## Related

- [[Firebase Realtime App Architecture]]
- [[Distributed Transactions - 2PC and Saga]]
- [[System Design Foundations]]
- [[PostgreSQL Relational Data Modeling]]
- [[PostgreSQL Schema Tradeoff Interview Practice]]

## References

- [[2026-06-07 - 6-7 weekly update]]
- [[2026-07-12 - 7-12 weekly update]]
- [[2026-07-19 weekly updates]]
