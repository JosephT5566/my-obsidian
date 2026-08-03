---
type: learning-note
status: growing
source:
  - "[[2026-08-02 weekly updates]]"
topics: [backend, interview, system-design, postgresql, concurrency, idempotency]
created: "2026-08-03"
---

# High-Concurrency Coupon Claim Interview Practice

## Source

2026-08-02 的 Backend interview 練習：設計一個同時面對大量請求、重複點擊與 ambiguous HTTP timeout，仍不能超發 10,000 張券或讓同一 user 重複領取的系統。這是架構推演，不代表既有專案已實作。

## Key Ideas

先列出 invariant，再分配給不同保證機制：

1. `claimed_count <= capacity`：由 [[PostgreSQL Concurrency Control|conditional atomic update 或 campaign row lock]] 仲裁。
2. 同一 `(campaign_id, user_id)` 只能有一筆 claim：由 [[Database Unique Constraint|composite unique constraint]] 保護。
3. 相同 API operation 安全重送：由 [[Idempotency Key|scoped idempotency key]]、request hash 與 response replay 處理。
4. 庫存、claim 與 operation result 必須一致：在同一 database transaction 一起 commit 或 rollback。

Idempotency 與業務唯一性不能互相替代：相同 key 的重送由 idempotency record 吸收；同一 user 使用不同 keys 重複領券，仍必須由 `(campaign_id, user_id)` uniqueness 擋下。

## Examples

核心 schema invariants：

```sql
UNIQUE (campaign_id, user_id)
UNIQUE (user_id, operation, idempotency_key)
```

庫存可以鎖住指定 campaign row，取得 lock 後重新檢查最新值；也可以在 transaction 中原子保留名額：

```sql
UPDATE coupon_campaigns
SET claimed_count = claimed_count + 1
WHERE id = $1
  AND claimed_count < capacity
RETURNING id;
```

接著建立 claim。若 unique conflict 表示 user 已領過，transaction 必須 rollback 前面的計數更新；結束失敗 transaction 後再查詢既有 claim，轉成穩定業務結果，而不是把 raw constraint error 暴露給 Client。

可信 API 結果可包含：

- `claimed`
- `already_claimed`
- `sold_out`
- `in_progress`
- `idempotency_key_reused`

HTTP response 遺失不代表 database transaction 失敗。Client 應使用相同 key retry 或查詢 claim 狀態；Backend 應回傳 authoritative database result，不再扣除一次庫存。

## Questions

- 哪些 invariant 是 per-request、per-user、per-campaign 或跨 rows？
- Conditional update 的 0 rows 如何區分活動不存在與 sold out？
- 熱門 campaign row 成為 contention hotspot 時，正確性與 throughput 要如何量測和取捨？
- Unique conflict 發生後，如何正確離開 aborted transaction 再查詢既有 claim？

## How This Changes My Work

面試回答依「business invariant → database guarantee → API behavior」展開。精確說明 `FOR UPDATE` 的 row-level lock granularity、transaction rollback、不同 idempotency keys 的業務重複，以及 commit 成功但 response 遺失時的 recovery；不要為同步短 transaction 過早加入 async queue。

## Related

- [[Backend Engineering Interview Practice]]
- [[PostgreSQL Concurrency Control]]
- [[Database Unique Constraint]]
- [[Idempotency Key]]
- [[Idempotent Request Handling]]
- [[API Contract Design]]

## References

- [[2026-08-02 weekly updates]]
