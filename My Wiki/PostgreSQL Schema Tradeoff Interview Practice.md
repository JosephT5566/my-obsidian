---
type: learning-note
status: growing
source: "[[2026-07-19 weekly updates]]"
topics: [backend, interview, postgresql, data-modeling, jsonb]
created: "2026-07-19"
---

# PostgreSQL Schema Tradeoff Interview Practice

## Source

一題多人記帳 schema design 練習：分類名稱可修改，但歷史報表要保留交易當時的名稱；延伸比較 normalized share table 與 JSONB。這是面試學習情境，不是既有專案的實作紀錄。

## Key Ideas

- 先精準對應 entity 與欄位；`expense_name` 不能代替 `category_name_snapshot`。
- Foreign key 保存目前 entity relationship，snapshot 保存事件發生時的歷史語意。
- 一筆 expense 有多個 category 時，snapshot 應放在 allocation join row。
- 穩定且重要的 participant、amount 與 payment status 優先正規化。
- JSONB 的彈性會犧牲 element-level foreign key、型別、unique constraint、精細更新與直接查詢。
- Middleware 提供輸入驗證與友善錯誤；database 保護所有寫入路徑都必須成立的 invariant。

完整技術整理見 [[PostgreSQL Relational Data Modeling]]。

## Examples

面試回答可先說：「核心關聯正規化，歷史名稱使用 explicit snapshot。」接著列出 primary key、foreign key、unique／check constraint，再比較 JSONB 的適用條件與 concurrency 成本。

## Questions

- 分帳總額等於 expense total 要由哪個 database mechanism 保護？
- Category 被刪除時，relation 與 snapshot 分別如何處理？
- 什麼規模與 access pattern 會讓 JSONB 不再適合？

## How This Changes My Work

Schema 題不只描述資料如何保存，也要說明非法資料如何被阻止、transaction boundary 在哪裡，以及 concurrent writes 如何避免 lost update。

## Related

- [[Backend Engineering Interview Practice]]
- [[PostgreSQL Relational Data Modeling]]
- [[SQL vs NoSQL]]
- [[Supabase]]

## References

- [[2026-07-19 weekly updates]]
