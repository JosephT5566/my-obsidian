# PostgreSQL 併發控制與資料一致性筆記

## 1. 核心觀念：讓資料庫保證 invariant

Invariant 是系統中「永遠不能被破壞的規則」，例如：

- 庫存不能小於 0
- 不可發出第 10,001 張折價券
- 同一位使用者不能重複領同一活動的券
- 餘額不能被扣成負數
- 同一筆工作不能被兩個 worker 同時處理

這些規則不要只靠 application code 的 `if` 判斷，應盡量交給 PostgreSQL 的 constraint、atomic SQL、lock 和 transaction 保證。

---

## 2. `UNIQUE constraint`

`UNIQUE` 保證單一欄位或欄位組合不能重複。

### 單一欄位唯一

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email TEXT UNIQUE
);
```

### 組合唯一

```sql
CREATE TABLE coupon_claims (
    campaign_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,

    UNIQUE (campaign_id, user_id)
);
```

代表：

- `campaign_id` 可以重複
- `user_id` 可以重複
- 但同一個 `(campaign_id, user_id)` 組合只能存在一次

適合：

- Email、username
- OAuth user ID
- Idempotency key
- 同一人不能重複領券
- 同一學生不能重複選同一門課

`PRIMARY KEY` 也具有唯一性，但一張 table 通常只有一個 primary key；`UNIQUE` 可以有很多個。

---

## 3. 不安全的「先查再改」

以下流程在高併發下可能有 race condition：

```sql
SELECT issued_count
FROM coupon_campaigns
WHERE id = 123;
```

Application：

```tsx
if (issuedCount < totalLimit) {
  await updateCampaign();
}
```

兩個 request 可能同時讀到：

```
issued_count = 9999
```

然後都認為還剩一張券。

即使後面的 `UPDATE` 會等待 row lock，前面的商業判斷已經基於舊資料完成，因此仍可能更新成 `10001`。

---

## 4. Atomic Conditional Update

較安全的方式是把「檢查條件」和「修改」放在同一條 SQL：

```sql
UPDATE coupon_campaigns
SET issued_count = issued_count + 1
WHERE id = 123
  AND issued_count < total_limit
RETURNING issued_count;
```

PostgreSQL 會原子地完成：

```
取得 row lock
→ 檢查最新資料
→ 判斷 WHERE
→ 符合才更新
```

如果兩個 request 同時執行：

```
Request A：更新為 10000
Request B：等待 A
Request B 取得最新資料後重新檢查 WHERE
10000 < 10000 為 false
→ UPDATE 0 rows
```

Application 必須檢查：

```
affected rows / returned rows
```

而不是只檢查 SQL 是否沒有拋出錯誤。

### 常見情境

扣庫存：

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 500
  AND stock > 0
RETURNING stock;
```

扣餘額：

```sql
UPDATE accounts
SET balance = balance - 800
WHERE id = 123
  AND balance >= 800
RETURNING balance;
```

搶任務：

```sql
UPDATE jobs
SET status = 'processing'
WHERE id = 100
  AND status = 'pending'
RETURNING *;
```

---

## 5. `SELECT ... FOR UPDATE`

`FOR UPDATE` 用於「需要先讀資料，再做較複雜判斷」的情境。

```sql
BEGIN;

SELECT issued_count, total_limit
FROM coupon_campaigns
WHERE id = 123
FOR UPDATE;

-- application 做判斷

UPDATE coupon_campaigns
SET issued_count = issued_count + 1
WHERE id = 123;

COMMIT;
```

### 鎖住後會發生什麼

一般 `SELECT`：

```sql
SELECT *
FROM coupon_campaigns
WHERE id = 123;
```

通常仍可立即讀取已提交的舊版本。

但以下操作會等待 lock：

```sql
SELECT ... FOR UPDATE;
UPDATE ...;
DELETE ...;
```

第二個 `SELECT ... FOR UPDATE` 不是先讀到舊資料再等待，而是：

```
找到 row
→ 嘗試取得 lock
→ 等待
→ 前一個 transaction commit
→ 讀取最新 committed row
→ 回傳結果
```

因此第二個 transaction 會看到更新後的資料。

### 使用時機

適合：

- 需要檢查多個欄位
- 複雜狀態轉換
- 付款或退款流程
- 需要先讀取，再執行多步驟操作

如果規則能直接寫進 `UPDATE ... WHERE`，通常 atomic update 更簡潔。

---

## 6. Transaction 的作用

Transaction 保證其中的操作：

```
全部成功
或
全部 rollback
```

例如：

```sql
BEGIN;

UPDATE coupon_campaigns
SET issued_count = issued_count + 1
WHERE id = 123
  AND issued_count < total_limit
RETURNING id;

INSERT INTO coupon_claims (campaign_id, user_id)
VALUES (123, 456);

COMMIT;
```

如果 `INSERT` 因為 unique constraint 失敗，前面的計數更新也會 rollback，因此不會浪費券的名額。

---

## 7. `ON CONFLICT`

`ON CONFLICT` 用來處理 unique conflict。

### 重複時忽略

```sql
INSERT INTO coupon_claims (campaign_id, user_id)
VALUES (123, 456)
ON CONFLICT (campaign_id, user_id)
DO NOTHING
RETURNING id;
```

