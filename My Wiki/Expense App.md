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
- **Receipt AI**：以三步驟 Wizard 完成 Upload、Analysis 與 Summary，詳細架構見 [[Expense Receipt AI Pipeline]]。

## Architecture Map

```text
Expense App
└── Receipt AI
    ├── Google Cloud Functions：驗證請求、發行 Signed URL、呼叫 AI API
    ├── Google Cloud Storage：暫存待分析的圖片
    └── Supabase：提供 App data 與 JWT-based authentication
        └── Supabase JWKS：讓 GCF 取得公開金鑰並驗證 JWT
```

在實際流程中，Client 先呼叫 [[Google Cloud Functions]]，Function 以 [[Supabase JWKS]] 驗證 [[Supabase]] JWT，再建立 [[Google Cloud Storage]] Signed URL。Client 直接上傳圖片後，再請 AI analysis Function 產生結構化結果，避免圖片先穿過既有 App backend。

## Evolution

1. 以 Shadcn components 統一介面並改善表單體驗。
2. 完成日期範圍與關鍵字搜尋頁。
3. 規劃並實作 [[Expense Receipt AI Pipeline]]。
4. 在 Summary 階段加入 Group、Ungroup、Ignore、Category 與 Split mode，最後轉成 `ExpenseRow[]`。

## Related

- [[Side Projects]]
- [[Expense Receipt AI Pipeline]]
- [[Supabase]]
- [[OAuth for Browser Apps]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- [[2026-05-17 - 5-17 weekly update]]
