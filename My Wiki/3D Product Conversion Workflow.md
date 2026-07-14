---
type: concept
status: growing
topics: [3d-model, catalog, websocket, state-management]
created: "2026-07-14"
---

# 3D Product Conversion Workflow

## Summary

2D-to-3D workflow 以 CatalogItem 表示產品，以 ConvertedModelData 表示生成中的 3D model。`externalSourceId` 連回 catalog item，`convertedModelId` 是後續 review、accept 與 reject 的核心關聯鍵。

## How It Works

狀態依序為 `PENDING → REVIEWING → ACCEPTED`，使用者放棄時轉為 `REJECTED`。Provider 在 mount 時取得 pending/review lists；backend 完成處理後以 WebSocket 通知前端 refetch。

使用者點 My Products item 時：若 `externalSourceId` 已存在於 review list，就組合 CatalogItem 與 convertedModelId 開啟 review modal；否則進入建立 model 流程。接受或拒絕會透過 mutation 更新 backend status。

Houzz Pro Clipper 可把其他網站的 product 收進 Pro Library；它以 iframe 執行，staging 測試需把 iframe 指向 staging domain。Painting/Rug 新類型共用 converted model table，但需要額外 `type` 區分 review UI，且可略過家具的 orientation step。

## Related

- [[TanStack Query Server State]]
- [[Webhook Architecture]]
- [[SQL vs NoSQL]]

## References

- `Row notes/2025-10-09 - 3D Clipper.md`
- `Row notes/2025-12-10 - Converted model.md`
