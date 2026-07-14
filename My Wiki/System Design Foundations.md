---
type: learning-note
status: growing
source:
  - "[[2026-06-07 - 6-7 weekly update]]"
  - "[[2026-07-12 - 7-12 weekly update]]"
topics: [system-design, scalability, reliability, architecture]
created: "2026-07-14"
---

# System Design Foundations

## Source

- ByteByteGo: Scale From Zero To Millions Of Users
- 900+ hours of Learning System Design in 9 Minutes

## Key Ideas

### 從單機逐步擴展

1. 先用 Single server 驗證產品。
2. 將 Web tier 與 Data tier 分離，讓兩者獨立擴展。
3. 使用 Load balancer 配合 Horizontal scaling 與 Health check。
4. 以 Database replication 分擔讀取並提供 Failover，但要理解 Replication lag。
5. 將 Session 移到共享儲存，建立 Stateless web tier。
6. 用 Cache 與 CDN 降低延遲和下游負載，同時管理 Stale data、TTL 與 Invalidation。
7. 用 Message queue 將可延後的工作非同步化，並處理 Retry、Idempotency 與 Ordering。
8. 以 Metrics、Logging、Automation 與 Capacity planning 支援營運。
9. 當單一資料庫無法承載時才評估 Sharding，並準備 Resharding、Hot partition 與跨分片查詢問題。

### 核心 Tradeoffs

- **Statelessness**：方便替換與水平擴展 Server，但狀態要移到 Redis、Database 或 Client token。
- **Caching**：用可能過期的資料換取速度；常見策略有 Cache-aside、Write-through 與 Write-back。
- **CAP**：發生 Network partition 時，需要按操作選擇偏向 Consistency 或 Availability。
- **Message queue**：降低同步耦合，但引入 Eventual consistency 與失敗處理。
- **SQL / NoSQL**：依資料保證與 Access pattern 選擇，不以流行度決定。
- **API design**：API 是對 Client 的契約，需考慮 Resource model、Versioning 與 Backward compatibility。

## Examples

- 金流與權限通常偏向 Strong consistency。
- Feed 與非關鍵狀態可接受 Eventual consistency。
- 圖片處理可先寫入 Queue，再由 Worker 非同步處理。
- 靜態資源可使用 CDN；熱門讀取可使用 Application cache。

## Questions

- 哪些狀態仍保存在單一 Application server？
- 哪些資料可接受過期，最長多久？
- 哪些操作必須同步完成？
- 依賴服務的可用性如何影響整體 SLA？
- Sharding 是否真的必要，還是垂直擴展與 Read replica 已足夠？

## How This Changes My Work

從瓶頸、資料保證與失敗模式開始設計，再選擇元件。每新增一個 Cache、Queue、Replica 或 Data center，都要說明它解決的問題、帶來的成本與驗證方式。

## Related

- [[Back-of-the-envelope Estimation]]
- [[Distributed Transactions - 2PC and Saga]]
- [[SQL vs NoSQL]]

## References

- [[2026-06-07 - 6-7 weekly update]]
- [[2026-07-12 - 7-12 weekly update]]
- [ByteByteGo System Design](https://bytebytego.com/courses/system-design-interview)
- [900+ hours of Learning System Design in 9 Minutes](https://www.youtube.com/watch?v=3Pusamd6BO4)

