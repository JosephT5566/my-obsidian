---
type: how-to
status: growing
topics: [cms, prismic, graphql, routing]
created: "2026-07-14"
---

# Prismic CMS Development

## Goal

在 Prismic CMS 新增 document type、slice 或 route，並從 GraphQL 到 Jukwaa page 完成驗證。

## Prerequisites

- 能存取 staging Prismic 與 GraphQL playground。
- 已了解 Type 是 document template、Doc 是依 Type 建立且可有 locale 的內容實例、Slice 是可重用內容區塊。

## Steps

1. 在 staging 建立或修改 Type、Doc 與 Slice fields。
2. 為 document 設定 data fetcher 與 GraphQL schema，包含 primary part 與 bodies。
3. 執行 `npm run build-fragment`，更新 slice fragments 與 mapping。
4. 在 Jukwaa/Prismic template 實作對應 component，先用 staging playground 驗證 schema 與 data。
5. 在 AppConfig 加入 route mapping。
6. Local 若需該 route，更新 nginx config 後重啟 local environment；否則直接在 staging 驗證。
7. 正式 routing change 透過 load balancer change process 更新 staging，再完成 production change。

## Troubleshooting

- Preview 預設可能讀 production data；用環境 cookie 指向 staging。
- `no service name specified` 可能是 feature evaluation service 未連線，應檢查 GrowthBook/Thrift endpoint configuration。
- Route 在 AppConfig 存在但仍無法進入時，檢查 local nginx 或 staging load balancer mapping。

## Related

- [[C2-Jukwaa Web Module]]
- [[Service Routing and Load Balancing]]
- [[Marketplace Growth Experiments]]

## References

- `Row notes/2023-11-24 - Prismic.md`
- `Row notes/2025-06-03 - Set new route.md`
