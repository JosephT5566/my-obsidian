---
type: decision-record
status: accepted
decision-date: "2026-06-07"
topics: [side-project, wedding, firebase, firestore, realtime, architecture]
---

# Firebase Realtime App Architecture

## Context

[[Wedding Table Service]] 需要快速上線、支援多台裝置、Google sign-in 與即時同步出席狀態。原始構想是 GitHub Pages、Google Sheets 與 Google Apps Script，再以 Polling 或 [[Webhook Architecture|Webhook]] 同步。

## Options Considered

### Google Sheets + Google Apps Script

- 能沿用熟悉的試算表與簡單 API。
- 多裝置同步需要額外設計 Polling 或 [[Webhook Architecture|Webhook]]。
- Auth、Hosting 與同步機制分散在不同服務。

### Firebase Hosting + Firestore + Firebase Authentication

- Firebase Hosting 部署前端。
- Firestore 保存 NoSQL document，Realtime listener 同步多裝置。
- Firebase Authentication 提供 Google sign-in。
- Security Rules 在資料路徑與操作層級授權。

## Decision

使用 Firebase 作為 Backend as a Service，以 Firestore、Firebase Hosting 與 Firebase Authentication 組成最小可行架構。

## Consequences

- 省去自行建立 Webhook、Polling 與 Auth backend 的時間。
- Client 可直接使用 Firebase SDK，但所有資料存取都必須由 Security Rules 保護。
- 資料模型需要依畫面與 Query pattern 設計；這正是 [[SQL vs NoSQL]] 中以 Access pattern 為中心的 NoSQL 思維，並需接受 Denormalization 與查詢限制。
- 若未來需要複雜報表、稽核或跨集合關聯，可能需要額外後端或遷移評估。

Security Rules 的基本流程：

1. Request 對應到 `match` path。
2. 判斷操作是 `read`、`create`、`update` 或 `delete`。
3. 執行相對應的 `allow ... if` 條件。
4. 條件為 `true` 才允許操作。

```javascript
match /weddings/{weddingId}/guests/{guestId} {
  allow read: if canRead(weddingId);
  allow create, update: if canEdit(weddingId);
}
```

## Follow-up Actions

- [ ] 使用 Firebase Emulator 測試允許與拒絕案例。
- [ ] 確認規則不只檢查登入狀態，也驗證使用者與 `weddingId` 的關係。
- [ ] 監控 Firestore Read/Write 次數與成本。
- [ ] 規劃資料匯出與備份方式。

## Related

- [[SQL vs NoSQL]]
- [[System Design Foundations]]
- [[Wedding Table Service]]
- [[Side Projects]]
- [[Webhook Architecture]]

## References

- [[2026-06-07 - 6-7 weekly update]]
