---
type: concept
status: growing
topics: [api, idempotency, http, concurrency, distributed-systems]
created: "2026-07-24"
---

# Idempotency Key

## Summary

Idempotency Key 是 Client 為一次 logical operation 產生的唯一識別值。Retry 必須沿用同一個 key；Server 再用 key 找回第一次執行的結果，避免重複建立 resource、扣款或啟動工作。Key 本身不會自動保證 idempotency，真正的正確性仍依賴 [[Idempotent Request Handling]] 的 atomic claim、payload validation 與結果保存。

## Why It Matters

Client 收到 timeout 或連線中斷時，無法判斷 Server 是尚未收到 request，還是已經完成 commit 但 response 遺失。若每次 retry 都產生新 key，Server 會把它們視為不同操作；沿用原 key 才能安全查回同一個結果。這也是 [[API Contract Design]] 必須明確定義的 retry contract。

## How It Works

### Client Contract

- 在首次送出有 side effect 的 operation 前產生 key，例如 UUID v4 或具足夠隨機性的 ULID。
- 同一次 logical operation 的 timeout、network retry 與手動 retry 都沿用同一個 key。
- 使用者明確開始另一筆 operation 時才產生新 key。
- HTTP API 通常放在 `Idempotency-Key` header；key 應是不含業務機密的 opaque value。
- Client 應把 key 與待送出的操作一起保存，至少保留到取得確定結果或放棄該操作為止。

### Server Scope

Key 不應全系統共用一個 namespace，而應至少綁定：

```text
(tenant_or_user_id, operation_or_endpoint, idempotency_key)
```

這可避免不同 tenant 或不同 endpoint 碰巧使用相同 key 時互相讀到結果。Authentication identity 必須由 Server 從可信 credential 取得，不能只相信 request body 內的 user ID。

### Payload Binding

Server 對會影響結果的 request fields 建立 canonical payload hash，並與 key 一起保存：

- 同 key、同 hash：視為 retry，回傳原本的 resource 或結果。
- 同 key、不同 hash：拒絕執行，通常回 `409 Conflict` 與穩定 error code，例如 `IDEMPOTENCY_KEY_REUSED`。

Canonicalization 必須明確處理 JSON key order、預設值與不影響語意的欄位，否則語意相同的 payload 可能產生不同 hash。

### Record Lifecycle

Idempotency record 常見狀態如下：

```text
processing → completed
           ↘ failed
```

- `processing`：第一個 request 已取得執行權；後續 retry 應指向同一個 resource／job，或回可重試狀態。
- `completed`：保存 HTTP status、必要 response headers，以及完整 response 或可重建 response 的 resource ID。
- `failed`：必須由 API contract 定義是否重播原 failure，或允許在同一 key 下繼續既有工作；不能在沒有規則時重新執行 side effect。

在取得 key 的 ownership 前就發現的 authentication、validation 等錯誤，通常不需要建立可重播的 idempotency record。已經開始 side effect 後發生的錯誤，則不能只刪除 record 再重跑，否則可能造成重複結果。

### Atomic Claim

Database [[Database Unique Constraint|unique constraint]] 必須負責仲裁同時抵達的 requests：

```sql
CREATE TABLE idempotency_records (
  tenant_id       text        NOT NULL,
  operation       text        NOT NULL,
  idempotency_key text        NOT NULL,
  request_hash    text        NOT NULL,
  status          text        NOT NULL,
  resource_id     text,
  response_code   integer,
  response_body   jsonb,
  created_at      timestamptz NOT NULL DEFAULT now(),
  expires_at      timestamptz NOT NULL,
  PRIMARY KEY (tenant_id, operation, idempotency_key)
);
```

可以把這個 constraint 想成 database 發號碼牌：相同 scope 只能有一筆 record，因此多台 Server 同時收到 request 時，也只有一個 request 能取得執行權。

參與流程的欄位各有不同責任：

| 欄位 | 來源 | 責任 |
| --- | --- | --- |
| `tenant_id` | Server 從 authenticated identity 取得 | 隔離不同 user 或 tenant |
| `operation` | Server 根據業務操作決定，例如 `create_parsing_job:v1` | 隔離不同 endpoint 或 operation |
| `idempotency_key` | Client 為 logical operation 產生 | 識別第一次 request 與後續 retry |
| `request_hash` | Server 對 canonical payload 計算 | 檢查同一個 key 是否被套用到不同內容 |
| `resource_id` | Winner 建立 resource 後保存 | 讓 retry 找回原本的 job 或 resource |

真正負責 unique arbitration 的是 `(tenant_id, operation, idempotency_key)`。`request_hash` 不放進 unique key，否則同一個 idempotency key 搭配不同 payload 會被當成兩筆合法操作，而不是被偵測為 misuse。`resource_id` 也不參與仲裁，它只是連回 winner 建立的結果。

