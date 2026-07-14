---
type: concept
status: growing
topics: [webhook, event-driven, n8n]
created: "2026-07-14"
---

# Webhook Architecture

## Summary

Webhook 是 event producer 在事件發生時主動呼叫 consumer URL 的 push pattern，可取代 client 反覆 polling。Endpoint 必須處理驗證、快速回應、重試、去重與非同步工作。

## Example

LINE 排程訊息可以讓 LINE webhook 先進入輕量判斷層，再把真正的排程指令送入 n8n：

```text
LINE event → lightweight webhook → command filter → n8n workflow → database/scheduler → LINE API
```

Webhook event 才能提供受保護的 chat ID。若所有聊天事件都喚醒較重的 n8n workflow，會增加 cold start、延遲與資源成本；前置 Cloudflare Worker/Vercel function 可先驗證與篩選。

## Tradeoffs

- Push 降低 polling 流量，但 consumer 必須公開可達並承受重送。
- Handler 應快速回傳並把長工作排入 queue。
- 事件可能重複或亂序，資料寫入要 idempotent，並驗證 provider signature。

## Related

- [[Airflow Data Pipelines]]
- [[Distributed Transactions - 2PC and Saga]]
- [[System Design Foundations]]

## References

- `Row notes/2025-06-15 - Webhook.md`
