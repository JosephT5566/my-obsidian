---
type: concept
status: growing
topics: [houzz, marketplace, product, growth, frontend]
created: "2026-07-15"
---

# Houzz Marketplace

## Summary

Houzz Marketplace 是 [[Houzz]] 的產品與工程知識領域之一。這個入口把 growth experiments、content/SEO、catalog browsing 與 3D product workflows 連到它們使用的通用技術。

## Product Areas

- [[Marketplace Growth Experiments]]：Category／Guide pages、recommendation surfaces、browsing continuity 與 email capture。
- [[3D Product Conversion Workflow]]：CatalogItem、ConvertedModel status、WebSocket notification、review flow 與 Pro Clipper。
- [[Prismic CMS Development]]：為 Guide page 等內容建立 CMS type、slice、GraphQL fragment 與 route。

## Engineering Practices

Marketplace feature 需要同時考慮 [[A-B Testing Rollout|experiment rollout]]、[[Frontend Analytics Instrumentation|analytics taxonomy]]、[[Web Rendering Best Practices|SEO and rendering]] 與 API ownership。React UI 的 server state 可以使用 [[TanStack Query Server State]] 管理；visibility tracking 或 lazy loading 可使用 [[Intersection Observer]]。

## Related

- [[Houzz]]
- [[Jukwaa]]
- [[Marketplace Growth Experiments]]
- [[3D Product Conversion Workflow]]

## References

- [[2023-08-07 - 08-06 Tech sharing]]
- [[2025-10-09 - 3D Clipper]]
- [[2025-12-10 - Converted model]]
