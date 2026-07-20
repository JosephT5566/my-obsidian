---
type: concept
status: growing
topics: [background-jobs, reliability, queue, retry, concurrency, observability]
created: "2026-07-19"
---

# Reliable Background Job Processing

## Summary

可靠背景工作的實際目標是「先 durable persist、允許 at-least-once execution、以 idempotency 吸收重複」，並讓 timeout、worker crash 與 stuck job 都能被持續偵測和恢復。

## Why It Matters

只在 Server restart 時掃描未完成工作並不足夠；服務可能一直存活，但某個 job 永久遺失。[[Full-stack Error Handling Interview Practice]] 以面試情境練習 durable job table、lease、retry 與 sweeper；這些內容是設計推演，不是已部署系統。

## How It Works

### Durable Job State

Job table 保存 `status`、`attempt_count`、`next_retry_at`、`locked_by`／`worker_id`、`lease_expires_at`、timestamps 與正規化 error category。典型狀態是：

```text
queued → processing → completed
                  ↘ queued / retry_scheduled
                  ↘ failed
```

名稱可以不同，但 database、API 與 monitoring 必須一致。

### Claim 與 Recovery

多個 worker 使用 conditional update 或 `SELECT ... FOR UPDATE SKIP LOCKED` atomic claim。Worker crash 後，lease 到期讓其他 worker 接手；background sweeper 持續尋找過久的 queued job 與 lease-expired processing job。

### Retry

為每次外部 call 設 timeout，只重試 network error、provider `429` 與部分暫時性 `5xx`。使用 exponential backoff 加 jitter、最大 attempt 或 retry window；invalid input、unsupported file、permission misconfiguration 與 terminal business state 通常不可重試。

### Database Polling 與 Queue

小型系統可讓 worker polling database job table，直接使用同一個 transaction 建立業務資料與 job。需要 backpressure、scheduled delivery、獨立擴展或跨服務 integration 時再加入 queue；多數 queue 仍是 at-least-once delivery，consumer 必須配合 [[Idempotent Request Handling]]。

## Example

```text
API transaction → business row + job row → commit → worker claim
                                             ↓ crash
                                   lease expiry + sweeper → retry
```

## Tradeoffs

- Database polling 元件少，但要管理 polling interval、locking 與 database load。
- Queue 改善派送與擴展，卻增加 delivery semantics、consumer deduplication 與 operational complexity。
- Provider 成功、本地保存前 crash 是單一 database transaction 無法關閉的 failure window；優先使用 provider idempotency、可查詢 job ID 或 reconciliation。
- 若 database commit 後還必須可靠發布 queue event，使用 [[Transactional Outbox]]，避免直接 dual write。
- Observability 至少包含 oldest pending age、processing duration、retry／terminal failure rate、provider latency 與 stuck lease；correlation IDs 串聯 API、job、event 與 provider。

## Related

- [[Full-stack Error Handling Interview Practice]]
- [[Async API Contract Interview Practice]]
- [[Idempotent Request Handling]]
- [[Transactional Outbox]]
- [[System Design Foundations]]

## References

- [[2026-07-19 weekly updates]]
