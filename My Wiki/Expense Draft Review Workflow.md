---
type: concept
status: seed
topics: [side-project, expense, ai, draft, workflow, optimistic-concurrency, idempotency]
created: "2026-07-28"
---

# Expense Draft Review Workflow

## Summary

Expense Draft Review Workflow 是 [[Expense App]] 的 Receipt AI subsystem design：AI extraction 不直接建立正式 expense，而是產生可驗證、可修改的 draft；使用者確認後，Server 才在 transaction 中建立 canonical expense。這是 2026-07-26 的設計推演，不代表目前 production implementation 已完成。

## Why It Matters

[[Expense Receipt AI Pipeline]] 的 model output 可能格式正確但內容錯誤，Client 也可能同時開啟多個畫面、重送 confirm，或在 Server commit 後失去 response。將 editable draft 與 formal expense 分開，可以讓 human review、validation、optimistic concurrency、idempotency 與 audit 各自有清楚的責任。

## How It Works

### State and Data Model

```text
extracting → pending_review → confirmed
     ↓              ↓
   failed        expired
```

Draft record 可保存：

```text
id
user_id
source_file_id
raw_extraction
draft_value
schema_version
model_version
prompt_version
status
version
validation_result
confirmed_expense_id
created_at
updated_at
confirmed_at
```

`raw_extraction` 保存不可修改的 AI evidence，`draft_value` 保存使用者目前正在編輯的 candidate。AI output 的 trust boundary 與 validation layers 整理在 [[AI Structured Output Validation]]。

### Edit with Optimistic Concurrency

使用者透過 PATCH 保存修正：

```http
PATCH /expense-drafts/{draft_id}

{
  "version": 1,
  "changes": {
    "total": 1000
  }
}
```

Server：

1. 驗證目前 actor 是 draft owner。
2. 驗證 status 是 `pending_review`。
3. 比對 request version 與 database version。
4. 將 changes 套用到完整 `draft_value` 後重新驗證。
5. Atomic update 並把 version 從 1 增加到 2。

舊畫面送出 stale version 時回 `409 Conflict`，避免覆蓋較新的修改。Version 防止 lost update；它不等於 [[Idempotency Key|idempotency key]]，後者處理同一 logical operation 因 timeout 或 retry 被重送。

### Confirm as a State Transition

Confirm 使用獨立 endpoint：

```http
POST /expense-drafts/{draft_id}/confirm
Idempotency-Key: <stable-client-key>

{
  "version": 2
}
```

Server 在同一個 database transaction 中：

```text
lock draft
→ verify owner
→ verify pending_review status
→ verify version
→ load latest draft_value
→ run complete validation
→ create expense
→ mark draft confirmed
→ save confirmed_expense_id
→ commit
```

這個 transaction 防止同一 draft 建立多筆 expense，也避免「expense 已建立，但 draft 未標記完成」。可再以 draft terminal state、`confirmed_expense_id` uniqueness 與 [[Idempotent Request Handling]] 吸收 concurrent confirm。

### PATCH and Confirm Remain Separate

- `PATCH`：保存可反覆修改的 candidate，支援 autosave 與稍後繼續。
- `confirm`：執行正式狀態轉換並建立 expense。

Frontend 可以把兩次 request 包裝成一個操作，但 network boundary 仍可能產生 partial failure。若產品要求「最後修改與確認」必須不可分割，可提供 `confirm-with-changes`，由 Server 在一個 transaction 內驗證 changes 並 confirm，而不是讓一般 PATCH 隱含建立正式資料。

### Timeout Recovery

```text
confirm committed
→ response lost
→ Client timeout
```

Client 使用相同 idempotency key retry。Server 若看到 draft 已經是 `confirmed`，就回傳既有 `confirmed_expense_id`；若仍是 `pending_review`，才依相同 operation contract 繼續。Frontend 也可以重新查詢 draft state，不能只用「是否收到 response」判斷成功與否。

Confirm API 與 error semantics 應遵循 [[API Contract Design]]；若 extraction 本身採 async job，則由 [[Reliable Background Job Processing]] 管理 provider call、retry、lease 與 recovery。

## Example

```text
Upload receipt
→ async extraction
→ pending_review draft v1
→ PATCH correction → draft v2
→ POST confirm(v2, idempotency key)
→ transaction creates expense_123
→ draft confirmed + confirmed_expense_id=expense_123
```

相同 confirm request 的 retry 回到 `expense_123`；不同 version 或同 key 不同 payload 應回 conflict，而不是再建立另一筆 expense。

## Tradeoffs

- Draft table 增加 lifecycle、cleanup、storage 與 query complexity，但提供清楚的 review 和 audit boundary。
- PATCH／confirm 分離適合 autosave，代價是多一個 partial-failure window。
- Optimistic concurrency 避免 silent overwrite，但 Client 必須處理 `409`、refresh 與 merge UX。
- Raw extraction 對 debug 有價值，也可能包含敏感 receipt data，需要 retention 與 access policy。
- 對低風險欄位可簡化 review；正式付款、報帳或跨 tenant state 不應只依賴 AI output。

## Related

- [[Expense App]]
- [[Expense Receipt AI Pipeline]]
- [[AI Structured Output Validation]]
- [[API Contract Design]]
- [[Idempotency Key]]
- [[Idempotent Request Handling]]
- [[Reliable Background Job Processing]]
- [[PostgreSQL Row-Level Security]]

## References

- [[2026-07-26 weekly updates]]