若回傳 0 rows，代表已經存在。

### Upsert

```sql
INSERT INTO user_settings (user_id, theme)
VALUES (456, 'dark')
ON CONFLICT (user_id)
DO UPDATE SET theme = EXCLUDED.theme;
```

適合：

- 使用者設定
- Webhook 去重
- 快取
- 每日統計
- Idempotent insert

---

## 8. Optimistic Locking

用 `version` 避免後寫入者覆蓋別人的新資料。

```sql
UPDATE orders
SET note = 'new note',
    version = version + 1
WHERE id = 100
  AND version = 5;
```

若另一人已先更新，版本可能已變成 6：

```
UPDATE 0 rows
```

Application 可提示使用者重新整理。

適合：

- 後台表單
- 文件編輯
- 訂單管理
- CRM
- 衝突不常發生，但不可默默覆蓋的資料

---

## 9. `CHECK constraint`

防止資料進入非法狀態。

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    stock INTEGER NOT NULL CHECK (stock >= 0),
    price NUMERIC NOT NULL CHECK (price >= 0)
);
```

其他例子：

```sql
CHECK (discount_percent BETWEEN 0 AND 100)
```

```sql
CHECK (end_at > start_at)
```

`CHECK` 適合驗證同一列中的規則，不適合跨很多列計算總量。

---

## 10. Foreign Key

Foreign key 保證關聯資料存在。

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id)
);
```

刪除策略：

```sql
ON DELETE RESTRICT
ON DELETE CASCADE
ON DELETE SET NULL
```

適合：

- Order → User
- Expense → Category
- Comment → Article
- Payment → Order

---

## 11. Partial Unique Index

只對符合特定條件的資料套用唯一限制。

例如：一個使用者只能有一個 active subscription。

```sql
CREATE UNIQUE INDEX unique_active_subscription
ON subscriptions (user_id)
WHERE status = 'active';
```

也可用於：

- 一個 primary address
- 一個 default payment method
- 一個進行中的驗證
- 一個未完成的 checkout

---

## 12. `FOR UPDATE SKIP LOCKED`

適合多個 worker 同時領取不同工作。

```sql
BEGIN;

SELECT id, payload
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs
SET status = 'processing'
WHERE id = ?;

COMMIT;
```

效果：

```
Worker A 鎖 Job 1
Worker B 跳過 Job 1，取得 Job 2
Worker C 取得 Job 3
```

常見於：

- Email queue
- AI parsing job
- 圖片處理
- Webhook delivery
- 報表產生

通常還需設計：

```
locked_at
locked_by
retry_count
```

以處理 worker crash。

---

## 13. Advisory Lock

當沒有自然存在的 row 可以鎖時，可使用自訂 lock key。

```sql
BEGIN;

SELECT pg_advisory_xact_lock(456);

-- 同一 user 的月結算

COMMIT;
```

適合：

- 月結算
- 同一份報表避免重複產生
- 同一 tenant 的 migration
- 同一使用者的批次同步

缺點是所有程式都必須遵守相同 lock 規則。

---

## 14. Exclusion Constraint

用來避免時間或範圍重疊。

```sql
EXCLUDE USING gist (
    room_id WITH =,
    tstzrange(start_at, end_at, '[)') WITH &&
);
```

適合：

- 會議室預約
- 飯店房間
- 醫師門診
- 車輛租借
- 排班

它比 application 先查是否重疊再 insert 更能避免 concurrent race condition。

---

## 15. Transactional Outbox

當需要同時：

```
寫入資料庫
+
發送 message/event
```

不要直接在兩個系統間連續操作，因為中途可能 crash。

應在同一 transaction 寫入：

```sql
BEGIN;

INSERT INTO orders (...);

INSERT INTO outbox_events (
    event_type,
    payload
)
VALUES (
    'order.created',
    '{"order_id": 500}'
);

COMMIT;
```

另一個 worker 再從 outbox 發送 Kafka、Email 或 webhook。

這能保證：

```
業務資料與事件紀錄一起成功或一起失敗
```

---

## 16. Idempotency Key

避免同一 request 因重試而重複執行。

```sql
CREATE TABLE idempotency_requests (
    user_id BIGINT NOT NULL,
    idempotency_key TEXT NOT NULL,
    request_hash TEXT NOT NULL,
    status TEXT NOT NULL,
    response JSONB,

    PRIMARY KEY (user_id, idempotency_key)
);
```

先嘗試：

```sql
INSERT INTO idempotency_requests (...)
VALUES (...)
ON CONFLICT DO NOTHING
RETURNING *;
```

若沒有回傳 row，代表該 request 已處理或正在處理。

適合：

- 付款
- 轉帳
- 建立訂單
- 報銷提交
- AI parsing
- Webhook handling

---

## 17. `SERIALIZABLE`

當規則跨越多筆資料，難以用單一 row lock 或 constraint 表達時，可考慮：

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

PostgreSQL 若偵測 concurrent transaction 可能導致不一致，會讓其中一個失敗：

```
could not serialize access due to read/write dependencies
```

Application 必須 retry 整個 transaction。

