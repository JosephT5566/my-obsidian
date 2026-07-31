---
type: concept
status: growing
topics: [postgresql, supabase, rls, authorization, multi-tenancy, security]
created: "2026-07-28"
---

# PostgreSQL Row-Level Security

## Summary

PostgreSQL Row-Level Security（RLS）是 database 層的 row authorization：policy 決定目前角色可以對哪些 existing rows 執行 `SELECT`、`UPDATE`、`DELETE`，以及哪些 new rows 可以被 `INSERT` 或由 `UPDATE` 形成。RLS 保護的是 row access boundary，不取代 request validation、column privilege、business workflow 或 data integrity constraint。

## Why It Matters

[[Supabase]] 讓 authenticated Client 能直接使用自動產生的 data API，因此 [[Expense App]] 不能只靠 Frontend 隱藏按鈕或 query filter 防止越權。RLS 會把 policy 條件套用到 database operation，即使 Client 發出 `.select('*')`，也只能看到 policy 允許的 rows。

在 multi-tenant system 中，「這筆 row 是誰建立的」與「這個 actor 是否屬於該 organization」是兩個不同問題。只檢查 ownership 可能漏掉 tenant boundary；只檢查 membership 也可能讓 organization 內所有成員取得過大的操作範圍。

## How It Works

### Policy Semantics

| Operation | `USING` | `WITH CHECK` |
| --- | --- | --- |
| `SELECT` | 檢查可看見的 existing row | — |
| `INSERT` | — | 檢查準備新增的 row |
| `UPDATE` | 檢查可以修改的 old row | 檢查 update 後的 new row |
| `DELETE` | 檢查可以刪除的 existing row | — |
| `ALL` | 檢查 existing row | 檢查 new row |

以只能讀取自己建立的 expense 為例：

```sql
create policy "read own expenses"
on expenses
for select
to authenticated
using (
  expenses.created_by = (select auth.uid())
);
```

`UPDATE` 同時跨越 old 與 new state：

```sql
create policy "update own expenses"
on expenses
for update
to authenticated
using (
  expenses.created_by = (select auth.uid())
)
with check (
  expenses.created_by = (select auth.uid())
);
```

`USING` 回答「actor 是否有權修改原本這筆 row」，`WITH CHECK` 回答「修改後的 row 是否仍然符合授權規則」。在 Supabase 中，`UPDATE` 要能找到 target row，通常也需要相應的 `SELECT` policy。

### Omitted `WITH CHECK`

不能一概而論地認為 `UPDATE` 只寫 `USING` 就一定可以任意改變 owner。對同時支援兩種 expression 的 `UPDATE`／`ALL` policy，如果沒有定義 `WITH CHECK`，PostgreSQL 會沿用 `USING` expression 檢查 new row。

實務上仍建議明確寫出 `WITH CHECK`：

- old row 與 new row 的授權條件可能不同。
- Policy intent 更容易 review 與測試。
- 多條 permissive／restrictive policies 組合後可能出現意外結果。
- Final-state check 只能證明 new row 合法，不一定能證明某個 sensitive column 從未被修改。

### Tenant Membership and Ownership

Tenant isolation 應把 membership 與 ownership 分開表達：

```sql
expenses.created_by = (select auth.uid())
and exists (
  select 1
  from organization_members m
  where m.organization_id = expenses.organization_id
    and m.user_id = (select auth.uid())
)
```

Policy 中的重要欄位應完整限定 table。若把 correlated subquery 寫成：

```sql
where m.organization_id = organization_id
```

未限定的右側名稱可能被解析成 subquery 內的欄位，使條件退化為近似 `m.organization_id = m.organization_id`。結果可能從「actor 是這筆 expense 所屬 organization 的 member」錯放成「actor 屬於任意 organization」。

Admin 權限也必須 tenant-scoped。問題不是 actor 是否在某處具有 admin role，而是 actor 是否為目前 row 所屬 organization 的 admin；`USING` 與 `WITH CHECK` 都要使用對應 old／new organization 的 scope。

### RLS Is Not Column Immutability

RLS policy 看到 old row 與 new row 是否分別符合 expression，但不能像 row trigger 一樣直接比較 `OLD.organization_id` 與 `NEW.organization_id`。如果 actor 同時是 organization A 與 B 的 member，只用 membership `WITH CHECK` 仍可能允許他把 row 從 A 移到 B。

若 `created_by`、`organization_id` 等欄位建立後不可變，需要其他防線：

- API schema 與 update allowlist：request 不接受 sensitive columns，也不把整個 body 直接傳給 `.update()`。
- Column privilege：只授予可修改欄位的 `UPDATE` 權限。
- 受控 RPC：只暴露業務允許的參數與 state transition。
- Trigger：以 `OLD`／`NEW` 比較所有 database write paths。
- [[Database Unique Constraint|Constraint]]：保護 uniqueness、range 與 relation 等 data invariant。

```sql
if new.organization_id is distinct from old.organization_id then
  raise exception 'organization_id cannot be changed';
end if;
```

Trigger 中的 `RAISE EXCEPTION` 會使 statement 失敗；若 exception 未被 savepoint 或 exception handler 隔離，所在 transaction 會進入 failed state，尚未 commit 的修改必須 rollback。`IS DISTINCT FROM` 能正確處理 `NULL`，比單純使用 `<>` 更適合比較 immutable fields。

### Responsibility Boundary

| Requirement | Primary enforcement layer |
| --- | --- |
| 哪些 rows 可以操作 | RLS |
| 哪些 columns 可以更新 | Column privilege、RPC、API allowlist |
| 比較 update 前後的資料 | Trigger |
| Range、uniqueness、relation | Check／unique／foreign-key constraint |
| Request shape | API schema |
| 付款、寄信、AI call | Backend、受控 RPC、async worker |
| UI 隱藏與提示 | Frontend |

這些層是 defense in depth。Frontend 改善 UX，API 提供清楚 contract，RLS 保護 final row authorization，trigger 與 constraints 保護資料不可變規則和 integrity。

## Example

Expense App 的 member update policy 可以拆成：

```text
USING
├─ actor 是 old row 的 creator
└─ actor 是 old row organization 的 member

WITH CHECK
├─ new row 的 created_by 仍是 actor
└─ actor 是 new row organization 的 member

Trigger / RPC / column privilege
├─ organization_id 未曾改變
└─ created_by 未曾改變
```

這份 policy design 是 [[Expense App Authentication]] 的 authorization follow-up 與 interview design exercise，不代表 production schema 已經部署。

## Tradeoffs

- RLS 將 authorization 靠近 data，但 policy composition、role、view、function execution identity 與 bypass privileges 都需要專門測試。
- Correlated subquery 能表達 tenant membership，卻增加 query 與 index 設計成本。
- 重複 `(select auth.uid())`、membership lookup 與 policy join 可能影響效能；優化前必須先保持語意正確。
- RLS 保護 database rows，不能自動授權 [[Google Cloud Storage|GCS]] object、AI provider 或其他 external resource。
- Service role 或 privileged backend 可能繞過一般 Client policy，因此 backend 仍須執行自己的 authorization。

## Related

- [[Supabase]]
- [[Expense App]]
- [[Expense App Authentication]]
- [[Expense Draft Review Workflow]]
- [[PostgreSQL Relational Data Modeling]]
- [[Database Unique Constraint]]
- [[API Contract Design]]
- [[Google Cloud Storage]]

## References

- [[2026-07-26 weekly updates]]
- [Supabase: Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [PostgreSQL: CREATE POLICY](https://www.postgresql.org/docs/current/sql-createpolicy.html)
