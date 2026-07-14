---
type: concept
status: growing
topics: [kubernetes, containers, networking, debugging]
created: "2026-07-14"
---

# Kubernetes Architecture and Debugging

## Summary

Kubernetes 用 declarative resources 管理 containerized workloads：Cluster 包含 control plane 與 nodes；Node Pool 聚合相同規格的 nodes；Deployment 維持 Pods；Service 以穩定 DNS/VIP 把流量送到 label-matched Pods。

## How It Works

```text
Cluster
├─ Control Plane (scheduler/controllers)
└─ Node Pools
   └─ Nodes
      └─ Pods
         └─ Containers
```

CI build/test 並產生 immutable image，CD 更新 Deployment image reference，Deployment controller 執行 rolling rollout。新 Pod 通過 readiness 後才導流，舊 Pod 再 drain/terminate。

Pod IP 會因重建而改變；Service 用 selector 找到多個 Pods，並提供穩定的 `service.namespace.svc.cluster.local`。`ClusterIP` 給 cluster 內使用，`NodePort`/`LoadBalancer` 提供不同形式的外部入口。

## Debugging

- `kubectl get pods -o wide`：確認狀態、node 與 Pod IP。
- `kubectl describe` 與 logs：檢查 scheduling、probe、image 與 runtime errors。
- `kubectl exec`：進入 container 做必要的診斷。
- `kubectl port-forward service/<name> <local>:<remote>`：經 API server 暫時把本機 port tunnel 到 Pod，不應當作正式 routing。

## Tradeoffs

多 cluster 能縮小 blast radius、隔離 IAM/network/resources 並獨立升級，但增加 cross-cluster routing、observability 與治理成本。Node Pools 讓 GPU/batch/general workloads 分開擴縮與優化成本。

## Related

- [[System Design Foundations]]
- [[Service Routing and Load Balancing]]
- [[C2 Development on Kubernetes]]

## References

- `Row notes/2025-12-19 - Houzz K8S.md`
