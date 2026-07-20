# PostgreSQL Schema Tradeoff 面試筆記

日期：2026-07-16

主題：正規化、歷史快照、JSONB 與分帳資料建模

## 面試題目

你正在設計多人記帳系統：交易可分攤到多個分類，而分類名稱之後可能被修改，但歷史報表必須保留交易發生當時的名稱。你會如何設計 PostgreSQL schema，並說明「完全正規化」、「交易中保存快照」與 `JSONB` 方案之間的取捨？

延伸情境：如果用 `JSONB` 保存分帳資料，要如何確保分帳者是真實使用者，而且分帳金額總和等於消費總額？

## 我的原始回答

我的初步規劃是：

- `public` schema 有 `expenses`、`categories`。
- `expenses.category_id` 作為 foreign key，連結 `categories`。
- 用 `public.expenses.expense_name` 保存交易發生時的名稱，達到 snapshot 效果。
- 混合使用正規化與 `JSONB`。
- 分帳資料放在 `public.expenses.expense_share JSONB`，避免額外管理 `public.expense_shares` table。
- 使用 application middleware，在寫入 JSONB 前驗證或計算資料。

## 回答評估

評級：**Weak / risky**

整體方向並非完全錯誤。回答已經展現以下概念：

- 知道主要 entity 可以透過 foreign key 正規化。
- 知道 snapshot 能保留歷史資料。
- 知道 relational schema 與 `JSONB` 可以混合使用。
- 開始考慮在 middleware 驗證 JSONB 內容。

主要問題在於需求與欄位沒有正確對應。原題需要保存「分類當時的名稱」，`expense_name` 保存的卻是「消費名稱」。分類改名後，單靠 `expense_name` 無法重建當時的分類名稱。

另一個缺口是選擇 `JSONB` 時，只提到少管理一張 table，尚未說明會失去哪些資料庫保證，例如 foreign key、欄位型別、唯一性、個別更新、查詢效率與併發控制。

## 建議的 Schema

如果一筆消費只有一個分類，可以使用：

