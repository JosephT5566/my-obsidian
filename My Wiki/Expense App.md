---
type: concept
status: growing
topics: [side-project, expense, product-architecture, frontend]
created: "2026-07-15"
---

# Expense App

## Summary

Expense App 是 [[Side Projects]] 之一，用來新增、搜尋與整理支出資料。除了基本的 Expense Row 操作，也加入收據圖片辨識，將 AI 結果轉成可由使用者確認的結構化資料。

## Product Areas

- **Expense management**：共用既有 Expense Row component，依日期範圍與文字搜尋資料；文字查詢使用 PostgreSQL `ILIKE`。
- **UI system**：使用 Shadcn shared components；大型表單採 Dialog，而不是空間較受限的 Drawer。
- **Authentication**：由 [[Supabase]] Auth 處理登入、session、token refresh 與 token issuance；[[Expense App Authentication]] 另記錄 GCF AI endpoint 如何用 [[Supabase JWKS]] 驗證 caller。
- **Data authorization**：Supabase API 透過 [[PostgreSQL Row-Level Security]] 保護 user／organization scoped rows。
- **Receipt AI**：以 Upload、Analysis 與 Review／Confirm 處理收據；pipeline 見 [[Expense Receipt AI Pipeline]]，AI candidate 到正式 expense 的設計見 [[Expense Draft Review Workflow]]。

## Architecture Map

```text
Expense App
├── Authentication
│   └── Supabase Auth → Supabase API → PostgreSQL RLS
└── Receipt AI
    ├── Supabase Access Token → Supabase JWKS verification
    ├── Google Cloud Functions：protected AI endpoint
    ├── Google Cloud Storage：暫存待分析的圖片
    ├── AI output validation → editable draft
    └── Confirm transaction → canonical expense
```

登入與 session boundary 整理在 [[Expense App Authentication]]，由 Supabase Auth 管理。[[Google Cloud Functions]] 只屬於 Receipt AI flow，不簽發 session；Client 呼叫 `get_upload_url` 或 `analyze_receipt` 時帶上 Supabase JWT，GCF 透過 [[Supabase JWKS]] 驗證後，才建立 [[Google Cloud Storage]] Signed URL 或執行收據分析。

## Evolution

1. 以 [[Expense App Authentication]] 確立 Supabase-managed login、session 與 RLS authorization，並讓 GCF AI endpoint 以 Supabase JWKS 驗證 request，而不是另外建立 login service。
2. 以 Shadcn components 統一介面並改善表單體驗。
3. 完成日期範圍與關鍵字搜尋頁。
4. 規劃並實作 [[Expense Receipt AI Pipeline]]。
5. 在 Summary 階段加入 Group、Ungroup、Ignore、Category 與 Split mode，最後轉成 `ExpenseRow[]`。
6. 以 [[Expense Draft Review Workflow]] 推演 raw extraction、editable draft、optimistic concurrency 與 idempotent confirm；此項目前是設計提案，不代表已部署。

## Related

- [[Side Projects]]
- [[Expense App Authentication]]
- [[Expense Receipt AI Pipeline]]
- [[Expense Draft Review Workflow]]
- [[PostgreSQL Row-Level Security]]
- [[Travel Split App]]
- [[Supabase]]
- [[OAuth for Browser Apps]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2026-05-17 - 5-17 weekly update]]
- [[2026-07-26 weekly updates]]
