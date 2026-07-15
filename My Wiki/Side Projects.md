---
type: concept
status: growing
topics: [side-projects, project-navigation, architecture]
created: "2026-07-15"
---

# Side Projects

## Summary

這是個人 Side Projects 的導覽頁。每個專案頁負責保留目標、功能與演進脈絡；可重用的技術與架構決策則拆成獨立頁面，讓同一項技術能被不同專案交叉引用。

## Projects

### [[Expense App]]

管理支出、搜尋紀錄與分攤資料的 App。[[Supabase]] Auth 管理登入、session 與 token issuance，RLS 保護 database；[[Google Cloud Functions]] 提供 [[Expense Receipt AI Pipeline|Receipt AI endpoints]]，並透過 [[Supabase JWKS]] 驗證 caller。責任分界記錄在 [[Expense App Authentication]]。

### [[Travel Split App]]

旅行費用分帳 App。[[Travel Split Backend Migration]] 記錄從 Cloudflare full proxy + Workers KV + GAS，遷移到 Cloudflare token issuer + HttpOnly／Secure cookie + [[Google Cloud Functions|GCF]] verification 的過程，主要目的是移除每一筆 request 都繞過 proxy 的延遲。

### [[Wedding Table Service]]

為婚禮現場快速建立的多裝置報到 App。它以 [[Firebase Realtime App Architecture]] 避免自行組合 Hosting、Auth、Polling 或 [[Webhook Architecture|Webhook]]。

## Organization

- 專案頁回答「為什麼做、有哪些功能、如何演進」。
- Architecture 或 Decision Record 回答「某個子系統為何採用這個方案」。
- Tool、Library 與 Concept 頁回答「技術如何運作，以及在不同情境如何重用」。
- 專案與技術頁都應在正文直接使用 Obsidian wikilink，而不只放在 `Related`。

## Related

- [[Expense App]]
- [[Travel Split App]]
- [[Wedding Table Service]]
- [[System Design Foundations]]
