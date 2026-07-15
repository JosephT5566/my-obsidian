---
type: tool-or-library
status: active
topics: [gcp, serverless, backend, authentication]
created: "2026-07-15"
---

# Google Cloud Functions

## Summary

Google Cloud Functions（GCF）是 event-driven serverless compute。[[Expense App]] 用它承接短小、用途明確的 backend 工作，例如驗證 [[Supabase]] JWT、簽發 [[Google Cloud Storage]] Signed URL，以及呼叫 AI API。

## Use Cases

- 建立需要 Cloud credential 的短效 Signed URL。
- 將第三方 API key 留在 server-side environment，不暴露給 Browser Client。
- 為低流量或事件驅動工作提供可獨立部署的 endpoint。

## Configuration

在 [[Expense Receipt AI Pipeline]] 中，Upload Function 先透過 [[Supabase JWKS]] 驗證 JWT，再以 Service Account 建立 GCS Signed URL。Analysis Function 接收已上傳圖片的 URL，搭配 server-side prompt 與 AI API credential 取得結構化結果。

Travel Split 與 Expense AI 使用不同 Function，因為用途、硬體配置與帳戶資源不同。這種分離能縮小權限與部署影響範圍，但也增加設定、監控與版本管理的數量。

## Strengths

- 不需長期維護 App server，能依請求量調整執行資源。
- 每個 Function 可採最小權限並獨立部署。
- 適合把 Cloud credential 與 Browser Client 隔離。

## Limitations

- Cold start、執行時間與 request size 會影響適用情境。
- 分散的 Functions 需要一致的 logging、error handling、CORS 與 deployment 管理。
- Service Account IAM 若設定過寬，會放大 credential 或程式漏洞的影響。

## Related

- [[Expense Receipt AI Pipeline]]
- [[Google Cloud Storage]]
- [[Supabase JWKS]]
- [[OAuth for Browser Apps]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