它適合複雜跨多列的商業規則，但不是預設首選。

---

# 實務選擇順序

## 防止重複

```
UNIQUE
partial unique index
ON CONFLICT
```

## 防止超賣、負數或超額

```
conditional UPDATE
CHECK constraint
RETURNING
```

## 防止覆蓋別人的更新

```
version column
optimistic locking
```

## 需要先讀取，再做複雜判斷

```
SELECT ... FOR UPDATE
```

## 多個 worker 領取不同工作

```
FOR UPDATE SKIP LOCKED
```

## 沒有 row 可以鎖

```
advisory lock
```

## 資料庫與訊息系統需要一致

```
transactional outbox
```

## 複雜跨多列規則

```
SERIALIZABLE
```

# 最重要的 Mindset

先問自己三個問題：

1. **什麼規則絕對不能被破壞？**
2. **這個判斷是否可能受到 concurrent request 影響？**
3. **能否把檢查與修改合併成一個 database operation？**

優先採用：

```
Database constraint
→ Atomic SQL
→ Row lock
→ Serializable transaction
```

不要只相信：

```
先 SELECT
→ application 判斷
→ 再 UPDATE
```

因為兩個 request 可能同時根據相同的舊資料作出決定。真正可靠的最後仲裁者，應該是 PostgreSQL。

# 限量折價券系統：面試筆記

## 題目情境

電商活動發放 10,000 張限量折價券：

- 每位使用者只能領一次。
- 活動開始時有大量並行請求。
- 使用者可能重複點擊。
- HTTP request 可能逾時並重送。
- 系統必須避免超發，並回傳可信的最終結果。

## 核心資料模型

```
coupon_campaigns
- id
- capacity
- claimed_count
- starts_at
- ends_at

coupon_claims
- id
- campaign_id
- user_id
- created_at

idempotency_records
- user_id
- operation
- idempotency_key
- request_hash
- status
- response
```

重要 constraints：

```
UNIQUE (campaign_id, user_id)
```

確保每位使用者在同一活動只能領一次。

```
UNIQUE (user_id, operation, idempotency_key)
```

確保相同操作可以安全重送。Idempotency key 應綁定 authenticated user、操作類型與 request hash，不能只相信 client 傳來的 UUID。

## 三層保證

### 1. Idempotency key：處理相同操作重送

相同 key、相同 request：

- 回傳第一次操作的既有結果。
- 不再次扣除庫存。

相同 key、不同 request：

- 回傳 conflict，例如 `IDEMPOTENCY_KEY_REUSED`。
- 避免錯誤地把舊結果套用到不同操作。

但不同 idempotency keys 仍可能代表同一位使用者重複領券，因此不能只依靠 idempotency key。

### 2. Unique constraint：處理業務重複

`UNIQUE (campaign_id, user_id)` 是「每人限領一次」的最終資料庫防線。

即使：

- 使用者同時從兩台裝置領券；
- client 產生兩個不同 idempotency keys；
- application layer 的檢查發生 race condition；

資料庫仍只允許一筆 claim。

第二個請求不應直接把 raw unique-constraint error 回傳前端，而應查詢既有 claim，回傳穩定的業務結果，例如：

```
{
  "status": "already_claimed",
  "claim_id": "..."
}
```

### 3. Transaction：維持庫存與 claim 一致

下列操作必須在同一個 transaction：

1. 仲裁庫存。
2. 建立 `coupon_claims`。
3. 更新 `claimed_count`。
4. 保存操作結果。

任一步失敗都 rollback，避免：

- 已扣庫存但沒有 claim。
- 有 claim 但庫存數量沒有增加。
- idempotency record 與實際結果不一致。

## 使用 `SELECT ... FOR UPDATE`

```
BEGIN;

SELECT capacity, claimed_count
FROM coupon_campaigns
WHERE id = $1
FOR UPDATE;
```

重要修正：`FOR UPDATE` 通常鎖住選中的 campaign row，不是整張 table。

第一個 transaction 取得該活動的 row lock；其他競爭同一活動的 transaction 必須等待。取得鎖後必須重新檢查最新庫存，不能沿用 transaction 外讀到的舊值。

簡化流程：

```
鎖定 campaign row
        ↓
檢查使用者是否已有 claim
        ↓
檢查 claimed_count < capacity
        ↓
新增 claim 並更新 claimed_count
        ↓
commit
```

這個設計容易解釋且正確，但熱門活動的單一 campaign row 會成為 contention hotspot，所有領券請求都會在該 row 排隊。

## 使用條件式原子更新

也可以把庫存檢查和扣減合併：

```
UPDATE coupon_campaigns
SET claimed_count = claimed_count + 1
WHERE id = $1
  AND claimed_count < capacity
RETURNING id;
```

- 有回傳 row：成功保留一張券。
- 沒有回傳 row：活動不存在或券已發完，應再依需求區分。
- 並行 update 由 PostgreSQL 仲裁，等待者會依最新資料重新判斷條件。

接著在同一 transaction 建立 claim。如果 claim insert 因為使用者已領過而違反 unique constraint，整個 transaction 必須 rollback，不能保留前面的庫存加一。

