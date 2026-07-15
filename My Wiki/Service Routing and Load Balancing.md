---
type: concept
status: growing
topics: [houzz, load-balancing, routing, elb, microservices]
created: "2026-07-14"
---

# Service Routing and Load Balancing

## Summary

Load balancer 位於 service 入口，把 request 分配到 target group 與實際 backend instances/pods。不同 traffic pools 可使用不同 endpoints，以隔離 workload、便於 debug 或獨立擴縮。

## How It Works

```text
Client → API/GraphQL service → ELB → Target Group → EC2 or Kubernetes Pods
```

Routing ownership 應放在 service/config/infra layer，而不是讓 GraphQL resolver 直接知道底層 ELB。需要新的 traffic pool 時，常見做法是由既有 wrapper service 根據 pool 選 endpoint，或在 service registry 中加入明確 abstraction。

在 [[Houzz]] 的 [[Jukwaa]]／Prismic route 使用案例中，Route path 的完整啟用通常有三層：application AppConfig、local reverse proxy（僅本機需要）、staging／production load balancer configuration。底層 targets 若由 Pods 提供，則會接到 [[Kubernetes Architecture and Debugging]] 的 Service 與 Deployment model。

## Tradeoffs

- 每個 pool 獨立 ELB 可提高隔離與可觀測性，但增加 endpoint、certificate、health check 與 infra 管理成本。
- 另建 service 能形成清楚邊界，但需要 contract、handler、registry 與 deployment 全套維護。

## Related

- [[Kubernetes Architecture and Debugging]]
- [[Prismic CMS Development]]
- [[System Design Foundations]]
- [[Houzz]]
- [[Jukwaa]]

## References

- `Row notes/2025-02-26 - Hugedeer ELB.md`
- `Row notes/2025-06-03 - Set new route.md`
