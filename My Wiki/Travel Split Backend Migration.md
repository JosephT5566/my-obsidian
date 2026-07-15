---
type: decision-record
status: accepted
decision-date: "2026-05-10"
topics: [side-project, travel, cloudflare, workers-kv, oauth, gcf, gas, google-sheets, authentication, cookies]
---

# Travel Split Backend Migration

## Context

[[Travel Split App]] 原先讓每一筆 request 都經過 Cloudflare proxy。Proxy 保存 Google OAuth client secret，並以 Cloudflare Workers KV 暫存 Access Token 與 Refresh Token；Access Token 過期時，Cloudflare 會向 Google 取得新 token，再把 request 送往 Google Apps Script（GAS），由 GAS 驗證來源使用者並取得 Google Sheets data。

這個 full-proxy design 讓 authentication 與 token refresh 集中管理，但每個 application request 都多經過 Cloudflare，導致回應速度過慢。

## Options Considered

### 保留 Cloudflare Full Proxy + GAS

- Google OAuth secret、Access Token 與 Refresh Token 都集中由 Cloudflare 管理。
- Token 過期時可由 proxy 統一 refresh，再把 request 送往 GAS。
- 缺點是每一筆 data request 都必須經過 proxy，額外 network hop 與 token lookup 造成延遲。

### Cloudflare Token Issuer + GCF Service

- Cloudflare 不再代理每一筆 application request，改為 token issuer／renewal endpoint。
- Token 透過 HttpOnly／Secure cookie 交付，Browser JavaScript 不直接讀取 credential。
- [[Google Cloud Functions|GCF]] service 接收 cookie／token，並執行安全驗證，application traffic 不必再繞過 Cloudflare full proxy。

## Decision

將 Travel Split backend 從 GAS 遷移到 [[Google Cloud Functions]]，並把既有 Cloudflare full proxy 調整為 token issuer。Cloudflare 繼續負責 Google OAuth credential／token renewal，但不再轉送每一筆 application request；Browser 透過 HttpOnly／Secure cookie 帶上 token，由 GCF service 完成 request verification。Local development 也調整 origin 與 cookie policy，使 request 維持 SameSite boundary。

```text
Browser → Cloudflare token issuer → HttpOnly / Secure cookie
Browser + cookie → GCF service → verification → application operation
```

## Consequences

- Application traffic 不再每次經過 Cloudflare full proxy，移除主要 latency bottleneck。
- Google OAuth client secret 仍留在 Cloudflare；Client 不直接持有 secret，token 透過 HttpOnly／Secure cookie 交付。
- Cookie-based flow 需要一起處理 SameSite、Secure、CORS、CSRF 與 local development origin；相關通用原則整理在 [[OAuth for Browser Apps]]。
- 這套 proxy-issued cookie boundary 不等同 [[Expense App Authentication]] 的 Supabase-managed session；兩個 App 的 tokens 不應被視為可互換。
- [[Expense Receipt AI Pipeline]] 沒有沿用 Travel Split GCF，而且不使用相同 auth model；Expense App authentication 完全交給 Supabase Auth。

## Follow-up Actions

- [ ] 補記新模式下 Cloudflare token issuer 與 Workers KV 的完整責任邊界。
- [ ] 為 cookie-based requests 驗證 SameSite、Secure、CORS 與 CSRF behavior。
- [ ] 記錄 cookie payload、GCF verification contract、token expiration 與 revocation flow。
- [ ] 比較 migration 前後的 end-to-end latency 與 error rate。
- [ ] 確認 local、staging 與 production 的 callback／origin configuration 一致。

## Related

- [[Travel Split App]]
- [[Google Cloud Functions]]
- [[OAuth for Browser Apps]]
- [[Expense App Authentication]]
- [[Expense Receipt AI Pipeline]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- User clarification (2026-07-16)
