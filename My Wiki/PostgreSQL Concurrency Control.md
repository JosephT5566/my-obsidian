---
type: concept
status: growing
topics: [postgresql, database, concurrency, transactions, data-integrity]
created: "2026-08-03"
---

# PostgreSQL Concurrency Control

## Summary

PostgreSQL concurrency control 的核心是先定義不可被破壞的 business invariant，再用 database constraint、atomic SQL、row lock 或 transaction isolation 讓所有 concurrent writers 接受同一個仲裁結果，而不是只依賴 Application 的「先查再改」。

## Why It Matters

庫存不能為負、餘額不能超扣、限量資源不能超發，以及同一 job 不能被多個 worker 同時取得，都是在高併發下仍必須成立的 invariant。若 Application 先 `SELECT`、在記憶體判斷、再 `UPDATE`，兩個 request 可能根據同一份舊資料同時做出通過決定。

[[Database Unique Constraint|UNIQUE constraint]]、`CHECK`、foreign key 與 exclusion constraint 適合直接表達資料規則；無法單靠 constraint 表達的狀態轉換，再使用 conditional update、lock 或 transaction。

## How It Works

### Atomic Conditional Update

把條件與修改放在同一個 statement，讓 PostgreSQL 取得 row lock 後依最新資料重新判斷 `WHERE`：

```sql
UPDATE coupon_campaigns
SET claimed_count = claimed_count + 1
WHERE id = $1
  AND claimed_count < capacity
RETURNING claimed_count;
```

有 row 代表成功；0 rows 代表條件不成立或 resource 不存在，Application 必須檢查 `RETURNING`／affected rows，而不能只看 statement 是否拋錯。同樣模式可用於 `stock > 0`、`balance >= amount` 或 `status = 'pending'`。

### Pessimistic Row Lock

需要先讀取多個欄位、做複雜判斷或執行多步驟狀態轉換時，可以在短 transaction 中使用：

```sql
BEGIN;

SELECT capacity, claimed_count
FROM coupon_campaigns
WHERE id = $1
FOR UPDATE;

-- 依取得 lock 後讀到的最新資料判斷並更新

COMMIT;
```

`FOR UPDATE` 鎖住選中的 rows，不是整張 table。競爭同一 row 的 `FOR UPDATE`、`UPDATE` 或 `DELETE` 會等待；等待者取得 lock 後會依目前可見的最新 committed row 繼續，不能沿用 transaction 外的舊判斷。

### Transaction Atomicity

需要一起成功或一起失敗的 database writes 應放在同一 transaction。例如先以 conditional update 保留券，再建立 claim；若 claim 違反 unique constraint，整個 transaction rollback，不會留下已加計數但沒有 claim 的狀態。

Transaction 只能原子化同一 database boundary。Database commit 與外部 message publish 的 dual write 應使用 [[Transactional Outbox]]；external provider side effect 則需要 idempotency 或 reconciliation。

### Optimistic Locking

衝突不常見、但不能接受 silent overwrite 的資料，可使用 version column：

```sql
UPDATE orders
SET note = $1,
    version = version + 1
WHERE id = $2
  AND version = $3;
```

0 rows 代表版本已變，Application 應提示重新讀取或依 contract 合併，不應直接覆蓋。

### Worker Claiming

[[Reliable Background Job Processing|多個 worker]] 要同時取得不同工作時，可在 transaction 中以 `FOR UPDATE SKIP LOCKED` 跳過已被其他 worker 鎖住的 rows。Claim 仍需搭配 `locked_by`、lease、retry count 與 crash recovery；`SKIP LOCKED` 只處理競爭取得，不處理 owner crash。

### Specialized Locks and Constraints

- Advisory lock：沒有自然 row 可鎖時，以 application-defined key 序列化同一 user、tenant 或 report 的操作；所有 writer 都必須遵守相同 convention。
- Partial unique index：只對符合 predicate 的 rows 保證唯一，例如每位 user 只能有一筆 active subscription。
- Exclusion constraint：拒絕相同 room 的時間範圍重疊，比 Application 先查再 insert 更能抵抗 race condition。
- `SERIALIZABLE`：複雜跨多 rows invariant 無法以單一 constraint 或 lock 清楚表達時，由 PostgreSQL 偵測危險的 read/write dependency；Application 必須能 retry 整個 transaction。

## Example

選擇機制時，可由最直接的 database guarantee 開始：

```text
防止重複                 → UNIQUE / partial unique index / ON CONFLICT
防止超賣、負數或超額     → conditional UPDATE + RETURNING / CHECK
防止 lost update          → version column / optimistic locking
先讀後做複雜判斷         → SELECT ... FOR UPDATE
多 worker 領不同工作      → FOR UPDATE SKIP LOCKED
沒有自然 row 可鎖         → advisory lock
跨多 rows 的複雜規則      → SERIALIZABLE + whole-transaction retry
```

## Tradeoffs

- Conditional update 最精簡，但複雜 decision tree 可能較難放進單一 statement。
- Row lock 容易推理，熱門單一 row 卻可能成為 contention hotspot；transaction 也必須保持短小。
- Optimistic locking 避免長時間持鎖，但將 conflict handling 交給 Application 與使用者流程。
- Advisory lock 不會自動連結資料完整性；遺漏任何一條寫入路徑就可能繞過規則。
- `SERIALIZABLE` 能保護更複雜的 invariant，但 serialization failure 是正常結果，必須有 bounded retry。
- Constraint、atomicity 與 idempotency 負責不同問題，不能互相替代。

## Related

- [[Database Unique Constraint]]
- [[PostgreSQL Relational Data Modeling]]
- [[High-Concurrency Coupon Claim Interview Practice]]
- [[Idempotent Request Handling]]
- [[Reliable Background Job Processing]]
- [[Transactional Outbox]]

## References

- [[2026-08-02 weekly updates]]
