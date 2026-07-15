---
type: concept
status: growing
topics: [houzz, jukwaa, browser-api, visibility, web-performance]
created: "2026-07-14"
---

# Intersection Observer

## Summary

`IntersectionObserver` 非同步觀察 target element 與 viewport 或指定 root 的相交比例，適合 [[Jukwaa]] 等 frontend 的 visibility tracking、lazy loading 與 infinite scrolling。若用於產品事件，還要配合 [[Frontend Analytics Instrumentation]] 的 naming 與曝光規則。

## How It Works

Callback 會收到 `IntersectionObserverEntry[]`。每當相交比例跨過 `threshold`，observer 就會通知，而不是要求程式在每次 scroll event 自行量測位置。

```javascript
const observer = new IntersectionObserver(
  ([entry]) => {
    if (entry.isIntersecting) loadContent();
  },
  { root: null, rootMargin: '200px 0px', threshold: 0 }
);

observer.observe(target);
```

- `root`：判斷相交的容器，`null` 代表 viewport。
- `rootMargin`：向外或向內調整偵測邊界，可提早載入內容。
- `threshold`：觸發比例，可為單一數值或陣列。

## Tradeoffs

它比 scroll handler 更適合可見性偵測，但 callback 代表跨過門檻，不代表元素持續可見；analytics 還需要停留時間與重複曝光規則。

## Related

- [[Web Rendering Best Practices]]
- [[Frontend Analytics Instrumentation]]
- [[Jukwaa]]

## References

- `Row notes/2023-04-19 - IntersectionObserver.md`
