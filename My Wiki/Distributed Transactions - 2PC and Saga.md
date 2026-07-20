---
type: concept
status: growing
topics: [system-design, distributed-systems, transactions, microservices]
created: "2026-07-14"
---

# Distributed Transactions: 2PC and Saga

## Summary

當一個業務操作跨越多個 Service 或 Database 時，單一資料庫的 ACID transaction 無法直接涵蓋全部步驟。Two-Phase Commit（2PC）追求協調式提交；Saga 則用一系列 Local transaction 與 Compensating transaction 達成最終一致性。

## Why It Matters

訂單、付款、庫存等流程若在中途失敗，系統可能留下部分完成的狀態。分散式交易策略決定系統如何在一致性、可用性、延遲與復原複雜度之間取捨。

## How It Works

### Two-Phase Commit

1. Coordinator 要求參與者 Prepare。
2. 所有參與者準備成功後才 Commit；任何一方失敗則 Rollback。

2PC 能提供較強的一致性，但 Coordinator 或 Participant 故障可能造成等待與資源鎖定。

### Saga

將長交易拆成多個 Local transaction。每一步成功後發布下一個動作；後續失敗時執行補償動作，例如取消訂單或退款。

- **Choreography**：服務透過事件彼此觸發，耦合較鬆，但流程分散、較難追蹤。
- **Orchestration**：由中央 Orchestrator 明確指揮步驟，流程較容易觀察，但 Orchestrator 會成為重要元件。
- **[[Transactional Outbox]]**：把業務資料與待發布事件寫入同一個 local transaction，再由背景程序可靠地發布；它解決 database-to-queue dual write，但仍需要 idempotent consumer 處理重複投遞。

## Example

```text
Create order
  → Reserve inventory
  → Charge payment
  → Confirm order

Payment failed
  → Release inventory
  → Cancel order
```

## Tradeoffs

- 若能合理共用資料庫，先避免不必要的分散式交易。
- 2PC 適合需要強一致性且參與系統支援協定的情境，但可用性與效能成本較高。
- Saga 更有彈性，但補償不等於真正 Rollback，必須處理 Idempotency、Retry、Ordering 與人工介入。

## Related

- [[System Design Foundations]]
- [[SQL vs NoSQL]]
- [[Transactional Outbox]]
- [[Idempotent Request Handling]]

## References

- [[2026-05-24 - 5-24 weekly update]]
- [[2026-07-19 weekly updates]]
- [Distributed Transactions Explained: 2 Phase Commit vs Saga Pattern](https://www.youtube.com/watch?v=DOFflggE_0Q)
