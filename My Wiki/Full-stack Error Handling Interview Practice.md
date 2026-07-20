---
type: learning-note
status: growing
source: "[[2026-07-19 weekly updates]]"
topics: [backend, interview, error-handling, reliability, outbox, observability]
created: "2026-07-19"
---

# Full-stack Error Handling Interview Practice

## Source

一題端到端 failure handling 練習：Client 提交資料、API 寫入 PostgreSQL、worker 呼叫不穩定的 AI provider，並推演 database commit 與 queue publish 之間的 crash。這是 Backend system design 學習筆記，不是既有專案的 production architecture。

## Key Ideas

- 先可靠保存 submission 與 job，再非同步處理外部 AI。
- Client retry 用同一 idempotency key 確認提交結果；worker retry 由 job state、error category、attempt 與 retry schedule 控制。
- Disable button 是 UX，不是防止重複資料的 correctness guarantee。
- Durable job 使用 atomic claim、lease 與 sweeper；recovery 不能只在服務重啟時執行。
- Worker call 設 timeout，對暫時性錯誤使用 exponential backoff 加 jitter 與 bounded retry。
- 只有需要可靠發布到外部 queue 時才使用 [[Transactional Outbox]]；database-backed job 的 MVP 不一定需要。
- Outbox 仍是 at-least-once delivery，consumer 必須 idempotent。

完整技術整理見 [[Reliable Background Job Processing]]。

## Examples

將狀態分開：business submission、background job 與 outbox event 各有獨立生命週期。以 request ID、submission ID、job ID 與 event ID 串聯 logs／traces，監控 oldest pending age、retry rate、terminal failure、provider timeout 與 outbox backlog。

## Questions

- Worker 在 provider 成功、本地結果保存前 crash 時如何 reconciliation？
- 哪些錯誤可 retry，哪些應進 terminal state 或 manual input？
- Queue publish 成功但 outbox 尚未標記時，重複 event 如何處理？

## How This Changes My Work

System design 回答要依序推演「寫入前 crash、寫入後 crash、回應前 crash、重試時會怎樣」，並以狀態機、恢復機制與 observability 證明系統可營運。

## Related

- [[Backend Engineering Interview Practice]]
- [[Idempotent Request Handling]]
- [[Reliable Background Job Processing]]
- [[Transactional Outbox]]
- [[API Contract Design]]

## References

- [[2026-07-19 weekly updates]]
