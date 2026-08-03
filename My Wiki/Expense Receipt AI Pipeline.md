---
type: decision-record
status: accepted
decision-date: "2026-05-17"
topics: [side-project, expense, ai, gcf, gcs, frontend, jwt]
---

# Expense Receipt AI Pipeline

## Context

[[Expense App]] 需要讓使用者上傳收據圖片，交由 AI 分析，再由使用者確認、分組與補齊欄位，最後保存成 `ExpenseRow[]`。圖片不應直接穿過既有 App backend，也不能把長期 Cloud credential 暴露給 Client；[[AI Structured Output Validation|AI structured output]] 只能成為 candidate，不能未經 Server validation 與 review 就直接寫入正式 expense。

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
2. **Analysis**：App 以同一種 Bearer token 呼叫 `analyze_receipt`；GCF 驗證 JWT 後從 GCS 讀取圖片，呼叫 AI provider，並依 [[AI Structured Output Validation]] 將回應限制為候選資料。
3. **Review／Confirm**：使用者 Group、Ungroup、Ignore 項目，補齊 Category 與 Split mode；[[Expense Draft Review Workflow]] 再以 editable draft、version 與 idempotent confirm 管理正式寫入。這是 2026-07-26 的設計提案，不代表目前 flow 已完成 draft persistence。

[[Supabase]] Auth 與 Row Level Security 處理 Expense App 的登入、session 與 database authorization；GCF 只處理 AI-related operations，但它也是這些 endpoints 的 JWT verification boundary。GCF Service Account 只授予建立 Signed URL 與讀取必要 Object 的最低 Cloud permissions；Cloud IAM 與 end-user authentication 是兩個不同責任。

截至 2026-08-02，google-ai-gcf PR #18–#22 已補上 canonical receipt schema、Gemini structured response、versioned normalized result contract、GCS ownership／image-input hardening，以及 prompt-injection control-plane protection。這些 correctness 與 security controls 是 [[Backend Service Production Readiness|production readiness]] 的 release-gate 基礎；rate／concurrency limits、metrics／alerts、retention policy、rollback 與 runbook 仍是後續 operational work。

## Follow-up Actions

- [ ] 限制 Signed URL 的 Object path、Content type、Size 與有效時間。
- [x] 在 Analysis 階段驗證 `file_path` 必須位於目前 JWT `sub` 對應的 prefix，並檢查 MIME、size、generation 與 magic bytes。
- [ ] 為暫存圖片設定 Lifecycle deletion。
- [x] 以 canonical schema 驗證 AI response，分離 raw／normalized result，並讓 invalid output 在 normalization 或 persistence 前停止。
- [ ] 依 [[AI Structured Output Validation]] 補齊 draft 的 business rules、owner 與 source-file validation，不直接信任 Model output。
- [ ] 評估實作 [[Expense Draft Review Workflow]]，保存 raw extraction／draft value，並以 version 與 idempotency 保護 confirm。
- [ ] 記錄 Analysis request ID，避免重試造成重複寫入。
- [ ] 檢查 GCF、GCS 與 AI API 的成本與錯誤率。

## Related

- [[AI Agent Architecture]]
- [[AI Structured Output Validation]]
- [[Expense App]]
- [[Expense Draft Review Workflow]]
- [[Expense App Authentication]]
- [[Google Cloud Functions]]
- [[Google Cloud Storage]]
- [[Supabase]]
- [[Supabase JWKS]]
- [[Travel Split App]]
- [[System Design Foundations]]
- [[Reliable Background Job Processing]]
- [[Backend Service Production Readiness]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2026-05-17 - 5-17 weekly update]]
- [[2026-07-26 weekly updates]]
- [[2026-08-02 weekly updates]]
- [Private implementation: google-ai-gcf/main.py](https://github.com/JosephT5566/google-ai-gcf/blob/main/main.py)
