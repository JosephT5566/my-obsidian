---
type: concept
status: growing
topics: [background-jobs, lease, sweeper, concurrency, recovery, distributed-systems]
created: "2026-07-27"
---

# Lease-Based Job Recovery

## Summary

Lease 是一種有期限的 ownership claim：worker 只在一段時間內擁有 job，時間到期後，系統可以讓其他 worker 接手。Sweeper（也常稱為 reaper 或 watchdog）則持續尋找 expired lease 與 stuck job，透過原子狀態轉換把失去 owner 的工作送回可執行流程。

## Why It Matters

在 [[Reliable Background Job Processing]] 中，worker 可能在 claim job 後 crash、失去網路，或卡在永遠不會完成的 external call。永久 lock 會讓 job 永遠停在 `processing`；lease 把 ownership 限制在有限時間內，而 sweeper 讓 recovery 不必等待 service restart。

這套設計提供的是 crash recovery 與 eventual progress，不是 exactly-once execution。同一個 job 仍可能被兩個 worker 先後執行，因此 side effect 必須搭配 [[Idempotent Request Handling]] 或 downstream provider 的 idempotency contract。

## How It Works

### Lease as a Time-bounded Claim

Job record 通常保存：

- `status`
- `lease_owner`
- `lease_expires_at`
- `lease_generation` 或 `fencing_token`
- `attempt_count`
- `next_retry_at`
- `updated_at`

Worker 透過 conditional update 或 row lock 原子取得 job，並設定 owner、到期時間及新的 generation。Lease duration 應長於正常處理時間的高百分位；長任務則由 heartbeat 定期續租，而不是一開始取得無限長的 lease。

續租只能在 `status = processing`、owner 與 generation 仍符合時成功。若續租失敗，worker 應視為已失去 ownership，停止啟動新的 side effect。

### Fencing Stale Workers

Lease 到期不代表舊 worker 已經停止。它可能只是暫停、網路分區，之後又繼續執行；此時新 worker 也可能已經接手。

每次 claim 都遞增 `lease_generation`。Worker 完成、續租或寫回結果時，必須在 conditional update 中帶上自己取得的 owner 與 generation。Sweeper 或新 worker 改變 generation 後，舊 worker 的 stale write 就會失敗。

若 side effect 發生在外部系統，只有 database fencing 不足以阻止重複效果。Downstream system 若支援 monotonic fencing token，可以拒絕舊 token；否則應使用穩定的 idempotency key，或在事後執行 reconciliation。

### Sweeper as a Recovery Loop

Sweeper 是週期性執行的 background process，分批掃描：

- `processing` 且 `lease_expires_at < now()` 的 job
- 超過預期時間仍未被 claim 的 `queued` job
- 到達 `next_retry_at` 的 retryable job

它不應先讀取再無條件更新，而要用 single statement 或 transaction 再次確認 status、expiry 與 generation，避免與正常 completion、heartbeat 或其他 sweeper 發生 race。Expired job 可以：

1. 推進 generation、清除 lease 並回到 `queued`。
2. 增加 `attempt_count`，進入 `retry_scheduled`。
3. 超過 retry budget 後轉為 terminal `failed` 或 dead-letter workflow。

Sweeper 可以有多個 instance，但每筆 recovery transition 必須是 atomic。掃描需使用支援 status、expiry 或 retry time 的 index，並限制 batch size，避免 recovery process 對 database 造成尖峰負載。

### Time and Tuning

Expiration 判斷最好以 database time 作為共同 clock，降低 application host clock skew 的影響。常見調整項目包括：

- `lease_duration`：太短會造成不必要的 duplicate execution；太長則讓 recovery 變慢。
- `heartbeat_interval`：需明顯短於 lease duration，並保留 transient failure 的容忍空間。
- `sweep_interval`：決定 expired job 被發現的延遲。
- `recovery_batch_size`：控制每輪 recovery 對 database 的壓力。
- `retry_backoff`：避免 external dependency 故障時形成 retry storm。

監控至少應包含 expired lease count、oldest expired age、reclaim rate、heartbeat failure、generation mismatch、recovery latency，以及同一 job 的 concurrent execution 訊號。

## Example

```text
Worker A claims job (generation 7, expires 10:05)
    ↓ Worker A pauses or loses network
Sweeper sees expiry and requeues job atomically
    ↓
Worker B claims job (generation 8)
    ↓
Worker A resumes, but generation 7 completion is rejected
Worker B completes with generation 8
```

簡化的 completion guard：

```sql
UPDATE jobs
SET status = 'completed',
    completed_at = now()
WHERE id = :job_id
  AND status = 'processing'
  AND lease_owner = :worker_id
  AND lease_generation = :generation
  AND lease_expires_at >= now();
```

Affected rows 為 `0` 表示 worker 已失去 ownership，不能把本次結果當成 canonical completion。

## Tradeoffs

- Lease 與 sweeper 能從 crash 和 stuck execution 恢復，但增加 state transition、index、monitoring 與 tuning 成本。
- 較短的 lease 提升 recovery speed，代價是 slow job 更容易被誤判為失效。
- Heartbeat 降低長任務被回收的機率，但 worker 與 database 會產生額外流量。
- Fencing token 能保護接受 token 的儲存層；對不支援 fencing 的 external side effect，仍要依賴 idempotency 或 reconciliation。
- Sweeper 本身也可能 crash，因此它應是可重入、可重複執行，且不能依賴單一 instance 的 in-memory state。

## Related

- [[Reliable Background Job Processing]]
- [[Expense Receipt AI Pipeline]]
- [[Idempotent Request Handling]]
- [[Transactional Outbox]]
- [[System Design Foundations]]

## References

- [[2026-07-26 weekly updates]]
