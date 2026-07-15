---
type: concept
status: growing
topics: [houzz, jukwaa, react, graphql, frontend-platform]
created: "2026-07-15"
---

# Jukwaa

## Summary

Jukwaa 是 [[Houzz]] 以 React 與 GraphQL 為主的 frontend application platform。它可以提供完整 page，也能經由 [[C2-Jukwaa Web Module]] 把 React UI 嵌入 legacy [[C2]] page。

## Architecture

```text
React component
→ Jukwaa store / data fetcher
→ GraphQL resolver
→ generated Thrift client
→ C2 service handler
```

GraphQL 負責 frontend-facing query layer，[[Apache Thrift RPC]] 負責跨 repository、跨語言的 service contract。修改 contract 時，需要同步 `c2thrift` definition、generated types，以及 C2/Jukwaa repositories 的 submodule pointer。

## Development Areas

- [[C2-Jukwaa Web Module]]：處理 C2 page 與 Jukwaa React module 的整合與 selected state。
- [[Prismic CMS Development]]：建立 content type、slice、GraphQL fragment、component 與 route。
- [[Houzz Localization Lang Files]]：透過 Webpack lang-loader 產生 localization metadata。
- [[Frontend Analytics Instrumentation]]：以 Experience、Section、Container、Component 標註 UI events。
- [[TanStack Query Server State]]：管理 server state cache、refetch 與 lifecycle。
- [[React Ref Composition]]、[[Intersection Observer]] 與 [[Web Rendering Best Practices]]：可跨 Jukwaa features 重用的 frontend patterns。

## Related

- [[Houzz]]
- [[C2]]
- [[C2-Jukwaa Web Module]]
- [[Apache Thrift RPC]]

## References

- [[2024-12-16 - C2 Web module]]
- [[2023-11-24 - Prismic]]
- [[2025-03-21 - Houzz L10n]]
