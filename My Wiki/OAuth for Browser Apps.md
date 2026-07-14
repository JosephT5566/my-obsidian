---
type: concept
status: growing
topics: [oauth, authentication, authorization, browser-security]
created: "2026-07-14"
---

# OAuth for Browser Apps

## Summary

OAuth 是 authorization framework，不等同登入協定。Browser app 要區分 ID Token（證明使用者身分）與 Access Token（呼叫 resource API 的權限），並選擇符合前端安全限制的 flow。

## How It Works

- Resource Owner：使用者。
- Client：應用程式。
- Authorization Server：核發 token。
- Resource Server：驗證 Access Token 並提供 API。

Google Identity 的 `google.accounts.id` 用於 authentication，回傳 ID Token；`google.accounts.oauth2` 用於 authorization，依 scopes 回傳 Access Token。自己的 backend 可驗證 ID Token 建立 session；Google Drive/Calendar 等 API 則需要 Access Token。

純 browser client 優先採 Authorization Code + PKCE 或 provider SDK 的安全模式。不要把長期 Refresh Token 暴露在 JavaScript/localStorage。若需要長期 session，可用 backend/OAuth proxy 將 refresh token 放在 server-side storage 或 HttpOnly cookie，browser 只持有短期 session。

## Tradeoffs

- Access Token 短效降低外洩影響，但需要 renewal path。
- Silent re-auth 依賴既有 provider session，失敗後仍需互動式 consent。
- ID Token 不能拿來替代第三方 API 所需的 Access Token。

## Related

- [[User Impersonation Safety]]
- [[TanStack Query Server State]]
- [[Expense Receipt AI Pipeline]]

## References

- `Row notes/2025-10-29 - Oauth.md`
