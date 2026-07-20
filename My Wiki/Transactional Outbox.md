---
type: concept
status: growing
topics: [distributed-systems, transaction, messaging, reliability, outbox]
created: "2026-07-19"
---

# Transactional Outbox

## Summary

Transactional Outbox 在同一個 local database transaction 中寫入業務資料與待發布事件，再由獨立 dispatcher 重試發布，避免「database 已 commit、queue message 卻遺失」的 dual-write failure。

## Why It Matters

Database 與外部 queue 無法共用一般 ACID transaction。[[Full-stack Error Handling Interview Practice]] 比較 database-backed job 與 outbox 的適用時機：只有需要可靠發布到外部 queue 時才引入 outbox；若 worker 直接 polling job table，database 本身已是 durable source，可以先不使用。

## How It Works

```text
Database transaction:
  INSERT business row
  INSERT job row
  INSERT outbox event

Dispatcher:
  claim pending event
  publish to queue
  mark event published
```

Outbox row 保存 `event_id`、aggregate／job ID、event type、payload、status、attempt、`next_attempt_at`、lease 與 `published_at`。多個 dispatcher 使用 atomic claim 或 `SELECT ... FOR UPDATE SKIP LOCKED`，失敗後依 backoff 重試。

Outbox 描述的是「這個 event 必須被發布」，不是 background job 的完整執行狀態。兩者生命週期與 state machine 不應混在同一欄位。

## Example

Dispatcher 可能已成功 publish，卻在更新 outbox row 前 crash。重啟後同一 event 會再次發布，因此 consumer 應使用穩定 `event_id` 或 `job_id` 去重，並以原子狀態轉換確認工作仍可執行。

## Tradeoffs

- Outbox 關閉 database-to-queue 的遺失窗口，但不提供 exactly-once delivery。
- 合理保證是 at-least-once delivery + [[Idempotent Request Handling|idempotent processing]]。
- 新增 dispatcher、cleanup、backlog monitoring、schema evolution 與 payload compatibility 成本。
- 只在確實有外部 publish boundary 時使用；單一 PostgreSQL job table 的 MVP 不一定需要。

## Related

- [[Full-stack Error Handling Interview Practice]]
- [[Reliable Background Job Processing]]
- [[Idempotent Request Handling]]
- [[Distributed Transactions - 2PC and Saga]]
- [[System Design Foundations]]

## References

- [[2026-07-19 weekly updates]]
