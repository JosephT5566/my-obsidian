---
type: how-to
status: growing
topics: [houzz, redis, cache, debugging]
created: "2026-07-14"
---

# Redis Debugging

## Goal

從 [[Houzz]] staging 或 production debug pod 查詢 Redis key，確認 [[C2]] 等 services 使用的 cache value；production 僅允許 read-only investigation。Pod 診斷建立在 [[Kubernetes Architecture and Debugging]] 的 container 與 cluster model 上。

## Steps

1. Staging 可從允許的網路直接用 `redis-cli -c -h <host> -p <port>` 連線。
2. Production 先在 RQ dashboard 建立 debug pod，再切到 production batch cluster。
3. 用 `kubectl exec` 進入 `rq-worker-container`，從 pod 連到 Redis cluster。
4. 用 `GET <key>` 等唯讀指令確認內容。

## Verification

- 使用 `-c` 支援 Redis Cluster redirect。
- 確認環境、host 與 key namespace 正確。
- Production 不執行 `SET`、`DEL`、flush 或其他 mutation。

## Related

- [[System Design Foundations]]
- [[Kubernetes Architecture and Debugging]]
- [[C2]]
- [[Houzz]]

## References

- `Row notes/2024-11-15 - Redis.md`
