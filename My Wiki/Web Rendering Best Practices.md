---
type: concept
status: growing
topics: [houzz, jukwaa, marketplace, web-performance, images, seo, links]
created: "2026-07-14"
---

# Web Rendering Best Practices

## Summary

圖片與連結的 markup 會直接影響 Layout Stability、loading cost、accessibility 與 SEO。這些原則可重用於 [[Jukwaa]] 與 [[Houzz Marketplace]] pages：預先保留圖片空間、只延後非關鍵圖片、輸出正確尺寸與格式，並依目的地設定 link attributes。

## How It Works

### Images

- 設定原始 `width` 與 `height`，讓瀏覽器先計算 aspect ratio，降低 Cumulative Layout Shift（CLS）。
- 非首屏圖片使用 `loading="lazy"`；內容圖片提供有意義的 `alt`，裝飾圖才使用空字串。
- 依 viewport 與 device pixel ratio 輸出合理尺寸，不要要求超過原圖的解析度。
- 照片優先使用 JPEG；主要是文字的視覺應盡量以 HTML/CSS 呈現。
- [[Houzz]] `fimg` 適合依 URL parameters 動態產生 thumbnail；`simg` 適合既有高解析尺寸。

```html
<img src="photo.jpg" width="800" height="600" loading="lazy" alt="產品正面照">
```

### Links

- 內部連結通常不需要 `nofollow`。
- `target="_blank"` 會開新分頁；對外部頁面應搭配合適的 `rel` policy。
- 若 UI library 自動加上不符合情境的 `rel`，需明確覆寫，但必須先確認安全與 SEO 規則。

## Tradeoffs

Lazy loading 用在首屏主圖可能延後 Largest Contentful Paint。圖片 URL 服務能降低傳輸成本，但多一層轉換與 cache invalidation 管理。

## Related

- [[Intersection Observer]]
- [[System Design Foundations]]
- [[Jukwaa]]
- [[Houzz Marketplace]]

## References

- `Row notes/2022-12-14 - Image lazy loading.md`
- `Row notes/2023-05-25 - About Link.md`
- `Row notes/2023-09-08 - Image best practice.md`
