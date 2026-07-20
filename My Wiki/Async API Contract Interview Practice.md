---
type: learning-note
status: growing
source: "[[2026-07-19 weekly updates]]"
topics: [backend, interview, api, async, idempotency]
created: "2026-07-19"
---

# Async API Contract Interview Practice

## Source

一題 Receipt AI API contract 練習：解析耗時、欄位可能無法辨識，Client 也可能在 timeout 後重送。Expense domain 只是題目情境，不代表既有 Receipt AI flow 已採用此 contract。

## Key Ideas

- 先定義 async job resource 與 `queued | processing | completed | failed`，再談 REST、JSON 或 implementation。
- `202 Accepted` 表示工作已可靠接受，不代表 parsing 已完成。
- 部分欄位無法辨識時，job 仍可 `completed`；欄位使用合法 JSON 的 `null`、status 與 confidence。
- HTTP status 表達通用語意，application error code 決定 Client 行為。
- Client retry 使用 [[Idempotent Request Handling]]；相同 key 與相同 payload 回同一個 job，不同 payload 回 `409`。
- Task-specific prompt 由 Server 管理和版本化，Client 只傳受控 hints。

完整技術整理見 [[API Contract Design]]。

## Examples

```http
POST /v1/receipt-parses
Idempotency-Key: 01J...

HTTP/1.1 202 Accepted
Location: /v1/receipt-parses/rp_123
```

Provider stack trace、內部 model 名稱與 credential 資訊不可暴露給 Client；使用安全 message、穩定 code 與 request／job ID 串聯觀測資料。

## Questions

- 哪些 validation error 使用 `400`，哪些使用 `422`？
- Client 如何得知 partial success，而不把它誤判為系統失敗？
- Idempotency record 的 scope、payload hash 與 retention 如何定義？

## How This Changes My Work

API design 回答採 contract-first：resource、endpoint、request、response、state transition、error semantics，最後才補 database 與 worker 實作。

## Related

- [[Backend Engineering Interview Practice]]
- [[API Contract Design]]
- [[Idempotent Request Handling]]
- [[Reliable Background Job Processing]]

## References

- [[2026-07-19 weekly updates]]
