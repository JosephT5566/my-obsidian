---
type: concept
status: growing
topics: [postgresql, database, normalization, jsonb, snapshots]
created: "2026-07-19"
---

# PostgreSQL Relational Data Modeling

## Summary

穩定且重要的關聯資料應優先以 table、foreign key、型別與 constraint 建模；需要保留歷史語意時，再加入名稱或價格等明確 snapshot。JSONB 適合形狀仍不穩定、很少跨 record 查詢且總是整體讀寫的資料。

## Why It Matters

Schema 不只決定資料怎麼存，也決定非法資料在哪裡被阻止、查詢是否自然，以及 concurrent writes 能否安全處理。[[PostgreSQL Schema Tradeoff Interview Practice]] 以多人記帳情境練習 relation、snapshot 與 JSONB tradeoff，但不代表任何專案已採用該 schema。

## How It Works

### Relation 與 Source of Truth

Foreign key 保存 entity relationship，例如 expense 目前屬於哪個 category。欄位型別、`CHECK` 與 `UNIQUE` 可由 PostgreSQL 對所有寫入路徑一致執行，不會被 SQL console、migration 或 background job 繞過。

### Historical Snapshot

Snapshot 保存事件發生當時的語意，不能用相似但不同的欄位代替。例如 `expense_name` 是消費名稱；`category_name_snapshot` 才能保存當時的分類名稱。多對多情境的 snapshot 應放在 join row，讓每個 allocation 都保有自己的歷史資料。

### JSONB

PostgreSQL 能用 `jsonb_array_elements` 展開 array、轉型與 aggregation，也能以 trigger 執行複雜驗證。但 JSONB array 內部無法直接使用一般 foreign key，手動重建這些保證會增加權限、效能與測試成本。

## Example

```sql
SELECT COALESCE(SUM((share->>'amount')::numeric), 0)
FROM jsonb_array_elements($1::jsonb) AS share;
```

若 share 是核心資料，normalized row 通常更合適：`expense_id` 與 `user_id` 使用 foreign key，`amount` 使用 `numeric` 與 `CHECK`，再用 `UNIQUE (expense_id, user_id)` 防止重複 participant。

## Tradeoffs

- Normalization 提供完整性、細粒度更新、索引與跨 record 查詢，代價是更多 tables、joins 與 migrations。
- Snapshot 是刻意的 denormalization；它保留歷史，但要明確定義建立時機與不可回填的語意。
- JSONB 提供 schema flexibility，代價是型別、關係、更新衝突與查詢複雜度。
- Middleware 適合 request shape 與友善錯誤；database constraint 保護核心 invariant；trigger 處理一般 constraint 無法表達、但所有寫入路徑都必須成立的規則。
- 跨多 rows 的寫入使用 transaction 與必要的 row lock，避免只完成一部分或在 concurrent update 下發生 lost update。

## Related

- [[PostgreSQL Schema Tradeoff Interview Practice]]
- [[SQL vs NoSQL]]
- [[Supabase]]
- [[Distributed Transactions - 2PC and Saga]]

## References

- [[2026-07-19 weekly updates]]
