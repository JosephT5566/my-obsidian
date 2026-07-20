---
type: concept
status: growing
topics: [api, http, json, error-handling, async]
created: "2026-07-19"
---

# API Contract Design

## Summary

API contract 應先定義 resource、request、response、狀態轉換與失敗語意，再選擇實作細節。對長時間工作，`202 Accepted` 與可查詢的 async job resource 通常比同步等待更能精準表達系統狀態。

## Why It Matters

REST、JSON 與 HTTP status 只是形式；Client 真正需要的是穩定、可預期且能決定下一步行為的語意。[[Async API Contract Interview Practice]] 以 AI parsing 題目練習如何區分接受、部分成功與永久失敗。

## How It Works

### Resource 與 State

- `POST` 建立工作，回 `202 Accepted` 與 `Location`。
- `GET` 取得 `queued | processing | completed | failed`。
- `202` 表示已接受，不代表下游工作已完成。
- Client timeout 後重送需搭配 [[Idempotent Request Handling]]，回到同一個 logical resource。

### Partial Success

工作已正常執行、但部分欄位無法辨識時，job 可以是 `completed`；欄位以 `null`、field status 與 confidence 表達品質。JSON 不支援 `undefined`。

### Error Contract

HTTP status 表達通用協定語意，application code 表達產品行為。Error response 可包含穩定 `code`、安全 `message`、`request_id`、`retryable` 與 retry hint。Client 依 `code` 行動，不解析 message，也不直接依賴 PostgreSQL 或 provider error。

常見區分：

- `400`：request 無法解析或基本格式錯誤。
- `401`／`403`：authentication 與 authorization。
- `409`：state 或 idempotency conflict。
- `422`：格式正確但內容無法處理。
- `429`／`503`：依 `Retry-After` 或 product policy 延後。
- `504`：同步架構中的 upstream timeout；async job 通常改由 job state 表達。

## Example

```json
{
  "error": {
    "code": "SERVICE_TEMPORARILY_UNAVAILABLE",
    "message": "The request could not be completed right now.",
    "request_id": "req_123",
    "retryable": true,
    "retry_after_seconds": 30
  }
}
```

## Tradeoffs

- Async resource 增加 polling、state storage 與 lifecycle 管理，但隔離長時間依賴的 latency 與失敗。
- Application codes 需要維護 compatibility，卻能避免 Client 與內部 provider 耦合。
- Task-specific API 由 Server 管理並版本化 prompt；Client 傳結構化 hints，可避免 contract drift 與 raw prompt injection。
- 第一版不必列完所有錯誤，只先定義會改變 Client 行為的類別。

## Related

- [[Async API Contract Interview Practice]]
- [[Idempotent Request Handling]]
- [[Reliable Background Job Processing]]
- [[System Design Foundations]]

## References

- [[2026-07-19 weekly updates]]
