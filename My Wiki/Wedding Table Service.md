---
type: concept
status: growing
topics: [side-project, wedding, realtime, firebase]
created: "2026-07-15"
---

# Wedding Table Service

## Summary

Wedding Table Service 是 [[Side Projects]] 之一，目標是在很短的開發時間內提供婚禮報到、Google sign-in 與多裝置即時同步。

## Architecture

最初方案是 GitHub Pages + Google Sheets + Google Apps Script，再用 Polling 或 [[Webhook Architecture|Webhook]] 同步。評估整合與測試成本後，改採 [[Firebase Realtime App Architecture]]：Firebase Hosting 部署前端、Firestore 保存資料、Firebase Authentication 處理登入，Realtime listener 同步各裝置狀態。

這個選擇也讓資料設計從 table relationship 轉向 query pattern；相關取捨整理在 [[SQL vs NoSQL]]。若未來加入複雜報表、稽核或跨集合關聯，再評估額外 backend 或 SQL data store。

## Project Constraints

- 優先快速上線，避免自行維護多個 backend services。
- 現場可能有多台裝置同時更新報到狀態。
- Client 直接使用 Firebase SDK，因此 Firestore Security Rules 是必要的授權邊界。

## Related

- [[Side Projects]]
- [[Firebase Realtime App Architecture]]
- [[SQL vs NoSQL]]
- [[OAuth for Browser Apps]]

## References

- [[2026-06-07 - 6-7 weekly update]]
