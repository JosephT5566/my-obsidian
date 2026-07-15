---
type: concept
status: growing
topics: [houzz, jukwaa, react, tanstack-query, cache, server-state]
created: "2026-07-14"
---

# TanStack Query Server State

## Summary

TanStack Query（原 React Query）管理 server state 的 fetching、cache、loading／error、background refetch 與同步。它可用於 [[Jukwaa]] 或 [[Houzz Marketplace]] 的 React workflows，例如 [[3D Product Conversion Workflow]] 的 model lists；它不是一般 global state store，cache persistence 與 authentication session 續期必須分開設計。

## How It Works

- `staleTime`：資料從 fresh 轉為 stale 前的時間；stale 後可以 background refetch。
- `gcTime`：query 沒有 observer 後，cache 保留多久才回收。
- `useQuery`：定義 queryKey、queryFn 與 lifecycle policy。
- `setQueryData`：同步寫 cache 並更新 data timestamp，不會改掉 query policy。
- `invalidateQueries`：主動標記資料 stale 並視情況 refetch。

```typescript
queryClient.setQueryData(['user'], user);
queryClient.invalidateQueries({ queryKey: ['user'] });
```

`initialData` 會進入 cache 並參與 freshness 計算；`placeholderData` 只用於暫時顯示。Persist Query Client 能把 cache 同步到 storage，但 token 到期後的 sign-in/refresh 應由 auth lifecycle 負責，不應只靠 query 變 stale 就把使用者登出。

## Tradeoffs

長 `staleTime` 減少 request，代價是資料可能較舊；長 `gcTime` 改善返回頁面的體驗，但增加記憶體。持久化 access token 需同時評估 XSS、token expiry 與 refresh strategy。

## Related

- [[OAuth for Browser Apps]]
- [[React Ref Composition]]
- [[3D Product Conversion Workflow]]
- [[Jukwaa]]
- [[Houzz Marketplace]]

## References

- `Row notes/2026-01-09 - React query.md`
