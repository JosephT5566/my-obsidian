---
type: how-to
status: growing
topics: [houzz, jukwaa, analytics, event-tracking, frontend]
created: "2026-07-14"
---

# Frontend Analytics Instrumentation

## Goal

在 [[Jukwaa]]／[[Houzz Marketplace]] 等 frontend surfaces，以一致的 hierarchy 與 naming 設計 UI click／impression events，讓 Amplitude 等分析工具可穩定查詢。Visibility event 的觸發機制可搭配 [[Intersection Observer]]，但仍要另外定義停留時間與重複曝光規則。

## Steps

1. 為頁面定義獨立的 Experience name。
2. 依 L1 Section、L2 Container、L3 Component 標註 UI hierarchy。
3. Container 放在實際 group（例如 `<ul>`），不要讓 Section 與 Container 重複表達同一層。
4. Component 使用可辨識名稱，例如 `Product Category Card`，避免只有 `Item`。
5. Click tracking 加入 `hz-track-me`；CTA event 同時提供 `data-cta`。
6. 更新允許的 attributes 定義，執行 validation 與 build 產生 method。
7. 在 tracking document 登記頁面實際使用的 events。

## Verification

- 在 analytics tool 可用 L3 component name 查到事件。
- Impression 不應掛在範圍過大的 Section。
- Container 與 Component naming 應能互相對應。

## Related

- [[Intersection Observer]]
- [[A-B Testing Rollout]]
- [[Jukwaa]]
- [[Houzz Marketplace]]

## References

- `Row notes/2023-01-09 - Omnilog.md`