應捕捉該業務衝突，在 transaction 結束後讀取既有 claim，轉換成可理解的 `already_claimed` 結果。

## 同一使用者使用不同 keys

這是 idempotency 與業務唯一性之間的重要差別：

```
相同 key
→ idempotency record 處理

不同 keys，但同一 user + campaign
→ coupon_claims unique constraint 處理
```

若採用 campaign row lock，可在取得鎖後先查詢既有 claim：

- 已存在：直接回傳 `already_claimed`，不扣庫存。
- 不存在：再檢查庫存並建立 claim。

若採用條件式 update，claim insert 的 unique conflict 必須使 transaction rollback，再讀取並回傳既有 claim。

## 逾時與可信結果

HTTP response 遺失不代表 transaction 失敗。

Client 不應產生新 key 盲目重試，而應：

- 使用相同 idempotency key 重送；或
- 查詢目前的領券狀態。

Backend 應回傳 authoritative database result：

- `claimed`
- `already_claimed`
- `sold_out`
- `in_progress`
- `idempotency_key_reused`

這個同步、短 transaction 的流程通常不需要先建立 job queue。只有涉及較慢的外部服務或必須非同步處理時，才需要 durable job state machine。

## 面試版完整回答

> 我會用三層機制保證正確性。第一層是 idempotency key，用來處理相同 API 操作的安全重送；key 會綁定 authenticated user、operation 和 request hash，相同內容會回傳第一次的結果，不同內容重用同一 key 則回傳 conflict。
> 
> 第二層是 `UNIQUE (campaign_id, user_id)`，確保即使 client 使用不同 idempotency keys，同一使用者仍只能領一次。
> 
> 第三層是在短 transaction 內仲裁庫存、建立 claim、更新數量並保存結果。我可以鎖定指定的 campaign row，取得鎖後重新檢查庫存；也可以使用 `UPDATE ... WHERE claimed_count < capacity RETURNING` 做條件式原子扣減。若後續 claim insert 失敗，整個 transaction rollback。
> 
> 如果 transaction 已提交但 response 遺失，client 會使用相同 key 重送或查詢領券狀態，backend 則回傳資料庫中的既有結果，不會再次扣除庫存。

## 本次表現整理

已展現的優點：

- 能主動想到 idempotency key、unique constraint 與 transaction。
- 理解並行請求需要由資料庫仲裁。
- 知道 transaction 可以維持 claim 與庫存更新的 atomicity。
- 能使用 `SELECT ... FOR UPDATE` 描述競爭請求的執行順序。

需要加強的地方：

- 精確描述 lock granularity：`FOR UPDATE` 鎖的是選中的 row，不是整張 table。
- 先列出完整 invariants，再分配給 idempotency、constraint 和 transaction。
- 不要過早將同步操作建模成 async job。
- 補充 transaction 成功但 HTTP response 遺失時的恢復流程。
- 區分「相同 key 重送」與「不同 keys 的業務重複」。

接下來三個練習重點：

1. 用「業務 invariant → 資料庫保證 → API 行為」的順序回答。
2. 熟悉 row lock、conditional update、unique conflict 和 rollback 的並行語意。
3. 練習 ambiguous timeout、穩定 error codes，以及 idempotent response 的設計。

# Backend Service 開發流程、Production Readiness 與指標

> Type：Project Learning
> 
> Date：2026-07
> 
> Topic：Backend Engineering、Production Readiness、Internal Service、LLM Security

## TL;DR

Backend service 沒有一個通用的「完整度百分比」。比較實用的判斷方式是同時使用：

1. **MVP Scope**：這一版要完成哪些核心功能。
2. **Release Gates**：哪些條件沒達成就不能上線。
3. **Readiness Scorecard**：服務目前成熟到什麼程度。
4. **SLO 與營運指標**：上線後是否持續健康。
    1. SLO: **Service Level Objective**。它回答：在一段時間內，服務應該可靠到什麼程度？

核心原則：

> MVP 決定做多少功能；Production Readiness 決定能不能安全上線。

即使是公司內部服務，也不能假設所有輸入與 caller 都可信。可以降低部分公開網路防護與高可用需求，但 authentication、authorization、input validation、failure handling 和 observability 仍然是基本要求。

---

## 1. 先定義 Use Case、資料契約與信任邊界

開發前先回答：

1. API 接受與回傳什麼資料？
2. 哪些資料來自不可信來源？
3. 哪些欄位是 domain model，哪些只是 provider raw data？
4. Caller 需要處理哪些成功與失敗狀態？
5. Model、schema、權限與其他控制參數由誰決定？

這一步會影響後續 API、database、frontend、validation 和測試設計。

### PR 例子

