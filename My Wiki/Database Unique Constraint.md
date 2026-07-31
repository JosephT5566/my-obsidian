---
type: concept
status: growing
topics: [database, postgresql, constraints, concurrency, data-integrity]
created: "2026-07-27"
---

# Database Unique Constraint

## Summary

Unique constraint 是 database schema 的完整性規則，用來保證一個欄位或一組欄位的值不會在 table 中重複。它不只檢查一般 insert，也會仲裁 concurrent writes，因此比 Application 先查詢「是否存在」更適合保護 uniqueness invariant。

## Why It Matters

Application validation 可以提供友善錯誤，但任何 API instance、background job、migration 或 SQL console 都可能寫入資料。若「同一個 expense 不能重複加入同一位 participant」或「同一個 scoped [[Idempotency Key|idempotency key]] 只能有一個 owner」是所有寫入路徑都必須遵守的規則，就應由 database constraint 作為最後一道 correctness boundary。

這也是 [[PostgreSQL Relational Data Modeling]] 優先把核心資料正規化的理由之一：一般 table column 可以直接使用 unique、foreign key 與 check constraint，而 JSONB element 通常無法直接取得相同的 database guarantee。

## How It Works

### Single-column and Composite Uniqueness

Single-column constraint 要求一個欄位不能重複：

```sql
CREATE TABLE users (
  id    bigint PRIMARY KEY,
  email text NOT NULL UNIQUE
);
```

Composite constraint 要求「欄位組合」不能重複，但每個欄位本身仍可重複：

```sql
CREATE TABLE expense_participants (
  expense_id bigint NOT NULL,
  user_id    bigint NOT NULL,
  amount     numeric NOT NULL,
  CONSTRAINT uq_expense_participant
    UNIQUE (expense_id, user_id)
);
```

`expense_id = 10` 可以有多個不同 user，`user_id = 7` 也可以參與多個 expense；只有 `(10, 7)` 這個組合不能出現兩次。因此 composite key 的欄位必須精準對應 business scope，不能只挑看起來「像 ID」的欄位。

### Database Arbitration

PostgreSQL 建立 unique constraint 時，會建立對應的 unique B-tree index。Insert 或 update 必須通過這個 index 的 uniqueness check；若兩個 transaction 同時寫入相同 key，database 會協調它們的結果，而不是讓兩邊都因為先前查不到資料就成功。

概念流程如下：

```text
Transaction A ─┐
               ├─ 寫入相同 unique key → database 仲裁
Transaction B ─┘

A commit   → B 發生 unique conflict
A rollback → B 可以繼續成功
```

哪個 transaction 先成功不應成為 business logic 的假設。Application 只應依結果判斷自己是建立者、遇到既有資料，或需要回報 conflict。

### Constraint and Unique Index

兩者密切相關，但用途不完全相同：

- `UNIQUE (...)` constraint 宣告 table 必須遵守的 data invariant，並自動建立支撐它的 unique index。
- `CREATE UNIQUE INDEX` 直接建立 index；PostgreSQL 可用 expression 或 `WHERE` predicate 表達 constraint syntax 無法直接描述的規則。
- Partial unique index 只限制符合條件的 rows，例如每位 user 同時只能有一筆 active subscription：

```sql
CREATE UNIQUE INDEX uq_one_active_subscription
ON subscriptions (user_id)
WHERE status = 'active';
```

如果規則是單純的欄位或欄位組合唯一，優先用 named unique constraint 表達 schema intent；需要「只有部分 rows」或 expression-based uniqueness 時，再評估 unique index。

### Conflict Handling

單純 `INSERT` 違反 constraint 時，PostgreSQL 會拒絕 statement。若 duplicate 是預期中的 concurrency outcome，可以使用 constraint 或 unique index 作為 `ON CONFLICT` 的 arbiter：

```sql
INSERT INTO expense_participants (
  expense_id,
  user_id,
  amount
)
VALUES (10, 7, 500)
ON CONFLICT (expense_id, user_id) DO NOTHING
RETURNING *;
```

- `RETURNING` 有 row：本次 insert 成功。
- 沒有 row：既有資料贏得 uniqueness，Application 應依 contract 查回既有 row 或回報 conflict。
- `DO UPDATE`：將 conflict 轉成 atomic upsert，但只有在「重複時確實應更新」符合業務語意時才能使用，不能把所有 uniqueness error 都自動覆寫。

[[Idempotent Request Handling]] 會利用這個特性，讓 database 對同時抵達的 requests 仲裁 winner；unique constraint 保證只有一筆 scoped record，而 request hash 與保存的 response 再決定 loser 是合法 retry 還是 key misuse。

### NULL Semantics

在 PostgreSQL 中，unique constraint 預設把 `NULL` 視為彼此不同，因此 nullable unique column 通常可以保存多筆 `NULL`。若規則要求 `NULL` 也只能出現一次，可以使用 `NULLS NOT DISTINCT`。

```sql
UNIQUE NULLS NOT DISTINCT (external_reference)
```

不同 database 對 nullable uniqueness 的行為可能不同。設計 schema 時應明確決定 column 是否允許 `NULL`，並確認目標 database 的語意，而不是假設 `UNIQUE` 自動等於 `NOT NULL`。Primary key 才同時包含 uniqueness 與 non-null requirement。

## Example

Idempotency record 常使用 composite unique constraint：

```sql
CONSTRAINT uq_idempotency_scope
  UNIQUE (tenant_id, operation, idempotency_key)
```

這表示不同 tenant 可以碰巧產生相同 Client key，同一個 tenant 也能在不同 operation 重用該值；只有完整 scope 相同時才視為同一個 logical operation。具體的 winner／loser 與 payload validation 流程見 [[Idempotency Key]]。

## Tradeoffs

- Unique constraint 會增加 index storage，insert／update 也要維護 index；這是換取一致性與併發安全的成本。
- 它只能比較同一個 table 中可索引的 key，不能直接保證跨多 rows 的加總、跨 table 的複雜規則或 external provider side effect。
- Constraint 保證 key 不重複，不會自動決定 conflict 應回既有結果、更新 row，還是回 error；這仍是 API 與 business contract 的責任。
- Email 等文字值可能需要先定義大小寫、空白與 Unicode normalization，否則 database 會依實際儲存值判斷是否重複。
- 對已有資料的 table 新增 constraint 前，必須先找出並處理 historical duplicates，否則 migration 會失敗。
- Soft delete 若要求「只有未刪除資料必須唯一」，通常需要 partial unique index，而不是把 nullable `deleted_at` 直接加入 composite constraint。

## Related

- [[Idempotency Key]]
- [[Idempotent Request Handling]]
- [[PostgreSQL Relational Data Modeling]]
- [[PostgreSQL Row-Level Security]]
- [[PostgreSQL Schema Tradeoff Interview Practice]]
- [[SQL vs NoSQL]]

## References

- [PostgreSQL: Unique Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-UNIQUE-CONSTRAINTS)
- [PostgreSQL: INSERT and ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
- [[2026-07-26 weekly updates]]