#### Winner and Loser Flow

Server 不應先 `SELECT` 判斷 record 是否存在，再決定要不要 `INSERT`。兩個 request 可能同時查到「不存在」，接著都開始執行 side effect。正確做法是直接嘗試 atomic insert：

```sql
INSERT INTO idempotency_records (
  tenant_id,
  operation,
  idempotency_key,
  request_hash,
  status,
  expires_at
)
VALUES (
  :tenant_id,
  :operation,
  :idempotency_key,
  :request_hash,
  'processing',
  :expires_at
)
ON CONFLICT DO NOTHING
RETURNING *;
```

流程如下：

1. `INSERT ... RETURNING` 有結果：這個 request 是 winner，取得執行權。
2. 沒有結果：代表相同 scoped key 已存在，這個 request 是 loser，必須讀取既有 record，不能再次執行 side effect。
3. Loser 比對 `request_hash`：
   - hash 相同且 status 是 `completed`：重播保存的 response，或透過 `resource_id` 找回原 resource。
   - hash 相同且 status 是 `processing`：回到同一個 job／resource，或回 contract 定義的可重試狀態。
   - hash 不同：回 `409 IDEMPOTENCY_KEY_REUSED`，拒絕把同一個 key 用於另一個操作內容。

若兩個 `INSERT` 同時抵達，database [[Database Unique Constraint|unique index]] 會讓其中一個等待仲裁。Winner commit 後，另一個 request 會遇到 conflict；若 winner rollback，另一個 request 才可能成功 insert 並成為新的 winner。

#### Commit the Result

只操作同一個 database 時，應盡量讓 idempotency record、business resource 與結果連結在同一個 transaction commit：

```text
BEGIN
  atomic insert idempotency record
  create business resource or durable job
  update record with resource_id and replayable response
COMMIT
```

對 async API，這裡完成的是「建立 job」這個 operation，不是長時間背景工作本身。Transaction 可以原子建立 `job_456` 並把它寫入 idempotency record，之後 retry 一律回到 `job_456`；真正的 job execution 再交給 [[Reliable Background Job Processing]]。

若 side effect 位於 payment、AI provider 等外部系統，local database transaction 無法涵蓋該 boundary。此時仍須把穩定 key 傳給 provider，或搭配 [[Transactional Outbox]] 與 reconciliation，不能因為 local atomic claim 成功就假設外部操作只會執行一次。

### Expiration

Record 需要 retention period 與 cleanup policy。TTL 至少要涵蓋 Client、queue 與 operator 可能重試的最長時間，並在 API contract 說明：

- key 可被安全重播多久；
- expiry 後重送是否會被當成全新 operation；
- 長時間 async job 是否要保留到 terminal state 之後一段時間。

TTL 過短可能讓延遲 retry 產生第二次 side effect；永久保存則增加 storage 與 privacy 成本。

## Example

```http
POST /v1/parsing-jobs HTTP/1.1
Authorization: Bearer <token>
Idempotency-Key: 01K0EXAMPLE8M8Y7K9Q4J5N6P
Content-Type: application/json

{"document_id":"doc_123","parser":"receipt-v2"}
```

第一次成功接受時：

```http
HTTP/1.1 202 Accepted
Location: /v1/parsing-jobs/job_456

{"job_id":"job_456","status":"queued"}
```

相同 key 與 payload 的 retry 應回到 `job_456`，而不是建立另一個 job。若 key 相同但 `document_id` 或 `parser` 不同，Server 應回 idempotency conflict。

## Tradeoffs

- 保存完整 response 能精準 replay，但 storage 成本較高；只保存 resource ID 較省空間，但 response 可能反映 resource 的最新狀態而非原始 snapshot。
- Key 能保護 Server 管理的 boundary，不能自動涵蓋 payment provider、AI provider 或 message queue；每個外部 side-effect boundary 都要傳遞穩定 key，或搭配 [[Transactional Outbox]]、deduplication 與 reconciliation。
- 可預測或未限制長度的 key 可能造成資料探測與 storage abuse；應依 authenticated scope 查詢、限制格式與長度，並避免在 log 暴露敏感 payload。
- Exactly-once execution 通常無法跨所有系統保證；Idempotency Key 的實際目標是讓 at-least-once delivery 產生一次可觀察的業務結果。

## Related

- [[Idempotent Request Handling]]
- [[Database Unique Constraint]]
- [[API Contract Design]]
- [[Expense Draft Review Workflow]]
- [[Reliable Background Job Processing]]
- [[Transactional Outbox]]

## References

- [[Idempotent Request Handling]]
- [[Async API Contract Interview Practice]]
- [[2026-07-26 weekly updates]]
