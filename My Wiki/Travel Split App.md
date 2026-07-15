---
type: concept
status: seed
topics: [side-project, travel, expense-splitting, cloudflare, oauth, google-sheets, gcf]
created: "2026-07-16"
---

# Travel Split App

## Summary

Travel Split App 是 [[Side Projects]] 之一，用於旅行費用分帳，並從 Google Sheets 取得資料。現有紀錄主要描述 authentication proxy 的效能問題，以及 backend 從 Google Apps Script（GAS）遷移到 GCF 的過程。

## Architecture Map

### Before: Full Proxy

```text
Browser request
→ Cloudflare proxy
   ├─ Google OAuth client secret
   ├─ Workers KV: access token + refresh token
   └─ Access Token expired → request a new token from Google
→ Google Apps Script verifies source user
→ Google Sheets data
```

每一筆 application request 都先經過 Cloudflare proxy。Proxy 保存 Google OAuth secret，並以 Workers KV 暫存 Access Token 與 Refresh Token；Access Token 過期時，由 Cloudflare 向 Google 更新後再把 request 送往 GAS。這條路徑雖集中管理 credentials，但多一層 network hop 與 token handling，造成明顯延遲。

### After: Token Issuer

```text
Browser → Cloudflare token issuer → HttpOnly / Secure cookie
Browser + cookie → Google Cloud Functions service → request verification
```

核心架構變更記錄在 [[Travel Split Backend Migration]]：Cloudflare 從每次 request 都經過的 full proxy，改為只負責 token issuing／renewal；Browser 以 cookie 帶上 credential，[[Google Cloud Functions|GCF]] service 再驗證 request。Access Token 過期時，Client 交由 Cloudflare token issuer 完成 renewal，而不是讓所有 application data traffic 都穿過 proxy。

## Relationship to Expense App

Travel Split 與 [[Expense App]] 都使用 GCF，但驗證來源與業務責任不同：Travel Split 的 GCF service 會配合 Cloudflare-issued cookie 進行安全驗證；Expense App 的 GCF 提供 [[Expense Receipt AI Pipeline|AI-related endpoints]]，並透過 [[Supabase JWKS]] 驗證 Supabase-issued Access Token。兩個專案沒有共用 Function。

## Known Gaps

- 原始資料未說明費用輸入、分帳規則、成員模型與結算流程。
- 原始資料未保留 GAS 與 GCF 的完整成本或效能比較。
- 新 token issuer mode 的 cookie payload、GCF validation contract 與 CSRF handling 尚未完整記錄。

## Related

- [[Side Projects]]
- [[Travel Split Backend Migration]]
- [[Google Cloud Functions]]
- [[OAuth for Browser Apps]]
- [[Expense App]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- User clarification (2026-07-16)
