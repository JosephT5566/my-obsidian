---
type: decision-record
status: accepted
decision-date: "2025-10-29"
topics: [side-project, expense, authentication, oauth, supabase, jwt]
---

# Expense App Authentication

## Context

[[Expense App]] 是 Browser-based application，登入、session、token refresh 與 Supabase data authorization 交由 [[Supabase]] Auth 處理。[[Google Cloud Functions]] 不負責登入或簽發 application session，但 [[Expense Receipt AI Pipeline|AI service endpoints]] 是受保護的 Resource Server：Client 呼叫時帶上 Supabase Access Token，GCF 再透過 [[Supabase JWKS]] 驗證身份。

原始 OAuth 學習筆記同時研究 Google Identity、Supabase session、OAuth proxy 與 backend token verification；這些是方案比較，不代表 Expense App 全部採用。

## Options Considered

### 直接使用 Google Identity 並自行管理 Session

- 這是原始筆記研究的通用方案，Expense App 沒有採用。
- `google.accounts.id` 可取得 Google ID Token，再由自有 backend 驗證並建立 application session。
- Backend 必須自行處理 session storage、cookie、logout、revocation 與 CSRF protection。

### 使用 GCF 驗證 Supabase JWT

- Client 以 `Authorization: Bearer <Supabase access token>` 呼叫 AI endpoint。
- GCF 使用 Supabase project 的 JWKS endpoint 取得 signing key，驗證 token signature、`ES256`、expiration 與 `audience="authenticated"`。
- 驗證成功後，`get_upload_url` 會從 `sub` 取得 user ID，放進 `uploads/{user_id}/...` 的 GCS object path；`analyze_receipt` 也會先驗證 JWT。
- GCF 仍只提供 AI workload，不是 login、session 或 token issuer；它在此扮演受保護 endpoint 的 authentication boundary。

### 使用 Supabase Auth 管理 Application Session

- [[Supabase]] 完成 Google social login 後，簽發自己的 Access Token（JWT）與 rotating Refresh Token。
- Supabase SDK 管理 Browser session 與 token refresh，並自動把 JWT 用於 Supabase requests。
- Supabase API 驗證 session，PostgreSQL Row Level Security（RLS）負責 data authorization。
- 這是 Expense App 實際採用的方案。

## Decision

Expense App 採用 [[Supabase]] Auth 作為 application authentication 與 session provider。Client 使用 Supabase SDK 登入、refresh session 並存取 Supabase data；當 Client 呼叫 [[Expense Receipt AI Pipeline|GCF AI endpoint]] 時，會重用 Supabase Access Token，由 GCF 透過 [[Supabase JWKS]] 驗證 request。也就是 Supabase 負責「登入與發行 token」，GCF 只負責「驗證 token 後執行 AI operation」。

```text
Google social login
→ Supabase Auth session
→ Supabase Access Token (JWT)
├→ Supabase API → Row Level Security authorization
└→ Authorization: Bearer <JWT>
   → GCF AI endpoint
   → Supabase JWKS verification
   → Signed URL or receipt analysis
```

## Consequences

- Login、session refresh 與 token issuance 集中在 Supabase；GCF 不維護另一套 session，但必須驗證每一筆受保護的 AI request。
- Browser session lifecycle 交由 Supabase SDK 管理；不能假設 Refresh Token 固定 30 天過期。Session lifetime 取決於 sign-out、password/security events，以及專案設定的 time-box、inactivity 或 single-session policy。
- Client 能直接存取 Supabase data，因此 Row Level Security 是必要的 authorization layer。
- `health` action 不驗證 JWT；`get_upload_url` 與 `analyze_receipt` 會先驗證 JWT，失敗時回傳 `401`。
- 目前程式註解把 JWT 初始化寫成 `RS256`，實際 `jwt.decode` 卻只接受 `ES256`；文件以執行邏輯為準，但程式註解應同步修正，並確認 Supabase 專案實際使用的 signing algorithm。
- [[TanStack Query Server State|Server-state cache]] 不負責 authentication；sign-out 或 session termination 時需要清除 user-scoped queries。

## Follow-up Actions

- [ ] 測試 page reload、multi-tab、token refresh、sign-out 與 expired session 的 UI 狀態。
- [ ] 檢查 Supabase RLS policies，避免只依賴前端隱藏功能。
- [ ] 對刪除帳號等敏感操作評估 Manual re-authentication。
- [ ] 驗證 `issuer`，並確認 key rotation、JWKS cache refresh 與找不到 `kid` 時的行為。
- [ ] `analyze_receipt` 除了驗證 JWT，也要確認每個 `file_path` 都屬於 `uploads/{sub}/`，避免已登入使用者讀取其他使用者的 Object。
- [ ] 加入 abuse prevention、rate limit 與 service-level monitoring。

## Related

- [[Expense App]]
- [[OAuth for Browser Apps]]
- [[Supabase]]
- [[Supabase JWKS]]
- [[Expense Receipt AI Pipeline]]

## References

- [[2025-10-29 - Oauth]]
- [[2026-05-10 - 5-10 weekly updates]]
- User clarification (2026-07-16)
- [Private implementation: google-ai-gcf/main.py](https://github.com/JosephT5566/google-ai-gcf/blob/main/main.py)
- [Google Identity Services: Choose an authorization model](https://developers.google.com/identity/oauth2/web/guides/choose-authorization-model)
- [Supabase Auth: User sessions](https://supabase.com/docs/guides/auth/sessions)