- [PR #18 — Strict receipt extraction schema](https://github.com/JosephT5566/google-ai-gcf/pull/18)
    - 使用 Pydantic v2 建立 canonical receipt schema。
    - 限制欄位、型別、金額、日期、幣別、字串長度與品項數量。
    - 區分 missing、explicit null、unrecognized、recognized。
    - 從同一份 canonical schema 衍生 application schema 與 Gemini-compatible schema。
- [PR #20 — Versioned receipt result contract](https://github.com/JosephT5566/google-ai-gcf/pull/20)
    - 建立 `receipt-result.v1`。
    - 將 raw provider output、normalized data、validation result 和版本資訊分開。

### 可沿用的 BE 原則

> 先定義 domain contract，再串接 provider 或建立 persistence，避免各層自行發明資料格式。

---

## 2. 建立 Authentication、Authorization 與 Resource Ownership

Authentication 回答「呼叫者是誰」，Authorization 回答「呼叫者可以操作什麼」。

Backend 不能因為 request 帶有合法 JWT，就直接相信 request 中指定的 object ID、user ID 或 storage path。所有 resource identifier 都需要額外的 ownership 或 permission check。

### PR 例子

- [PR #21 — GCS ownership and image-input hardening](https://github.com/JosephT5566/google-ai-gcf/pull/21)
    - 使用 verified JWT `sub` 作為 GCS namespace authorization boundary。
    - 只允許存取 `uploads/{verified_sub}/...`。
    - 拒絕跨使用者 path、相似 prefix、dot segment、反斜線與控制字元。
    - Authorization 失敗時，不執行 GCS lookup、download 或 Gemini call。

### 可沿用的 BE 原則

> Authentication 不等於 Authorization；難猜的 UUID、timestamp 或 object path 也不等於權限控制。

---

## 3. 驗證所有外部輸入，再執行昂貴操作

外部輸入不只包含 JSON request，也包含：

- Path、query parameter 與 uploaded filename。
- MIME type、file size 與檔案內容。
- 第三方 API、webhook 或 LLM response。
- Storage metadata。
- 由其他內部服務傳入的 user context。

建議 runtime 順序：

```
Parse request
→ Validate shape and limits
→ Authenticate caller
→ Authorize resources
→ Inspect metadata
→ Load content
→ Execute expensive provider call
```

### PR 例子

- [PR #21 — Image limits and metadata validation](https://github.com/JosephT5566/google-ai-gcf/pull/21)
    - 最多 4 張圖片。
    - 單檔最多 10 MiB，整體 request 最多 20 MiB。
    - 只允許 JPEG、PNG、WebP。
    - 下載前檢查 GCS MIME、size、generation；下載後檢查 byte length 與 magic bytes。
    - 使用 generation precondition 降低 metadata check 與 download 之間的 TOCTOU race condition。

### 可沿用的 BE 原則

> 先完成 validation、authorization 與 resource-limit checks，再產生網路流量、計費或資料庫副作用。

---

## 4. 隔離外部 Provider，建立 Application Trust Boundary

第三方 provider 應包在清楚的 adapter 或 boundary 後面。Provider-specific response 不應直接成為 application domain model。

建議資料流：

```
Provider response
→ SDK parsing
→ Application-side validation
→ Internal domain model
→ Service logic / persistence
```

### PR 例子

- [PR #19 — Gemini SDK structured response](https://github.com/JosephT5566/google-ai-gcf/pull/19)
    - 使用 `response.parsed`，不再解析 `response.text` 或 Markdown JSON fences。
    - SDK structured parsing 後，仍使用 canonical Pydantic schema 再驗證。
    - Schema mismatch 統一轉換成 `InvalidModelOutputError`。
    - Invalid output 會 short-circuit，不進入 normalization 或未來的 persistence。

### 可沿用的 BE 原則

> Provider schema 可以降低錯誤率，但不能取代 application-side validation。

這個模式同樣適用於 payment gateway、OAuth provider、email service、webhook 和其他外部 API。

---

## 5. Normalize 成可信任的 Domain Model

通過 schema validation，只代表資料形狀合法；資料通常仍需要 normalization 與 business classification。

常見工作包括：

- 統一日期、currency 與 monetary representation。
- 將 provider enum 映射成 domain enum。
- 保留 missing、null 與 unknown 的不同語意。
- 計算 derived status 或 warning。
- 執行 business semantic validation。

### PR 例子

- [PR #20 — Raw and normalized data separation](https://github.com/JosephT5566/google-ai-gcf/pull/20)
    - `raw_extraction` 保留 provider extraction。
    - `normalized_receipt` 提供 caller 應使用的 canonical values。
    - Decimal 序列化為字串，避免跨語言浮點誤差。
    - 依完整度與 confidence 分成 `auto_acceptable`、`needs_review`、`invalid`、`retryable_failure`。

### 可沿用的 BE 原則

> Raw data 可以保留作為 tracing 證據，但 business logic 只能依賴 validated、normalized data。

---

## 6. 定義版本化 API Contract 與 Error Taxonomy

成功與失敗都應具有穩定、可預測的 response contract。Caller 應依賴 stable status、error code 與 version，而不是 provider wording 或自由文字 error message。

設計時需要區分：

1. Client 輸入錯誤：caller 應修改 request，通常不應 retry。
2. Authorization 錯誤：caller 沒有權限。
3. Storage／dependency failure：可能適合稍後 retry。
4. Provider invalid output：可依 policy retry 或送人工處理。
5. Internal failure：回傳 sanitized response，詳細資訊只留在受控 logs。

### PR 例子

- [PR #19](https://github.com/JosephT5566/google-ai-gcf/pull/19)
    - 將 schema mismatch 映射成 sanitized `502 / INVALID_MODEL_OUTPUT`。
- [PR #20](https://github.com/JosephT5566/google-ai-gcf/pull/20)
    - 建立統一的 `receipt-result.v1` envelope 與四種 public statuses。
- [PR #21](https://github.com/JosephT5566/google-ai-gcf/pull/21)
    - 分離 client input、GCS Storage 與 Gemini provider failure domain。
- [PR #22](https://github.com/JosephT5566/google-ai-gcf/pull/22)
    - Schema-invalid output 的 `raw_extraction` 改為 `null`，避免任意 provider payload 被反射給 caller。

### 可沿用的 BE 原則

> Error code 應描述責任歸屬與下一步行動：修改輸入、停止重試、重新上傳或稍後重試。

---

## 7. Security Hardening：分離 Control Plane 與 Data Plane

Security 不是最後才補的功能，而是橫跨每一層的 review。

在 LLM integration 中：

- **Control plane**：system instruction、model、schema、tools、temperature 與 generation settings。
- **Data plane**：圖片、OCR、使用者文字與其他待處理內容。

不可信輸入可以影響 schema 允許的資料值，但不能修改 control plane。

### PR 例子

- [PR #22 — Prompt-injection trust-boundary hardening](https://github.com/JosephT5566/google-ai-gcf/pull/22)
    - 使用 receipt-specific system instruction 定義 instruction/data boundary。
    - Model、prompt、schema 與 generation config 完全由 server 控制。
    - Receipt extraction 明確不提供 tools。
    - Client 無法透過 request 覆蓋 model、prompt、schema、temperature 或 tool config。
    - 加入 URL、tool request、table name、user ID 與 configuration injection tests。

### 可沿用的 BE 原則

> 通過 schema 的文字仍然只是合法資料，不代表它是可信指令。

類似概念也適用於 SQL injection、configuration injection、template injection 與 command injection。

---

## 8. Testing：不只驗證結果，也驗證 Pipeline 沒有繼續

Backend tests 應同時涵蓋：

1. Happy path 與最小合法輸入。
2. Boundary values、missing fields、extra fields 與錯誤型別。
3. Authorization、cross-user access 與 resource limits。
4. Provider／Storage failure 和 sanitized error response。
5. Invalid input 後沒有執行 download、normalization、persistence 或 provider call。

### PR 例子

- [PR #18](https://github.com/JosephT5566/google-ai-gcf/pull/18)：36 tests，涵蓋 schema、金額、日期、幣別與 serialization。
- [PR #19](https://github.com/JosephT5566/google-ai-gcf/pull/19)：40 tests，加入 structured response 與 pipeline short-circuit。
- [PR #20](https://github.com/JosephT5566/google-ai-gcf/pull/20)：50 tests，加入 contract、status 與 normalization protection。
- [PR #21](https://github.com/JosephT5566/google-ai-gcf/pull/21)：88 tests，加入 ownership、MIME、size、generation 與 Storage failure。
- [PR #22](https://github.com/JosephT5566/google-ai-gcf/pull/22)：97 tests，加入 prompt injection 與 failure-path reflection protection。

### 可沿用的 BE 原則

> Negative test 不應只檢查 HTTP status，也要確認錯誤資料沒有流入下一層或產生副作用。

---

## 9. MVP、Release Gate、Readiness Scorecard 與 SLO

這四個概念負責不同問題：

|機制|回答的問題|
|---|---|
|MVP Scope|這一版需要哪些核心功能？|
|Release Gate|哪些條件沒完成就不能上線？|
|Readiness Scorecard|Service 現在成熟到什麼程度？|
|SLO / Metrics|上線後是否持續健康？|

### MVP 不等於 Minimum Safe Product

MVP 可以暫時省略進階報表、多 region、高度自動化和完整管理後台，但不應省略基本 authorization、input validation、secret protection 和 safe failure handling。

### Mandatory Release Gates

以下任一問題通常都應阻擋正式上線：

1. 存在跨使用者資料存取或缺少 ownership check。
2. Secrets、provider payload 或敏感資料可能出現在 response／logs。
3. 任意輸入可以控制 provider、database 或其他副作用。
4. 沒有基本 payload、file size 或 concurrency limits。
5. 無法發現 critical failure，或無法安全 rollback。

### PR 例子

- [PR #21](https://github.com/JosephT5566/google-ai-gcf/pull/21) 解決 cross-user GCS access，屬於 release gate，不只是成熟度加分。
- [PR #22](https://github.com/JosephT5566/google-ai-gcf/pull/22) 解決 control-plane override 與 invalid-output reflection，也屬於 release gate。

---

## 10. Production Readiness Scorecard

每個面向可使用 0–3 分：

- `0`：尚未處理。
- `1`：只有 happy path。
- `2`：主要情境已處理。
- `3`：有測試、文件與 production verification。

|面向|檢查內容|對應 PR|
|---|---|---|
|Correctness|Use case、schema、business rules|[#18](https://github.com/JosephT5566/google-ai-gcf/pull/18)、[#20](https://github.com/JosephT5566/google-ai-gcf/pull/20)|
|Security|AuthN、AuthZ、input limits、injection protection|[#21](https://github.com/JosephT5566/google-ai-gcf/pull/21)、[#22](https://github.com/JosephT5566/google-ai-gcf/pull/22)|
|Reliability|Timeout、retry、failure isolation、idempotency|[#19](https://github.com/JosephT5566/google-ai-gcf/pull/19)、[#20](https://github.com/JosephT5566/google-ai-gcf/pull/20)|
|Observability|Logs、metrics、tracing、alerts|PR 已加入 version/error signals；production metrics 仍待補|
|Recoverability|Rollback、runbook、backup／restore|目前 PR scope 外，正式上線前需規劃|

Scorecard 不能取代 release gates。Service 即使平均 90 分，只要仍存在 IDOR 或敏感資料洩漏，就不能視為 ready。

---

## 11. 上線後應追蹤的指標

### Service Health

|指標|範例|
|---|---|
|Availability|合法 request 成功取得結果的比例|
|Latency|p50、p95、p99 extraction latency|
|Error Rate|4xx、5xx、GCS failure、Gemini failure|
|Saturation|CPU、memory、concurrency、queue depth|
|Dependency Health|GCS／Gemini latency 與 error rate|

### Business／AI Quality

1. Extraction success rate。
2. `needs_review` rate。
3. `retryable_failure` rate。
4. Auto-accepted receipt 的人工修正率。
5. 不同 model／prompt／schema version 的品質差異。

### Security Signals

1. Cross-user path rejection。
2. Oversized upload rejection。
3. Unsupported MIME／signature mismatch。
4. Schema-invalid provider output。
5. Operational-field／configuration injection attempts。

### SLO 範例

> 99.5% 的合法 receipt requests，在 10 秒內取得非 provider-failure 結果。

實際門檻應由使用情境、Gemini latency、成本與 caller tolerance 校準。

---

## 12. 公司內部服務的規劃方式

Internal service 可以簡化 public-facing product features，但「內部網路」不能取代 authentication。

### 先定義 Service Tier

|Tier|使用情境|建議要求|
|---|---|---|
|Internal Experimental|少數開發者測試|Best effort、business-hours support|
|Internal Shared|多個團隊正式使用|Stable contract、alerts、rollback、基本 SLO|
|Internal Critical|阻塞財務或核心營運|HA、on-call、DR、嚴格 SLO|

Receipt extraction 若只是輔助內部流程，可先視為 `Internal Shared`；若失敗會阻塞報帳或財務流程，則接近 `Internal Critical`。

### Internal Shared 最低要求

1. Service Account／Workload Identity 與 caller allowlist。
2. Strict request schema、image limits 與 provider output validation。
3. Timeout、concurrency limit、quota 與成本保護。
4. Structured logs、metrics、alerts 與 correlation ID。
5. Deployment rollback、service owner 與簡短 runbook。

### Caller Service 與 End User 要分開

如果內部 service 代表使用者呼叫 receipt service，需要區分：

- **Caller service identity**：哪一個 backend service 發出 request。
- **End-user identity**：收據與 GCS object 屬於哪位使用者。

不能直接信任 request 中任意傳入的 `user_id`。

### PR 例子

- [PR #21](https://github.com/JosephT5566/google-ai-gcf/pull/21) 的 ownership check 仍應保留，但 authentication 可由 JWT 調整成 GCP IAM／service identity。
- [PR #20](https://github.com/JosephT5566/google-ai-gcf/pull/20) 的 versioned contract 能避免多個 internal callers 綁死 Gemini response shape。
- [PR #22](https://github.com/JosephT5566/google-ai-gcf/pull/22) 的 prompt-injection 防護仍必須保留，因為圖片內容仍是不可信輸入。

---

## 13. 建議的 Internal Receipt Service Runtime Flow

```
Internal caller
→ 驗證 service identity
→ 檢查 caller allowlist
→ 驗證 end-user / object ownership context
→ 驗證 image path、MIME、size、generation
→ 下載圖片
→ 使用 server-owned Gemini configuration
→ Application-side Pydantic validation
→ Normalize and classify
→ 回傳 receipt-result.v1
→ 記錄 safe metrics、audit event 與 provider cost
```

### Timeout 與 Retry 原則

1. 外層 caller timeout 必須長於 receipt service 的 internal timeout。
2. GCS 與 Gemini 各自設定明確 timeout。
3. 只有 `retryable_failure` 可以自動 retry。
4. 使用 exponential backoff 與最大 retry 次數。
5. 使用 idempotency key 避免重複分析與重複計費。

---

## 14. 目前已完成與下一步

### 已完成

1. Canonical receipt schema 與 application boundary validation。
2. Gemini structured response integration。
3. Versioned、normalized result contract。
4. GCS object ownership 與 image-input hardening。
5. Prompt-injection 與 control-plane override protection。

### Production／Internal Shared 下一步

1. Rate limiting、per-caller quota 與 provider cost protection。
2. Metrics、dashboard、alerts 與 correlation tracing。
3. 真實 Gemini／GCS smoke test。
4. Receipt image retention、privacy 與 deletion policy。
5. Deployment rollback、service owner 與 runbook。

---

## 15. 最終判斷框架

每次準備 release 時，回答四個問題：

1. **Correct**：核心 use case、schema 與 contract 是否正確？
2. **Safe**：Caller 是否只能操作有權限的資料？
3. **Observable**：失敗時能否快速發現、定位與量化？
4. **Recoverable**：部署或 dependency 出問題時能否停止、rollback 或恢復？

判斷方式：

- 功能還沒完成：調整 MVP scope。
- Authorization 或資料保護沒完成：Release blocker。
- 沒有觀測能力：不適合無限制 production release。
- 無法 rollback／恢復：限制流量或延後上線。

> 最實用的 Backend readiness 模型：
> 
> **MVP Scope + Production Readiness Scorecard + Mandatory Safety Gates + 上線後 SLO**

# AI Development Memory

## 核心 Mindset

不要把 AI 的「記憶」理解成聊天紀錄。

應該把知識分成三層：

```
Repo Memory
    ↓
Shared Project Memory
    ↓
Session Memory
```

不同層負責不同的事情。

---

# 1️⃣ Repo Memory（每個 Repo 自己的知識）

> 回答：「這個 repo 是怎麼運作的？」

每個 repository 都維護自己的文件。

例如：

```
expense-web/
├── AGENTS.md
└── docs/
    ├── architecture.md
    ├── domain.md
    ├── conventions.md
    └── decisions/
```

### 保存內容

- Repo 負責什麼
- Folder 結構
- Architecture
- Domain Model
- Coding Convention
- 已確認的設計決策（ADR）
- 常見坑與限制

### 更新時機

✅ 架構改變

✅ Convention 改變

✅ 新增重要決策

❌ 每次聊天都更新

---

# 2️⃣ Shared Project Memory（跨 Repo 的知識）

> 回答：「多個 Repo 如何合作？」

推薦建立一個獨立 repository，例如：

```
project-memory/
```

內容可以包含：

```
project-memory/
├── repository-map.md
├── system-overview.md
├── api-contracts.md
├── event-flow.md
├── glossary.md
└── decisions/
```

### 保存內容

- Repository Map
- System Architecture
- Cross-repo API
- Event Flow
- 共用名詞
- 跨 Repo 的 ADR

例如：

```
expense-web
    ↓
expense-api
    ↓
expense-worker
    ↓
AI Provider
```

---

# 3️⃣ Session Memory（短期工作記憶）

> 回答：「這次做到哪裡？」

例如：

```
Goal

Completed

Pending

Next
```

只保存：

- 工作進度
- Debug 過程
- 未完成事項
- 下一步

功能完成後：

✅ 整理進正式文件

或

🗑️ 刪除

不要永久保存。

---

# 📦 GitHub 如何管理？

## Repo Memory

跟著 Code 一起管理。

```
Commit
    ↓
GitHub
    ↓
Version Control
```

任何人 Clone Repo 都能取得。

---

## Shared Memory

通常建立另一個 Repository。

例如：

```
github.com/company/project-memory
```

所有 Repo 都可以讀取。

它不是程式碼，而是：

- 系統知識
- 架構
- Repository 關係

---

# 🤖 Agent 如何讀取？

不要一次讀所有 Repo。

應該：

```
收到任務
    ↓
讀目前 Repo
    ↓
需要跨 Repo？
    ↓
讀 Shared Memory
    ↓
必要時才讀其他 Repo
```

避免 Context 爆炸。

---

# 📚 哪些文件值得永久保存？

## ✅ 值得

- [AGENTS.md](http://AGENTS.md)
- Architecture
- ADR
- API Contract
- Repository Map
- Domain Model
- Known Pitfalls

---

## ❌ 不值得

- 聊天紀錄
- AI 推測
- Debug Log
- 已完成的 Todo
- 過期 Session

---

# 📈 建立順序（推薦）

不要一開始建立巨大 Knowledge Base。

按照下面順序：

```
Step 1

每個 Repo 建立 AGENTS.md

↓

Step 2

補 Architecture

↓

Step 3

補 ADR

↓

Step 4

出現跨 Repo 功能

↓

建立 project-memory
```

只有真的跨 Repo 時，才開始建立 Shared Memory。

---

# 🧩 推薦目錄

## 每個 Repo

```
AGENTS.md

docs/
├── architecture.md
├── domain.md
├── conventions.md
├── decisions/
└── current-work.md
```

---

## Project Memory Repo

```
project-memory/

├── repository-map.md
├── system-overview.md
├── glossary.md
├── api-contracts.md
├── event-flow.md
└── decisions/
```

---

# ✅ 更新規則（最重要）

每次開發結束，只問三個問題：

### ① Repo 規則改了嗎？

👉 更新 Repo Memory。

---

### ② 跨 Repo 流程改了嗎？

👉 更新 Shared Memory。

---

### ③ 只是這次工作做到哪？

👉 更新 Session Memory。

完成後即可刪除或整理。

---

# 🎯 最後一句話（最好記）

```
Repo Memory
→ 記自己的知識

Project Memory
→ 記彼此的關係

Session Memory
→ 記目前做到哪
```

不要把聊天當記憶。

真正的 Source of Truth 永遠是：

```
Code
    ↓
Repo Memory
    ↓
Project Memory
    ↓
Session Memory
    ↓
Chat History
```