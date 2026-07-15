---
type: concept
status: growing
topics: [oauth, oidc, authentication, authorization, browser-security, tokens]
created: "2026-07-14"
---

# OAuth for Browser Apps

## Summary

OAuth 2.0 是 authorization framework，不是登入協定；Browser app 的登入通常還會使用 OpenID Connect 或 provider 的 identity layer。設計時要先區分 authentication、application session 與 delegated API authorization，再決定由 Browser、Backend 或 Auth provider 管理 tokens。

## Why It Matters

同一個「使用 Google 登入」需求可能包含三個不同問題：

1. 使用者是誰？由 ID Token 或 Auth provider session 回答。
2. 使用者可不可以使用 App data／API？由 application token、session 與 authorization rules 回答。
3. App 可不可以代表使用者呼叫 Google API？由帶有特定 scopes 的 Google Access Token 回答。

把這三者混在一起，常會把 token 傳給錯誤的 Resource Server，或讓 Browser 承擔不該持有的長期 credential。

## How It Works

- Resource Owner：使用者。
- Client：應用程式。
- Authorization Server：核發 token。
- Resource Server：驗證 Access Token 並提供 API。

### Token Responsibilities

| Token | 回答的問題 | 主要接收者 | Browser 注意事項 |
|---|---|---|---|
| ID Token | 使用者是誰 | 自己的 login／backend endpoint | 驗證 signature、issuer、audience、expiration；不能拿來呼叫 Google resource API |
| Access Token | Client 可執行哪些 actions | 發行它的 Resource Server | 短效、依 scopes 限權；外洩者可在有效期內使用 |
| Refresh Token | 如何取得下一組 Access Token | Authorization Server／Auth SDK | 長期且高敏感；純 Browser code 無法真正隱藏它 |

JWT 只是一種 token format。只 decode payload 不等於驗證；接收端仍要驗證 signature、claims 與 application-level permissions。[[Supabase JWKS]] 是 asymmetric JWT verification 的實際例子。

## Google Identity in Browser Apps

### Authentication: Sign in with Google

`google.accounts.id` 回傳 ID Token，用於 authentication。自己的 backend 可以驗證 ID Token，再建立 application session；ID Token 到期不代表 application session 必須同時結束。若使用 Google HTML POST integration，還需要驗證 CSRF token。

### Authorization: Google APIs

`google.accounts.oauth2` 用於取得呼叫 Google APIs 的授權：

- **Token Model**：Browser 在使用者操作時取得短效 Access Token，適合直接呼叫 Google APIs；通常不提供 offline access。
- **Code Model**：Browser 取得 Authorization Code，backend 驗證 request 後交換 Access Token 與 Refresh Token；適合 server-side storage 與 offline access。

Google ID Token、Google Access Token 與 application provider token 是不同 credentials。例如 [[Expense App Authentication]] 由 [[Supabase]] Auth 管理 session 並簽發 Access Token；[[Expense Receipt AI Pipeline|GCF AI endpoint]] 是另一個 Resource Server，透過 [[Supabase JWKS]] 驗證該 token。只有需要 Google-owned resources 時，才另外要求 Google scopes。

## Choosing a Session Model

### Auth Provider-managed Session

[[Supabase]] 等 Auth provider 可以封裝 social login、application session 與 refresh rotation。自有 backend 仍可驗證 provider-issued token，而不必自己變成 token issuer；Expense App 的 GCF + JWKS 就是這種組合。Browser SDK 仍需面對 XSS 與 storage 風險，因此要搭配 CSP、dependency hygiene、Row Level Security 與短效 Access Token。Session lifetime 應以 provider／project configuration 為準，不能假設固定天數。

### Backend or OAuth Proxy

需要 offline access、集中呼叫第三方 APIs，或不希望 Browser JavaScript 接觸 Refresh Token 時，可以讓 backend／BFF 完成 Code Flow 並保存長期 credential。[[Travel Split Backend Migration]] 是實際案例：舊架構由 Cloudflare full proxy 保存 Google OAuth secret，並以 Workers KV 管理 Access／Refresh Tokens；因每一筆 request 都經過 proxy 而過慢，後來改成 Cloudflare token issuer + HttpOnly／Secure cookie，由 GCF service 驗證 application request。

### Browser-only Authorization

若只是由 Browser 在使用者操作時短暫呼叫第三方 API，可採 provider 的 Token Model 或支援 Authorization Code + PKCE 的 public-client flow。Access Token 到期不應直接視為 App 已登出；可以先重新授權，失敗後再要求使用者互動。

## Common Mistakes

- 把 OAuth 當成 authentication protocol，而沒有定義 application session。
- 用 ID Token 呼叫 Google Drive／Calendar 等 APIs。
- 把 Google Access Token 當成 Supabase 或自有 API 的 session token。
- 在 Browser bundle 放 Client Secret，或自行長期保存 Refresh Token。
- 只 decode JWT，不驗證 signature、issuer、audience 與 expiration。
- 讓 [[TanStack Query Server State|query cache]] 決定登入狀態，或在 sign-out 後保留 user-scoped cache。

## Tradeoffs

- Provider-managed session 開發快，但要接受 SDK storage model、provider configuration 與 service dependency。
- Backend-managed token 提供較完整的 credential isolation、revocation 與 auditing，代價是增加 server、cookie、CSRF 與 session operations。
- Browser-held Access Token 架構簡單，但 XSS 會直接暴露有效 token，因此 scopes 與 lifetime 必須最小化。

## Related

- [[Expense App Authentication]]
- [[Expense App]]
- [[Travel Split Backend Migration]]
- [[Supabase]]
- [[Supabase JWKS]]
- [[TanStack Query Server State]]

## References

- [[2025-10-29 - Oauth]]
- [Google Identity Services: Sign in with Google](https://developers.google.com/identity/gsi/web/guides/overview)
- [Google Identity Services: Choose an authorization model](https://developers.google.com/identity/oauth2/web/guides/choose-authorization-model)
- [Supabase Auth: User sessions](https://supabase.com/docs/guides/auth/sessions)