```sql
CREATE TABLE public.categories (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES auth.users(id),
  name text NOT NULL,
  UNIQUE (user_id, name)
);

CREATE TABLE public.expenses (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES auth.users(id),
  category_id uuid REFERENCES public.categories(id),
  category_name_snapshot text NOT NULL,
  expense_name text NOT NULL,
  amount numeric(12, 2) NOT NULL CHECK (amount >= 0),
  occurred_at timestamptz NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

這裡兩個名稱的意義不同：

- `expense_name`：使用者輸入的消費名稱，例如「團隊晚餐」。
- `category_name_snapshot`：建立交易時的分類名稱，例如「餐飲」。

`category_id` 保留與目前分類 entity 的關係，`category_name_snapshot` 則保留歷史語意。

如果一筆消費可以分配到多個分類，snapshot 應放在 join table：

```sql
CREATE TABLE public.expense_allocations (
  id uuid PRIMARY KEY,
  expense_id uuid NOT NULL REFERENCES public.expenses(id) ON DELETE CASCADE,
  category_id uuid REFERENCES public.categories(id),
  category_name_snapshot text NOT NULL,
  amount numeric(12, 2) NOT NULL CHECK (amount >= 0)
);
```

## 分帳資料的建模選擇

### 選項 A：正規化的 `expense_shares` table

```sql
CREATE TABLE public.expense_shares (
  id uuid PRIMARY KEY,
  expense_id uuid NOT NULL REFERENCES public.expenses(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES auth.users(id),
  amount numeric(12, 2) NOT NULL CHECK (amount >= 0),
  payment_status text NOT NULL
    CHECK (payment_status IN ('pending', 'paid')),
  UNIQUE (expense_id, user_id)
);
```

優點：

- PostgreSQL 可以直接維護 `expense_id` 與 `user_id` 的 foreign key。
- `numeric` 與 `CHECK` constraint 能約束金額。
- `UNIQUE (expense_id, user_id)` 能避免同一人重複分帳。
- 容易查詢某位使用者參與的所有分帳。
- 容易更新單一使用者的付款狀態。
- 比較容易使用 transaction、row lock 與索引處理併發和效能。

成本：

- 多一張 table、migration 與 join。
- 寫入一筆 expense 可能需要新增多筆 share rows。
- 必須用 transaction 確保 expense 與 shares 一起成功或失敗。

### 選項 B：`expenses.expense_share JSONB`

範例資料：

```json
[
  { "user_id": "user-1", "amount": 60, "status": "paid" },
  { "user_id": "user-2", "amount": 40, "status": "pending" }
]
```

優點：

- Prototype 階段開發快速。
- 結構容易增加暫時性欄位。
- 如果每次都隨 expense 整體讀寫，可減少應用層組裝資料的工作。

風險：

- 無法直接對陣列中的每個 `user_id` 建立 foreign key。
- JSON 中的 `amount` 不會自動具有 `numeric` 欄位的型別保證。
- 跨 expense 查詢某位使用者的所有 shares 較複雜。
- 更新其中一個 participant 容易變成整份 JSONB 的 read-modify-write。
- 兩個請求同時修改同一份 JSONB 時，可能發生 lost update。
- DB constraint、索引和 migration 都比 relational columns 複雜。

### 選擇原則

若分帳成員、金額與付款狀態是穩定的核心業務資料，優先使用 `expense_shares` table。若資料結構仍高度不確定、只隨 expense 整體存取，而且目前只是小規模 prototype，則可暫時使用 `JSONB`，但要明確接受完整性與查詢成本。

## PostgreSQL 對 JSONB 的運算

PostgreSQL 可以展開 JSONB array 並加總金額：

```sql
SELECT COALESCE(SUM((share->>'amount')::numeric), 0)
FROM jsonb_array_elements($1::jsonb) AS share;
```

也可以在 trigger 中驗證分帳總額：

```sql
CREATE FUNCTION validate_expense_shares()
RETURNS trigger AS $$
DECLARE
  share_total numeric;
BEGIN
  SELECT COALESCE(SUM((share->>'amount')::numeric), 0)
  INTO share_total
  FROM jsonb_array_elements(NEW.expense_share) AS share;

  IF share_total <> NEW.amount THEN
    RAISE EXCEPTION 'Share total must equal expense amount';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER validate_expense_shares_before_write
BEFORE INSERT OR UPDATE OF expense_share, amount
ON public.expenses
FOR EACH ROW
EXECUTE FUNCTION validate_expense_shares();
```

Trigger 也能查詢 `auth.users` 來驗證 JSONB 中的 `user_id`，但這是手動重建 foreign key 的部分行為，會增加權限、效能、測試與維護成本。

## Middleware 與 Database 的責任

Application middleware 可以：

- 驗證 request shape。
- 提前計算分帳總額。
- 回傳對使用者友善的錯誤訊息。
- 正規化資料後再執行 `INSERT`。

但 middleware 不應是唯一防線，因為 SQL console、migration、background job 或另一個 API 都可能繞過它。

較穩健的責任分配是：

- Middleware：輸入驗證與友善錯誤。
- Database constraint／foreign key：核心 relational invariants。
- Database trigger：無法用一般 constraint 表達，但必須在所有寫入路徑成立的規則。
- Transaction：確保 expense、allocations 與 shares 的多筆寫入具備 atomicity。

## 改善後的面試回答

> 我會混合使用正規化與 snapshot。主要 schema 包含 `public.expenses` 和 `public.categories`，由 `expenses.category_id` foreign key 連到 `categories.id`。
> 
> 因為分類名稱之後可能修改，但歷史報表必須顯示交易發生時的名稱，所以我會另外保存 `category_name_snapshot`。`category_id` 用來維持與分類 entity 的關聯，snapshot 則保留歷史語意。建立 expense 時，我會在同一個 transaction 中取得分類名稱並寫入 snapshot。
> 
> 分帳部分，我傾向建立 `expense_shares` table，包含 `expense_id`、`user_id`、`amount` 和付款狀態，並建立 foreign key、unique constraint 與金額 constraint。這讓 PostgreSQL 能維持 relational integrity，也方便查詢個別使用者的待付款項。
> 
> 如果產品仍是早期 prototype，而且分帳結構尚未穩定、資料只會隨整筆 expense 一起讀寫，我可以先使用 JSONB。但代價是無法直接對內部 user ID 建立 foreign key，型別驗證、個別查詢與併發更新也會更困難。即使 application middleware 已驗證資料，我仍會讓 database 保護重要 invariant，避免其他寫入路徑繞過驗證。

## 建議的回答結構

面試時可以依序回答：

1. 先直接說選擇，例如「核心關聯正規化，歷史名稱使用 snapshot」。
2. 列出 tables、關鍵 columns、primary keys 與 foreign keys。
3. 解釋 snapshot 保存的是哪個欄位以及原因。
4. 比較另一個合理方案，不只描述自己的選擇。
5. 說明 constraint、transaction 與併發寫入行為。
6. 最後補充規模或需求改變時，何時會重新選擇 JSONB 或 relational table。

不需要一開始就寫完整 `SELECT`。SQL 可以用來證明 schema 支援需求，但面試官更在意資料模型、invariants 與 tradeoffs。

## 本題展現的強項

- 能辨識正規化與 JSONB 可以混合使用。
- 已有 foreign key 與 snapshot 的基本概念。
- 會從減少 schema 複雜度與開發速度思考方案。
- 主動想到 middleware 計算與驗證。

## 本題需要加強的地方

- 先精準對應需求中的 entity 與欄位，避免把 `expense_name` 和 category snapshot 混為一談。
- 選擇 JSONB 時要主動說明失去的 database guarantees。
- 不只描述資料如何存，也要說明非法資料如何被阻止。
- 補充 concurrency、transaction 與所有寫入路徑都必須成立的 invariant。

## 接下來三個練習優先項目

1. 練習從需求辨識 entity、relationship、snapshot 與 source of truth。
2. 練習比較 normalized table 與 JSONB 在 constraint、query、index、migration 和 concurrency 上的差異。
3. 練習用 database constraint、trigger 與 transaction 表達核心業務 invariant。

## 一句話記憶

**固定且重要的關聯資料優先正規化；需要保留歷史語意時保存明確 snapshot；JSONB 的彈性是用資料庫保證與查詢便利性換來的。**

# API Contract Design 面試練習筆記

日期：2026-07-17

## 題目

你的 expense app 要開放「上傳收據並由 AI 解析成支出草稿」的 API。解析可能耗時數秒、部分欄位可能無法辨識，而且 client 可能因 timeout 重送請求。

請設計這個 API contract，並說明如何定義成功、部分成功與失敗的語意。

## 討論過程與回答回顧

### 1. 初始 API contract

我的回答：

> API 採用 REST，response 使用 JSON，並提供文件說明。無法辨識的欄位設為 `undefined`；發生錯誤時回傳 HTTP error code，例如 timeout 回 `408`。另外考慮是否由使用者在呼叫 API 時提供 prompt。

評價：**Weak / risky**

方向正確的部分：

- 選擇 REST 與 JSON 作為清楚、常見的介面形式。
- 知道 contract 需要文件化。
- 有意識到欄位可能無法辨識，以及 timeout 需要明確語意。

需要修正的部分：

- `undefined` 不是合法 JSON value。無法辨識的欄位應省略或設為 `null`，並提供辨識狀態。
- HTTP status 與 application error code 應分開設計。
- `408 Request Timeout` 通常表示 server 等待 client 完成 request 時逾時；AI upstream timeout 較可能是 `504 Gateway Timeout`，或在非同步 job 中標記失敗。
- 尚未定義非同步處理、retry、重複 request 與部分成功的 contract。
- Task-specific API 通常不應開放 raw prompt。應由 server 管理、版本化 prompt，client 僅傳入受控的結構化 hints，例如 locale 或 expected currency。

### 2. Idempotency 基本策略

我的回答：

> 檢查 `Idempotency-Key`。如果已存在，就回傳第一次請求的結果；如果不存在，則執行請求並儲存該 key。

評價：**Acceptable but incomplete**

方向正確的部分：

- 相同的 idempotency key 不應再次啟動相同工作。
- 願意保存 key 與第一次執行結果。

需要補充的部分：

- 第一次請求可能仍在執行，因此不一定已有最終結果；應回傳同一個 job resource 與目前狀態。
- 相同 key 搭配不同 payload 時，應拒絕而不是回傳舊結果。
- key 應綁定 authenticated user、endpoint 和 retention period。
- 「檢查後再新增」會有 race condition，必須依賴 database unique constraint 與 atomic insert。

### 3. 多台 server 同時收到相同 request

我的回答：

> 建立 `idempotency_requests` table，以 `Idempotency-Key` 和 `user_id` 作為 constraint。沒有衝突時成功 insert；衝突時 `DO NOTHING`。只有成功 insert 並取得回傳值的 request 才建立 parsing job。Idempotency record 與 parsing job 放在同一個 transaction，確保一起成功。

評價：**Strong**

這個回答已抓到跨 server instance 的核心：

- Database unique constraint 是**一致性**的仲裁者。
- `INSERT ... ON CONFLICT DO NOTHING RETURNING ...` 可以決定哪一個 request 取得執行權。
- Idempotency record 與 job record 在同一個 transaction 建立，避免只建立其中一筆。

仍可補充：

- 發生 conflict 時，需要讀取既有 record 並比較 payload hash。
- 相同 key、不同 payload 應回 `409 Conflict`。
- 實際 AI parsing 不應放在 database transaction 中，避免長時間占用 connection 與 lock。

### 4. Transaction commit 後、交給 worker 前 crash

我的回答：

> 服務重啟時檢查 parsing job 狀態。Worker 完成後會寫回 parsing job table，因此可以找到未完成 job 並重新執行。

評價：**Acceptable but incomplete**

方向正確的部分：

- 將 database job table 視為 durable source of truth。
- 可以藉由 job status 找回未完成工作。

需要補充的部分：

- 不能只在服務重啟時掃描，否則服務一直存活時，遺失的 job 可能永久卡住。
- 多個 worker 需要以 atomic claim 或 `SELECT ... FOR UPDATE SKIP LOCKED` 避免重複領取。
- `processing` job 需要 lease，例如 `lease_expires_at`，以判斷 worker 是否已失聯。
- Background sweeper 應定期回收 lease 過期的 job。
- 應有 `attempt_count`、retry limit、backoff、last error 與永久失敗狀態。

### 5. Provider 已成功，但結果尚未寫回 database 就 crash

我的回答：

> 先確認 provider 的能力。如果 provider 支援 idempotency key，重試時帶相同 key，避免重複執行與收費。如果 provider 會回傳 request ID，worker 重啟後可查詢結果。如果額外費用較高，可以改成人工確認，或設定費用上限。

評價：**Strong**

這個回答展現了良好的 production judgment：

- 先根據 provider 提供的 guarantee 設計整合方式。
- 優先使用 provider idempotency。
- 以 provider request ID 進行 reconciliation。
- 無法保證 exactly-once 時，誠實採用 bounded retry、人工處理與成本上限。

需要注意：如果 request ID 是 provider 回應後才取得，worker 仍可能在收到 ID、但尚未寫入 database 前 crash，因此 request ID 無法完全關閉 failure window。

## 建議的 API contract

### 建立解析工作

```
POST /v1/receipt-parses
Authorization: Bearer <token>
Idempotency-Key: 01J...
Content-Type: application/json
```

```json
{
  "receiptFileId": "file_123",
  "locale": "zh-TW",
  "expectedCurrency": "TWD"
}
```

Server 驗證並建立非同步工作後回：

```
HTTP/1.1 202 Accepted
Location: /v1/receipt-parses/rp_123
```

```json
{
  "id": "rp_123",
  "status": "queued",
  "createdAt": "2026-07-17T10:00:00Z"
}
```

### 查詢解析狀態

```
GET /v1/receipt-parses/rp_123
```

仍在處理：

```json
{
  "id": "rp_123",
  "status": "processing",
  "attempt": 1
}
```

完成且有部分欄位無法辨識：

```json
{
  "id": "rp_123",
  "status": "completed",
  "result": {
    "merchant": {
      "value": "Example Store",
      "status": "recognized",
      "confidence": 0.97
    },
    "total": {
      "value": 420,
      "currency": "TWD",
      "status": "recognized",
      "confidence": 0.94
    },
    "purchasedAt": {
      "value": null,
      "status": "unrecognized",
      "confidence": 0.18
    }
  }
}
```

「部分欄位無法辨識」仍可視為解析工作成功，因為 job 已正常完成；欄位層級的 `status` 負責表達部分成功，而不是將整個 request 回成 HTTP error。

永久失敗：

```json
{
  "id": "rp_123",
  "status": "failed",
  "error": {
    "code": "PARSING_PROVIDER_UNAVAILABLE",
    "message": "We could not process this receipt.",
    "retryable": true
  }
}
```

不要把 provider stack trace、原始錯誤訊息、內部 model 名稱或 credential 資訊暴露給 client。詳細內容應保留在 internal logs，並以 trace ID 或 job ID 串聯。

### 建議的 HTTP 與錯誤語意

|情境|HTTP status / job status|說明|
|---|---|---|
|Job 建立成功|`202 Accepted`|解析尚未完成|
|Request JSON 格式錯誤|`400 Bad Request`|無法解析 request|
|欄位格式正確但內容不可處理|`422 Unprocessable Content`|例如不支援的檔案類型|
|未登入或無權限|`401` / `403`|Authentication 與 authorization 分開|
|相同 key、不同 payload|`409 Conflict`|防止錯誤重用 idempotency key|
|AI upstream 同步逾時|視架構使用 `504`|若採 async job，通常記錄為 job retry/failure|
|部分欄位無法辨識|Job `completed`|在欄位層級標示 `null` 與辨識狀態|
|超過 retry 上限|Job `failed`|回穩定 application error code|

## Idempotency 資料設計

概念上的欄位：

```
idempotency_requests
- user_id
- endpoint
- idempotency_key
- request_hash
- job_id
- status
- response_status
- response_body
- created_at
- expires_at

UNIQUE (user_id, endpoint, idempotency_key)
```

處理流程：

1. 對 canonicalized request payload 計算 hash。
2. 開啟 database transaction。
3. 使用 `INSERT ... ON CONFLICT DO NOTHING RETURNING id` 建立 idempotency record。
4. Winner 在同一 transaction 建立 parsing job，並把 `job_id` 關聯回 record。
5. Commit 後才由 worker 執行 AI parsing。
6. Loser 讀取既有 record：hash 相同則回同一個 job；hash 不同則回 `409 Conflict`。

要避免先 `SELECT`、再 `INSERT` 的 check-then-act 寫法，因為兩台 server 可能同時查到「不存在」，然後各自建立一個 job。

## Parsing Job 的可靠性設計

### 核心 mindset

分散式系統無法完全避免 timeout、crash 與 retry。與其追求「絕不重複執行」，更實際的目標是讓工作即使重複執行，也不會產生錯誤或重複副作用。

可以將整體保證記成：

```
可靠持久化 -> 至少執行一次 -> 接受重試 -> 每一層都保持冪等
```

這裡要區分兩類問題：

- **Database polling 或 message queue** 解決「工作不能遺失」。
- **Idempotency-Key、unique constraint 與冪等 consumer** 解決「重複工作不能產生重複副作用」。

### 各層的可靠性機制

|機制|解決的問題|重要語意|
|---|---|---|
|`Idempotency-Key`|Client timeout 後重送，可能重複建立 job|同一個 user operation 重試時沿用相同 key；相同 key、不同 payload 應拒絕|
|Unique constraint + transaction|多台 server 同時收到相同操作|由 database 仲裁 winner；idempotency record 與 job 要嘛一起 commit，要嘛一起 rollback|
|Durable job table|API server commit 後、通知 worker 前 crash|Job 先以 `queued` 狀態持久化，再由 worker 持續 polling，不依賴 API server 成功通知或重新啟動|
|Lease + retry|Worker 領取工作後 crash|將 job 標記為 `processing` 並設定 lease；到期後可由其他 worker 重新領取|
|冪等結果寫入|多個 attempt 同時或先後完成|對 parsing result 的 `job_id` 設 unique constraint，並以 atomic insert/update 避免產生多筆結果|
|Message queue|改善任務分派、backpressure、retry、擴展與監控|Queue 通常仍是 at-least-once delivery，因此 consumer 必須冪等|
|Transactional outbox|Database commit 成功，但 publish queue event 失敗|在同一個 transaction 寫入 job 與 outbox event，再由 relay 重試發布|
|Provider idempotency|Provider 成功後，本地 worker 在儲存結果前 crash|優先使用 provider idempotency key 或可查詢的 provider job ID|

狀態轉換可以設計為：

```
queued -> processing -> completed
                    -> queued (retry)
                    -> failed
```

`queued` 也可以命名為 `pending`，但 database、API contract 與 monitoring 必須一致。

Worker 領取工作時應使用 atomic claim，例如 conditional update 或 `SELECT ... FOR UPDATE SKIP LOCKED`，並寫入：

- `lease_expires_at`
- `attempt_count`
- `started_at`
- `worker_id`
- `last_error_code`

Background sweeper 應持續尋找：

- 長時間停在 `queued` 的 job。
- Lease 已過期的 `processing` job。

因此 recovery 不應只在服務重啟時執行。服務即使一直沒有重啟，卡住的 job 也必須能被偵測並重新處理。

### Database polling 與 message queue

小型系統可以直接使用 database-backed job table：worker 持續 polling 並以 atomic claim 領取工作。它的優點是元件少，而且 job 建立與業務資料可以使用同一個 transaction。

當 workload、backpressure 或 worker scaling 需求增加時，可以加入 message queue。Queue 能改善 delivery、retry scheduling、監控與 worker 擴展，但不能取代 consumer idempotency，因為多數 queue 提供的是 **at-least-once delivery**。

若 database 是 job 的 source of truth，但 API server 在 commit 後直接 publish message，會出現「database 已成功、message 尚未送出就 crash」的 dual-write failure。此時可使用 **Transactional Outbox**：在建立 job 的同一個 transaction 寫入 outbox event，再由獨立 relay 持續 publish；重複發布仍由冪等 consumer 處理。

### AI provider 的 failure window

整體設計通常提供的是 **at-least-once execution**，不是 exactly-once。只要流程橫跨本地 database 與外部 provider，就存在無法用單一 transaction 關閉的 failure window。

最棘手的情境是：provider 已成功完成並可能已收費，但 worker 在把結果寫回本地 database 前 crash。Recovery worker 只看到 job 尚未完成，無法確定 provider 是否已經執行。

降低風險的優先順序：

1. 使用 provider idempotency key，並以穩定的 internal job ID 當 key。
2. 保存並使用 provider async job ID 查詢既有結果。
3. 執行 reconciliation，核對 provider 與本地狀態。
4. 若 provider 不提供上述能力，明確在「可能重複呼叫與付費」和「可能讓工作不完成」之間取捨。
5. 使用有限 retry、exponential backoff、費用上限、監控與人工 review 控制最壞情況。
6. 將「建立 expense」設計成另一個有 unique constraint 或 idempotency protection 的操作，避免重複解析進一步建立重複支出。

對 parsing result 的 `job_id` 設 unique constraint，只能避免本地產生多筆結果；它無法阻止外部 provider 被重複呼叫。因此每一個 side-effect boundary 都需要自己的 idempotency 或 reconciliation 策略。

## 一個較完整的面試回答

> 我會將 receipt parsing 設計成非同步 resource。Client 呼叫 `POST /v1/receipt-parses`，帶 receipt file ID、locale、expected currency，以及 `Idempotency-Key`。Server 驗證成功後回 `202 Accepted`，並提供固定的 parsing job ID；client 再透過 `GET /v1/receipt-parses/{id}` 查詢狀態。
> 
> 我不會讓 client 傳 raw prompt，因為這是 task-specific API。Prompt 由 server 控制與版本化，client 只提供受控 hints，避免 contract 漂移與 prompt injection。
> 
> 解析完成但有欄位無法辨識時，job 仍是 `completed`。無法辨識的欄位使用合法 JSON 的 `null`，並附上欄位狀態與 confidence。真正的系統失敗則把 job 標記為 `failed`，對 client 回穩定且不洩漏 provider 細節的 application error code。
> 
> 為處理 client retry，我會建立 idempotency table，對 `(user_id, endpoint, idempotency_key)` 設 unique constraint，並保存 request hash。Idempotency record 與 parsing job 在同一個 transaction 中建立；相同 key 與 payload 回同一個 job，不同 payload 則回 `409`。
> 
> Worker 透過 atomic claim 與 lease 處理 job，sweeper 會回收卡住的工作，因此系統提供 at-least-once execution。對外部 AI provider，我會優先使用 provider idempotency 或 request ID reconciliation；如果 provider 無法提供 exactly-once guarantee，就使用 bounded retry、監控、人工處理與費用上限控制風險。

## 這次展現的優勢

- 很快理解 idempotency key 的用途，並能從 API 層一路推進到 database constraint。
- 能提出 transaction boundary，確保 idempotency record 與 job 一起建立。
- 面對跨 provider 的 failure window，沒有武斷宣稱 exactly-once，而是先確認 provider guarantee。
- 有成本意識，能把 retry 次數、人工確認與費用上限納入 production decision。

## recurring gaps

- 初始回答偏向列出技術形式，例如 REST、JSON、HTTP code，但沒有先定義 resource、狀態與語意。
- 對 HTTP 與 JSON 細節需要更精確，例如 `undefined`、`408`、部分成功與 application error code。
- Recovery 最初集中在「服務重啟後處理」，可以更主動想到持續 sweeper、lease、concurrent worker 與 stuck job。

## 接下來三個練習重點

1. **Contract-first 表達**：先說 resource、endpoint、request、response、狀態轉換，再補 implementation。
2. **Failure window 推演**：每次設計跨系統流程，都依序問「寫入前 crash、寫入後 crash、回應前 crash、重試時會怎樣」。
3. **語意精準度**：熟悉 JSON nullability、常見 HTTP status、idempotency scope、payload hash，以及 at-least-once 和 exactly-once 的界線。

# Full-stack Error Handling 面試筆記

## 題目

使用者在前端提交一筆報銷資料；API 需要驗證輸入、寫入 PostgreSQL，並呼叫第三方 AI 解析收據。AI 偶爾逾時，而使用者也可能因畫面沒有反應而重複點擊。

請設計端到端的 error handling：從前端呈現、API 錯誤契約，到後端如何處理逾時、重試、重複請求與資料一致性。

後續延伸題：如果 expense 已經寫入資料庫，但服務在 job 成功送進 queue 前 crash，如何保證這筆 expense 最終一定會被處理。

---

## 我的回答整理

### 1. 先可靠接收，再非同步處理 AI

- API 穩定地接收請求、驗證資料並寫入 PostgreSQL。
- AI parsing 不放在同步 request 中等待，而是交由 background worker 處理。
- API 接受工作後回傳 `202 Accepted`，前端顯示「處理中」。
- AI 最終失敗時，保留使用者原本送出的 expense data，讓使用者可以手動修正，不必重新提交整筆內容。

### 2. 前端錯誤處理

- 使用者送出後暫時 disable button，降低短時間內重複點擊的機會。
- 收到 `202 Accepted` 後顯示處理中狀態。
- 遇到可重試錯誤，例如 `503 Service Unavailable`，提示使用者稍後再試。
- 前端重送同一筆操作時沿用相同的 `Idempotency-Key`。
- 最終解析失敗時顯示「自動解析失敗」，並提供手動填寫或修正流程。

### 3. 穩定的 API 錯誤契約

- 不讓前端直接解析 PostgreSQL 或 AI provider 的錯誤碼與訊息。
- 對外提供系統自己的穩定 error code。
- 優先處理會改變前端行為的錯誤，例如：
    - `400`：輸入錯誤
    - `401`：尚未驗證身分
    - `403`：沒有操作權限
    - `409`：狀態或 idempotency 衝突
    - `429`：請求過多，延後重試
    - `500`：未預期錯誤，僅在操作可安全重送時有限度重試
    - `503`：暫時無法服務，可稍後重試

### 4. Timeout 與 retry

- AI call 必須有明確 timeout，不能無限等待。
- Worker 根據 AI provider 的錯誤類型判斷是否可以 retry。
- Retry 還需要同時考慮 job 狀態、已嘗試次數及 `next_retry_at`。

### 5. Idempotency 與資料一致性

- Client request 帶入 `Idempotency-Key`，避免重複建立 submission。
- `expense_submissions` 與 `parsing_jobs` 在同一個 database transaction 中建立。
- AI call 不放在 database transaction 裡，避免長交易、鎖定資源以及把外部服務的不確定性帶進交易。

### 6. Transactional Outbox

- 若系統需要將工作可靠地發送到外部 queue，可建立 `outbox_events` table。
- 在同一個 transaction 中寫入 `expense_submissions`、`parsing_jobs` 與 `outbox_events`，一起 commit 或一起 rollback。
- Backend dispatcher 持續尋找尚未發布且已到重試時間的 outbox event，再將它們送到 queue。

---

## 核心設計原則

### 先可靠接收資料，再處理不穩定的依賴

同步 API 的主要責任是：

1. 驗證身分、權限與輸入。
2. 保證同一 logical request 不會重複建立資料。
3. 持久化 submission 與 background job。
4. 快速回覆 client 已接受工作。

AI parsing 是耗時且不穩定的外部工作，應由 background worker 執行。這可縮短 API response time，也讓 timeout、retry、backoff 與人工修正更容易管理。

```
Client
  │  POST /expense-submissions + Idempotency-Key
  ▼
API ── transaction ──► expense_submissions + parsing_jobs
  │
  └── 202 Accepted + submission_id + status

Dispatcher / Scheduler ──► Worker ──► AI provider
                              │
                              └── update parsing job and submission
```

### 前端重送與 Worker 重試是兩件事

|機制|觸發者|目的|判斷依據|
|---|---|---|---|
|Client retry|前端或使用者|確認原始提交是否被 API 接收|相同 `Idempotency-Key`、HTTP status、application error code|
|Worker retry|後端 worker|重新執行暫時失敗的 AI 工作|provider error、job status、attempt count、`next_retry_at`|

前端重送不應直接建立新的 background job。後端收到相同 key 與相同 payload 時，應回傳既有 submission 及其狀態。

後端是狀態的最終決策者。前端可以要求 retry，但如果 job 正在執行、已完成、屬於不可重試錯誤，或已超過最大嘗試次數，後端應拒絕狀態轉換並回傳穩定的 error code。

### Disable button 不是冪等性保證

Disable button 能改善 UX，但無法處理以下情況：

- 使用者重新整理頁面。
- 多個分頁同時送出。
- Client 在收到 response 前發生 network timeout。
- Proxy、SDK 或使用者重送相同 request。

真正的防重必須由後端與 database constraint 保證。

---

## 狀態與資料模型

業務資料、背景工作狀態與事件發布狀態應分開管理，因為它們有不同的生命週期。

### `expense_submissions`：業務資料

保存使用者提交的原始 expense，以及目前可呈現給使用者的整體狀態。AI 失敗不應刪除這筆資料。

可能狀態：

```
processing → completed
           → needs_manual_input
```

### `parsing_jobs`：背景工作執行狀態

保存 AI job 的執行與恢復資訊，例如：

- `job_id`
- `submission_id`
- `status`: `pending | processing | retry_scheduled | succeeded | failed`
- `attempt_count`
- `next_retry_at`
- `locked_by` / `lease_expires_at`
- 安全且經正規化的 failure category
- `created_at` / `updated_at`

狀態轉換應由後端以條件更新控制，避免兩個 worker 同時把同一個 job 從 `pending` 轉為 `processing`。

### `outbox_events`：事件發布狀態

只有在需要可靠發布到外部 queue 時才需要，例如：

- `event_id`
- `aggregate_id` 或 `job_id`
- `event_type`
- `payload`
- `status`: `pending | publishing | published | failed`
- `attempt_count`
- `next_attempt_at`
- `lease_expires_at`
- `published_at`

Outbox 保存的語意是「這個事件接下來必須被發布」，不是 job 本身的完整執行狀態。

---

## Idempotency 設計

建議將 idempotency record 綁定：

- authenticated user 或 tenant
- API endpoint / operation
- `Idempotency-Key`
- request payload fingerprint
- 建立出的 `submission_id`
- response status 或可重建 response 所需的資訊

處理規則：

1. 新 key：建立 submission 與 job，回傳 `202`。
2. 相同 key、相同 payload：回傳既有 submission 與目前狀態，不重複建立。
3. 相同 key、不同 payload：回傳 `409 IDEMPOTENCY_CONFLICT`。
4. 多個 request 同時使用相同 key：由 unique constraint 與 transaction 決定唯一結果。

前端必須保存同一 logical operation 的 key。只有使用者明確開始一筆全新的 submission 時才產生新 key。

---

## API contract

不需要在第一版預先列出所有可能的 error code。優先定義會讓前端採取不同行為的錯誤。

範例格式：

```json
{
  "error": {
    "code": "SERVICE_TEMPORARILY_UNAVAILABLE",
    "message": "The request could not be completed right now.",
    "request_id": "req_...",
    "retryable": true,
    "retry_after_seconds": 30
  }
}
```

設計重點：

- 前端依賴穩定的 `error.code`，不要解析 message 文字。
- Message 應可安全顯示，不包含 SQL、stack trace、provider response 或敏感資料。
- `request_id` 用於串聯 client、API、dispatcher、worker 與 provider log。
- `429` 或 `503` 可搭配 HTTP `Retry-After`。
- HTTP status 表達通用語意，application code 表達產品內的具體行為。
- `500` 不代表一定可以重試；必須先確定操作具有冪等保護。

可能優先定義的 application codes：

|HTTP|Application code|前端行為|
|---|---|---|
|400|`VALIDATION_FAILED`|顯示欄位錯誤，不自動重試|
|401|`AUTHENTICATION_REQUIRED`|引導重新登入|
|403|`OPERATION_FORBIDDEN`|顯示無權限，不重試|
|409|`IDEMPOTENCY_CONFLICT`|停止重送並回報衝突|
|409|`JOB_ALREADY_RUNNING`|顯示既有處理狀態|
|409|`JOB_NOT_RETRYABLE`|提供手動修正流程|
|429|`RATE_LIMITED`|依 `Retry-After` 延後|
|500|`INTERNAL_ERROR`|安全且有限度地重送或提示稍後再試|
|503|`SERVICE_TEMPORARILY_UNAVAILABLE`|保留相同 key，稍後重送|

---

## Retry 與 timeout

### 可重試的典型情況

- Network timeout 或 connection reset。
- Provider `429`，並遵守其 `Retry-After`。
- 部分暫時性的 provider `5xx`。

### 通常不可重試的情況

- 輸入格式不合法。
- 不支援或已損壞的檔案。
- 認證或權限設定錯誤；這通常需要人工修復設定。
- 已被業務規則取消或已進入 terminal state 的 job。

### Retry policy

- 設定每次 AI call 的 timeout。
- 使用 exponential backoff 加 jitter，避免大量 job 同時重試。
- 設定最大嘗試次數或最大 retry window。
- 每次嘗試記錄開始時間、結束時間、結果分類與 provider latency。
- 超過限制後進入 `failed`，將 submission 轉為 `needs_manual_input`。
- 只有在明確了解 provider 操作語意後，才能安全重送可能具有 side effect 的外部操作。

---

## Transaction 與 Outbox

### Database transaction 能保護什麼

`expense_submissions` 與 `parsing_jobs` 應在同一個 transaction 中建立：

```
BEGIN
  INSERT expense_submission
  INSERT parsing_job
COMMIT
```

這能確保不會出現只有 expense、卻沒有任何 job record 的中間狀態。

AI call 不應放入 transaction。外部呼叫可能很慢或失敗，會造成 transaction 長時間持有 connection 與 lock，而且 database rollback 也無法撤銷已發生的外部 side effect。

### Outbox 解決的問題

當資料庫與外部 queue 是兩套無法共同 commit 的系統時，直接執行以下 dual write 會有缺口：

```
1. COMMIT database
2. publish queue message
```

若服務在兩步之間 crash，資料已存在但 message 遺失。Transactional outbox 改為：

```
Database transaction:
  INSERT expense_submission
  INSERT parsing_job
  INSERT outbox_event

Dispatcher:
  read pending outbox event
  publish to queue
  mark event as published
```

多個 dispatcher 可使用 `SELECT ... FOR UPDATE SKIP LOCKED`，或使用具有到期時間的 lease 來 claim event。發布失敗時更新 `attempt_count` 與 `next_attempt_at`。

### Outbox 仍然不是 exactly-once

Dispatcher 可能已將 message 成功送入 queue，卻在標記 `published` 前 crash。重新啟動後，它會再次發布同一事件。

因此合理的保證是：

```
at-least-once delivery + idempotent processing
```

Queue consumer 應使用穩定的 `event_id` 或 `job_id` 去重，並以原子狀態轉換確認 job 是否仍可執行。不要將「queue 收到一次」或「worker 只啟動一次」當成正確性的前提。

### MVP 是否需要 Outbox

不一定。

- 如果 worker 可以直接 polling PostgreSQL 的 `parsing_jobs`，database 本身已是可靠的 job source，MVP 可以先不建立 outbox。
- 當系統真的需要可靠地把事件發布到外部 queue、event bus 或其他服務時，再加入 outbox 解決 dual-write 問題。
- 是否使用 outbox 應由可靠性需求與架構邊界決定，而不是把它當成所有非同步工作的預設元件。

---

## Failure scenarios 與恢復方式

|Failure scenario|預期結果|恢復方式|
|---|---|---|
|Client request timeout，但 API 已 commit|不建立第二筆 submission|相同 idempotency key 查回既有狀態|
|使用者重複點擊或多分頁送出|只有一筆 logical submission|unique constraint + idempotency lookup|
|AI provider timeout|Job 保持可恢復|設定 `next_retry_at`，有限度重試|
|AI 永久失敗|原始 expense 仍存在|`needs_manual_input`，讓使用者修正|
|Worker 執行中 crash|Lease 到期後可再次取得|at-least-once + idempotent worker|
|DB commit 後、queue publish 前 crash|Event 不遺失|transactional outbox + dispatcher|
|Queue publish 成功、標記 published 前 crash|Message 可能重複|consumer 依 event ID / job ID 去重|
|Dispatcher 或 worker backlog 增加|使用者仍能看到真實狀態|監控 queue depth、oldest age 與 processing duration|

---

## 可觀察性與營運

至少應追蹤：

- API acceptance rate、latency 與 error rate。
- Idempotency hit 與 conflict 數量。
- Pending job 數量與 oldest pending age。
- Job success rate、retry rate、terminal failure rate。
- AI provider latency、timeout rate 與各 error category。
- 長時間停留在 `processing` 的 job。
- Pending outbox event 數量、oldest unpublished age 與 publish failure rate。
- 使用者最終改用 manual input 的比例。

使用 `request_id`、`submission_id`、`job_id` 與 `event_id` 串聯 structured logs 與 traces。告警應針對可採取行動的症狀，例如 backlog age 超過目標，而不只針對單次失敗。

---

## 面試評價

### 整體評價：Acceptable but incomplete

回答的方向是正確的，已展現以下能力：

- 知道將同步 API 與不穩定的 AI 處理解耦。
- 能區分 client retry 與 backend retry。
- 理解 idempotency、timeout、錯誤分類與 graceful degradation。
- 知道 database transaction 與 transactional outbox 各自解決的問題。
- 沒有為了追求 exactly-once 而忽略實務上的重複投遞。

仍需補強的部分：

- 一開始可更明確描述 submission、job 與 outbox 的狀態機。
- 講到 idempotency key 時，要主動補充 payload fingerprint、unique constraint，以及相同 key／相同 payload 的回應。
- 講到 retry 時，要補充 exponential backoff、jitter、最大次數與 terminal state。
- 講到 dispatcher 時，要補充 claim、lease、發布後標記，以及 crash recovery。
- 面試回答最後應補上 observability，證明這套設計能在 production 被發現、操作與恢復。

### 表達上的建議

回答時可以先用一句話定義保證，再依資料流往下說：

> 我的目標是讓 submission 一旦被 API 接受就不會遺失，同一 logical request 不會重複建立，而 AI 工作即使重複投遞也能安全執行。

接著依序說明：

1. API 接收與 transaction。
2. `202 Accepted` 與前端狀態。
3. Idempotency 規則。
4. Worker timeout、retry 與狀態機。
5. Outbox 的適用時機與 at-least-once 保證。
6. Failure recovery 與 observability。

這樣比逐一列出 HTTP status 更能展現端到端 ownership。

---

## 改寫後的面試回答

> 我會把穩定接收 submission 與不穩定的 AI parsing 分開。前端送出時產生 `Idempotency-Key`；API 完成驗證後，在同一個 database transaction 中建立 `expense_submission` 與 `parsing_job`，然後回傳 `202 Accepted`、submission ID 和目前狀態。AI call 不放在 transaction 或同步 request 中，而是由 background worker 執行。
> 
> 前端 disable button 只是 UX 保護，真正的防重由後端完成。Idempotency record 會綁定使用者、operation 與 request payload fingerprint。相同 key 和相同 payload 重送時，後端回傳既有 submission；相同 key 對應不同 payload 時才回 `409 IDEMPOTENCY_CONFLICT`。
> 
> Client retry 與 worker retry 是不同機制。Client retry 沿用相同 key，目的是確認提交結果；worker retry 則由後端根據 job status、error category、attempt count 和 `next_retry_at` 決定。AI call 會有 timeout，只針對 timeout、429 與部分暫時性 5xx 使用 exponential backoff 加 jitter 做有限重試。超過限制後，job 進入 failed，submission 轉為 `needs_manual_input`，但保留使用者原始資料。
> 
> 後端是狀態的最終決策者。即使前端要求 retry，如果 job 正在執行、已完成、不可重試或已超過限制，後端也不會再次執行，而是回傳穩定的 application error code。API 不暴露 database 或 AI provider 的底層錯誤，response 會包含穩定 code、安全 message、request ID，以及適用時的 retry hint。
> 
> 如果 MVP 的 worker 直接 polling PostgreSQL `parsing_jobs`，我可以先以 database 作為可靠的 job source，不必立刻加入 outbox。如果需要可靠地把 job 發布到外部 queue，我會在建立 submission 與 job 的同一個 transaction 中寫入 `outbox_event`，再由 dispatcher 發布。Outbox 解決 database 與 queue 的 dual-write 缺口，但仍可能重複發布，所以系統採用 at-least-once delivery，consumer 以 event ID 或 job ID 做 idempotent processing。
> 
> 最後，我會監控 pending job age、retry rate、terminal failure rate、AI timeout、outbox backlog，以及長時間停在 processing 的 job，並用 request ID、submission ID、job ID 與 event ID 串聯 logs 和 traces，讓流程可以被觀察、重試與恢復。

---

## Mindset

> 把流程設計成可觀察、可重試、可恢復，而不是假設每一步都只會成功執行一次。

重要心法：

- 接受 client request、queue message 與 worker job 都可能重複。
- 不追求難以證明的 exactly-once，而是採用 at-least-once delivery 與 idempotent processing。
- 先定義狀態與允許的狀態轉換，再決定誰可以觸發 retry。
- 使用 transaction 保護同一個 database 內的一致性。
- 使用 outbox 解決 database 與外部 queue 的 dual-write 問題。
- 將業務資料、背景工作狀態與事件發布狀態分開管理。
- 系統的正確性不只包含成功路徑，也包含 crash 後如何恢復。

## 接下來三個練習重點

1. 練習用一分鐘清楚說出 submission、job 與 outbox 三種狀態的邊界。
2. 練習回答「某一步已成功、但下一步前 crash」的 partial-failure 題型。
3. 練習用具體 metrics、alerts 與 correlation IDs 證明系統可以被營運。