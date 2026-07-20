---
type: concept
status: growing
topics: [api, idempotency, concurrency, database, distributed-systems]
created: "2026-07-19"
---

# Idempotent Request Handling

## Summary

Idempotency 讓同一個 logical operation 即使因 timeout、重複點擊或 at-least-once delivery 被執行多次，也只產生一個可觀察的業務結果。Frontend disable button 只改善 UX；正確性必須由 Server 與 database atomicity 保護。

## Why It Matters

Client 不知道 timeout 發生時 Server 是否已 commit，多台 Server 也可能同時收到同一 request。[[Async API Contract Interview Practice]] 與 [[Full-stack Error Handling Interview Practice]] 分別從 API contract 和 production failure 角度練習這個問題。

## How It Works

Idempotency record 通常綁定：

- authenticated user 或 tenant
- endpoint／operation
- idempotency key
- canonical request payload hash
- 建立出的 resource ID
- response status 或重建 response 所需資料
- retention／expiry time

對 `(user_id, endpoint, idempotency_key)` 建立 unique constraint。不要使用「先 `SELECT`、不存在才 `INSERT`」的 check-then-act；多個 instance 可能同時看到不存在。應以 atomic insert 或 `INSERT ... ON CONFLICT DO NOTHING RETURNING ...` 讓 database 仲裁 winner。

處理規則：

1. 新 key：在同一 transaction 建立 idempotency record 與業務 resource。
2. 相同 key、相同 payload：回既有 resource 與目前狀態，不再次執行。
3. 相同 key、不同 payload：回 `409 IDEMPOTENCY_CONFLICT`。
4. 第一次工作仍在執行：回同一個 job，而不是等待或建立第二個 job。

## Example

```sql
UNIQUE (user_id, endpoint, idempotency_key)
```

若外部 provider 也有 side effect，internal key 不能自動保護該 boundary。應把穩定 job ID 傳為 provider idempotency key，或保存 provider async job ID 供 reconciliation。

## Tradeoffs

- 需要保存 payload fingerprint、resource linkage 與 expiry policy。
- exactly-once execution 通常無法跨 database、queue 與 provider 證明；實務上採 at-least-once delivery 加 idempotent processing。
- Result table 對 `job_id` 設 unique constraint，只能避免本地重複結果，不能阻止 provider 被重複呼叫。
- 每個 side-effect boundary 都要個別選擇 idempotency、deduplication 或 reconciliation 策略。

## Related

- [[Async API Contract Interview Practice]]
- [[Full-stack Error Handling Interview Practice]]
- [[API Contract Design]]
- [[Reliable Background Job Processing]]
- [[Transactional Outbox]]

## References

- [[2026-07-19 weekly updates]]
