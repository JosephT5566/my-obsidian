---
type: tool-or-library
status: active
topics: [backend-as-a-service, postgresql, authentication, jwt]
created: "2026-07-15"
---

# Supabase

## Summary

Supabase 是以 PostgreSQL 為核心的 Backend as a Service，整合 database、authentication、storage 與 API。[[Expense App]] 使用 Supabase Auth 處理登入、session、token refresh 與 token issuance，並以 Supabase API／Row Level Security 保護 App data；呼叫 AI GCF 時，也會把同一個 Supabase Access Token 交給 endpoint 驗證。

## Use Cases

- 以 PostgreSQL 儲存並查詢 application data。
- 使用 Supabase Auth 封裝第三方登入、session 與 token refresh。
- 讓 Supabase API 驗證 session JWT，並透過 PostgreSQL Row Level Security 執行 authorization。

## Authentication

Supabase OAuth flow 會在 provider 登入後建立自己的 application session，包含 Access Token（JWT）與 rotating Refresh Token。Browser 端不同 token 的責任整理在 [[OAuth for Browser Apps]]；[[Expense App Authentication]] 則記錄 Expense App 選擇 Supabase Auth 的實際理由與 request path。

Access Token 通常是短效 JWT；Refresh Token 一般只能使用一次，成功 refresh 後會換成新的一組。Session 預設可持續存在，直到 sign-out、security event，或專案設定的 time-box、inactivity、single-session policy 終止它，因此不應把 session 寫成固定 30 天有效。

[[Expense App Authentication]] 同時採用 provider-managed session 與 external backend verification：[[Expense Receipt AI Pipeline]] 的 GCF 不建立另一套 session，而是透過 [[Supabase JWKS]] 驗證 Client 傳來的 Supabase JWT。Supabase API 與 GCF 是兩個不同的 Resource Server；兩者各自在自己的 boundary 驗證 token 並執行 authorization。

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
- [[Expense App Authentication]]
- [[Supabase JWKS]]
- [[OAuth for Browser Apps]]
- [[SQL vs NoSQL]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2025-10-29 - Oauth]]
- [Supabase Auth: User sessions](https://supabase.com/docs/guides/auth/sessions)
- [Private Expense AI implementation: google-ai-gcf/main.py](https://github.com/JosephT5566/google-ai-gcf/blob/main/main.py)
