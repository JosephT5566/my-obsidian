---
type: tool-or-library
status: active
topics: [gcp, serverless, backend, authentication]
created: "2026-07-15"
---

# Google Cloud Functions

## Summary

Google Cloud Functions（GCF）是 event-driven serverless compute。[[Travel Split App]] 透過 [[Travel Split Backend Migration]] 把 backend 從 GAS 移到 GCF，並讓 GCF service 驗證 token issuer 所建立的 cookie／token。[[Expense App]] 的另一個 GCF 則提供 AI-related endpoints，並透過 [[Supabase JWKS]] 驗證 Supabase Access Token；它驗證 request，但不負責登入、session refresh 或 token issuance。

## Use Cases

- 建立需要 Cloud credential 的短效 Signed URL。
- 將第三方 API key 留在 server-side environment，不暴露給 Browser Client。
- 為低流量或事件驅動工作提供可獨立部署的 endpoint。

## Configuration

在 [[Expense Receipt AI Pipeline]] 中，Client 以 Bearer token 呼叫 GCF。GCF 從 Supabase JWKS endpoint 取得 signing key，以 `ES256` 與 `audience="authenticated"` 驗證 JWT，再以 Service Account 建立 15 分鐘的 GCS Signed URL，或用 server-side prompt 與 AI API credential 分析已上傳圖片。[[Expense App Authentication]] 說明 Supabase 負責發行 token、GCF 負責驗證 AI requests 的責任分界。

[[Travel Split App]] 的 GCF service 接收 Cloudflare token issuer 建立的 cookie／token，並進行安全驗證；[[Expense Receipt AI Pipeline|Expense AI]] 使用另一個 Function，驗證 Supabase JWT 後承接 AI workload。兩者的 token issuer、verification method、用途、硬體配置與帳戶資源不同，因此分開部署。

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
- [[Expense App Authentication]]
- [[Travel Split App]]
- [[Travel Split Backend Migration]]
- [[Google Cloud Storage]]
- [[OAuth for Browser Apps]]
- [[Supabase JWKS]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [Private Expense AI implementation: google-ai-gcf/main.py](https://github.com/JosephT5566/google-ai-gcf/blob/main/main.py)
