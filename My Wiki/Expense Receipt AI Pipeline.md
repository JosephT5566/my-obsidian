---
type: decision-record
status: accepted
decision-date: "2026-05-17"
topics: [side-project, expense, ai, gcf, gcs, frontend, jwt]
---

# Expense Receipt AI Pipeline

## Context

[[Expense App]] 需要讓使用者上傳收據圖片，交由 AI 分析，再由使用者確認、分組與補齊欄位，最後保存成 `ExpenseRow[]`。圖片不應直接穿過既有 App backend，也不能把長期 Cloud credential 暴露給 Client。

## Options Considered

### 由既有 Backend 接收圖片

- 架構直接，但增加 Backend 流量、記憶體與用途耦合。
- 圖片需要經過 App server 再轉送到 Storage 或 AI。

### Client 使用 Signed URL 直接上傳 [[Google Cloud Storage|GCS]]

- AI-related [[Google Cloud Functions|GCF]] endpoint 發行短效 Signed URL；它不負責登入或簽發 session，但會先依 [[Expense App Authentication]] 使用 [[Supabase JWKS]] 驗證 Supabase JWT。
- Client 直接 `PUT` 圖片到 [[Google Cloud Storage|GCS]]。
- Client 將上傳後的 GCS object path 交給 AI analysis GCF。

## Decision

採用 [[Google Cloud Functions|GCF]] + [[Google Cloud Storage|GCS]] + AI API 的分離 Pipeline，並讓 [[Travel Split App]] 與 Expense AI 使用不同 GCF，以分離用途、設定與帳戶資源。Travel Split 原有 Function 的背景見 [[Travel Split Backend Migration]]。

## Consequences

流程分成：

1. **Upload**：App 以 Supabase JWT 呼叫 GCF 的 `get_upload_url`；驗證成功後，GCF 使用 token `sub` 建立 `uploads/{user_id}/...` path，回傳 15 分鐘有效的 Signed URL，App 再直接上傳 [[Google Cloud Storage|GCS]]。
2. **Analysis**：App 以同一種 Bearer token 呼叫 `analyze_receipt`；GCF 驗證 JWT 後從 GCS 讀取圖片，呼叫 Gemini，並回傳結構化結果。
3. **Summary**：使用者 Group、Ungroup、Ignore 項目，補齊 Category 與 Split mode，再轉成 `ExpenseRow[]` 寫入資料庫。

[[Supabase]] Auth 與 Row Level Security 處理 Expense App 的登入、session 與 database authorization；GCF 只處理 AI-related operations，但它也是這些 endpoints 的 JWT verification boundary。GCF Service Account 只授予建立 Signed URL 與讀取必要 Object 的最低 Cloud permissions；Cloud IAM 與 end-user authentication 是兩個不同責任。

## Follow-up Actions

- [ ] 限制 Signed URL 的 Object path、Content type、Size 與有效時間。
- [ ] 在 Analysis 階段驗證 `file_path` 必須位於目前 JWT `sub` 對應的 prefix。
- [ ] 為暫存圖片設定 Lifecycle deletion。
- [ ] 驗證 AI 回應 Schema，不直接信任 Model output。
- [ ] 記錄 Analysis request ID，避免重試造成重複寫入。
- [ ] 檢查 GCF、GCS 與 AI API 的成本與錯誤率。

## Related

- [[AI Agent Architecture]]
- [[Expense App]]
- [[Expense App Authentication]]
- [[Google Cloud Functions]]
- [[Google Cloud Storage]]
- [[Supabase]]
- [[Supabase JWKS]]
- [[Travel Split App]]
- [[System Design Foundations]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2026-05-17 - 5-17 weekly update]]
- [Private implementation: google-ai-gcf/main.py](https://github.com/JosephT5566/google-ai-gcf/blob/main/main.py)
