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

管理支出、搜尋紀錄與分攤資料的 App。收據辨識功能的實作與取捨記錄在 [[Expense Receipt AI Pipeline]]，並串接 [[Google Cloud Functions]]、[[Google Cloud Storage]] 與 [[Supabase]]。

### [[Wedding Table Service]]

為婚禮現場快速建立的多裝置報到 App。它以 [[Firebase Realtime App Architecture]] 避免自行組合 Hosting、Auth、Polling 或 [[Webhook Architecture|Webhook]]。

## Organization

- 專案頁回答「為什麼做、有哪些功能、如何演進」。
- Architecture 或 Decision Record 回答「某個子系統為何採用這個方案」。
- Tool、Library 與 Concept 頁回答「技術如何運作，以及在不同情境如何重用」。
- 專案與技術頁都應在正文直接使用 Obsidian wikilink，而不只放在 `Related`。

## Related

- [[Expense App]]
- [[Wedding Table Service]]
- [[System Design Foundations]]
