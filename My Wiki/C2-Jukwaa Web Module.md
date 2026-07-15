---
type: concept
status: growing
topics: [houzz, c2, jukwaa, react, graphql]
created: "2026-07-14"
---

# C2-Jukwaa Web Module

## Summary

Web Module 讓 legacy [[C2]] page 嵌入由 [[Jukwaa]] 提供的 React UI。C2 用 AJAX request 請求 module，Jukwaa handler／fetcher 經 GraphQL 與 [[Apache Thrift RPC|Thrift]] 取得資料後，回傳可渲染的 component bundle。

## How It Works

```text
C2 page
  → Web Module request parameters
  → Jukwaa handler and data fetcher
  → GraphQL resolver
  → C2 Thrift service
  → Jukwaa store and React component
```

以 Profile Header 新增 tab 為例，需要一起追蹤：C2 page 設定的 `currentTab`、Web Module request、Jukwaa handler、GraphQL variable/schema、C2 Navigation service、Thrift enum／contract 與 React store。完整環境操作可參考 [[C2 Development on Kubernetes]]；只修改 response 內容，不會自動讓 selected state 正確。

## Tradeoffs

- 優點：不用遷移整個 legacy page，就能重用 React logic 與 styles。
- 成本：debug path 跨 C2、Jukwaa、GraphQL 與 C2Thrift，多個 repo 必須保持 contract 一致。

## Related

- [[Apache Thrift RPC]]
- [[C2 Development on Kubernetes]]
- [[Prismic CMS Development]]
- [[C2]]
- [[Jukwaa]]
- [[Houzz]]

## References

- `Row notes/2024-12-16 - C2 Web module.md`
