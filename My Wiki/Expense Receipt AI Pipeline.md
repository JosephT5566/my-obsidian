---
type: decision-record
status: accepted
decision-date: "2026-05-17"
topics: [side-project, expense, ai, gcf, gcs, jwt, frontend]
---

# Expense Receipt AI Pipeline

## Context

[[Expense App]] 需要讓使用者上傳收據圖片，交由 AI 分析，再由使用者確認、分組與補齊欄位，最後保存成 `ExpenseRow[]`。圖片不應直接穿過既有 App backend，也不能把長期 Cloud credential 暴露給 Client。

## Options Considered

### 由既有 Backend 接收圖片

- 架構直接，但增加 Backend 流量、記憶體與用途耦合。
- 圖片需要經過 App server 再轉送到 Storage 或 AI。

### Client 使用 Signed URL 直接上傳 [[Google Cloud Storage|GCS]]

- [[Google Cloud Functions|GCF]] 先以 [[Supabase JWKS]] 驗證 [[Supabase]] JWT，再發行短效 Signed URL。
- Client 直接 `PUT` 圖片到 [[Google Cloud Storage|GCS]]。
- Client 將上傳後的 Object URL 交給 AI analysis GCF。

## Decision

採用 [[Google Cloud Functions|GCF]] + [[Google Cloud Storage|GCS]] + AI API 的分離 Pipeline，並讓 Travel Split 與 Expense AI 使用不同 GCF，以分離用途、設定與帳戶資源。

## Consequences

流程分成：

1. **Upload**：App 向 [[Google Cloud Functions|GCF]] 取得約 10 分鐘有效的 Signed URL，再直接上傳 [[Google Cloud Storage|GCS]]。
2. **Analysis**：App 呼叫 AI GCF 分析圖片並顯示結構化結果，可選擇重新執行。
3. **Summary**：使用者 Group、Ungroup、Ignore 項目，補齊 Category 與 Split mode，再轉成 `ExpenseRow[]` 寫入資料庫。

驗證使用 [[Supabase JWKS]] 的 asymmetric signature；GCF 可從 [[Supabase]] 的公開端點取得 verification key，不需持有 Legacy symmetric JWT secret。Service Account 只授予建立 Signed URL 與讀取必要 Object 的最低權限。

## Follow-up Actions

- [ ] 限制 Signed URL 的 Object path、Content type、Size 與有效時間。
- [ ] 為暫存圖片設定 Lifecycle deletion。
- [ ] 驗證 AI 回應 Schema，不直接信任 Model output。
- [ ] 記錄 Analysis request ID，避免重試造成重複寫入。
- [ ] 檢查 GCF、GCS 與 AI API 的成本與錯誤率。

## Related

- [[AI Agent Architecture]]
- [[Expense App]]
- [[Google Cloud Functions]]
- [[Google Cloud Storage]]
- [[Supabase]]
- [[Supabase JWKS]]
- [[System Design Foundations]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2026-05-17 - 5-17 weekly update]]
