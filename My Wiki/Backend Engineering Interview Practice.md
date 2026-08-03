---
type: learning-note
status: growing
source:
  - "[[2026-07-19 weekly updates]]"
  - "[[2026-07-26 weekly updates]]"
  - "[[2026-08-02 weekly updates]]"
topics: [backend, interview, system-design, security, ai, learning]
created: "2026-07-19"
---

# Backend Engineering Interview Practice

## Source

2026-07-16 至 2026-07-26 的 Backend interview 練習紀錄。題目借用 expense domain 作為情境，但內容是架構推演與回答訓練，不代表任何既有專案已採用或實作這些設計。

## Key Ideas

- [[PostgreSQL Schema Tradeoff Interview Practice]]：練習 relation、historical snapshot、JSONB 與 database invariants。
- [[Async API Contract Interview Practice]]：練習 async resource、partial success、HTTP semantics 與 idempotency。
- [[Full-stack Error Handling Interview Practice]]：練習 client retry、worker retry、transaction、outbox、recovery 與 observability。
- [[PostgreSQL Row-Level Security]]：練習 ownership、tenant membership、`USING`／`WITH CHECK`、column immutability 與 organization-scoped admin。
- [[AI Structured Output Validation]] 與 [[Expense Draft Review Workflow]]：練習 untrusted model output、human review、version、confirm transaction 與 partial-failure recovery。
- [[High-Concurrency Coupon Claim Interview Practice]]：練習先定義限量資源 invariant，再以 conditional update／row lock、unique constraint、transaction 與 idempotency 分層保護。
- 技術回答應先定義資料模型、狀態與保證，再補充元件、SQL 或 HTTP status。
- 面對跨系統 failure window，不宣稱無法證明的 exactly-once；應說清楚 at-least-once、idempotency 與 reconciliation 的邊界。

## Examples

這組練習由三個層次組成：

1. Data model：entity、relationship、snapshot 與 source of truth。
2. API contract：resource、request、response、狀態轉換與 error semantics。
3. Production reliability：timeout、retry、crash recovery、queue delivery 與營運指標。

## Questions

- 能否在一分鐘內先講清楚 system guarantee，再展開 implementation？
- 每個 database、queue 與 provider boundary 發生 crash 時，下一次 retry 會怎樣？
- 哪些規則由 middleware、database constraint、trigger 或 transaction 保護？
- 哪些 metrics 與 correlation IDs 能證明系統可被營運？

## How This Changes My Work

回答 Backend design 題時，依序說明 resource／entity、state machine、invariants、failure windows、recovery 與 observability。使用具體 domain 幫助推演，但要明確區分「學習情境」與「實際專案決策」。

## Related

- [[Interview Preparation with AI]]
- [[System Design Foundations]]
- [[PostgreSQL Relational Data Modeling]]
- [[API Contract Design]]
- [[Reliable Background Job Processing]]
- [[PostgreSQL Row-Level Security]]
- [[AI Structured Output Validation]]
- [[Expense Draft Review Workflow]]
- [[High-Concurrency Coupon Claim Interview Practice]]

## References

- [[2026-07-19 weekly updates]]
- [[2026-07-26 weekly updates]]
- [[2026-08-02 weekly updates]]
