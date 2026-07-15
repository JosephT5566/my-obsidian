---
type: tool-or-library
status: active
topics: [backend-as-a-service, postgresql, authentication, jwt]
created: "2026-07-15"
---

# Supabase

## Summary

Supabase 是以 PostgreSQL 為核心的 Backend as a Service，整合 database、authentication、storage 與 API。[[Expense App]] 使用它保存 App data，並用 Supabase 簽發的 JWT 作為 Client 呼叫 [[Google Cloud Functions]] 時的身分憑證。

## Use Cases

- 以 PostgreSQL 儲存並查詢 application data。
- 使用 Supabase Auth 封裝第三方登入、session 與 token refresh。
- 讓自有 backend 驗證 Supabase access token，再執行受保護的工作。

## Authentication

Supabase OAuth flow 會在 provider 登入後簽發自己的 Access Token（JWT）與 Refresh Token。Browser 端的 session lifecycle 與 ID Token、Access Token 的差異整理在 [[OAuth for Browser Apps]]。

外部 backend 不應只 decode JWT，而要驗證 signature、issuer、audience 與 expiration。[[Expense Receipt AI Pipeline]] 採用 [[Supabase JWKS]] 公開金鑰驗證新式 asymmetric JWT，避免把 Legacy symmetric JWT secret 複製到 GCF。

## Strengths

- 將 PostgreSQL、Auth 與自動產生的 API 整合在同一個服務。
- Frontend SDK 能處理 session 與 token refresh。
- PostgreSQL 適合 Expense App 後續的搜尋、篩選、統計與關聯需求。

## Limitations

- Client 能直接接觸 data API 時，Row Level Security policy 必須正確且持續測試。
- Supabase session token 不等於 Google API Access Token，不能直接拿來呼叫 Google Drive 等 resource API。
- JWT signing mode 或 key rotation 改變時，所有驗證端都必須正確處理。

## Related

- [[Expense App]]
- [[Supabase JWKS]]
- [[OAuth for Browser Apps]]
- [[SQL vs NoSQL]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2025-10-29 - Oauth]]
