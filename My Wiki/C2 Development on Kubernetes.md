---
type: how-to
status: growing
topics: [houzz, c2, kubernetes, debugging, codepath]
created: "2026-07-14"
---

# C2 Development on Kubernetes

## Goal

在 [[Kubernetes Architecture and Debugging|Kubernetes]] 環境中，用 AWS SSO、EKS context、Codepath pod 與 VS Code remote attachment 調試 [[C2]] 或 C2Thrift code。

## Prerequisites

- AWS CLI v2、`kubectl`、正確的 AWS profile 與 kubeconfig。
- VS Code Kubernetes/Dev Containers extensions。

## Steps

1. 設定 `AWS_PROFILE` 並執行 `aws sso login --profile <profile>`。
2. 用 `aws eks update-kubeconfig --profile <profile> --name <cluster> --alias <cluster>` 建立 context。
3. 以 `kubectx` 切換 cluster，確認 namespace 與 deployment。
4. 部署 branch Codepath，取得 codepath cookie 與 pod。
5. Attach VS Code 到正確 container：一般 C2 page 用 `c2web`；[[Jukwaa]]／GraphQL 所呼叫的 service 通常在 `c2thrift`。
6. 在 `/home/clipu/c2` 修改與觀察 log；C2 JS/CSS 另開啟 Debug JS/CSS。
7. [[Apache Thrift RPC|Thrift]] contract 的正式改動必須在 `c2thrift` repo 完成，再 bump C2/Jukwaa submodule pointer。

## Verification

- `kubectl get deployment -n <namespace>` 能列出資源。
- Staging 使用 codepath cookie 後能看到 branch 變更。
- Thrift generator 已執行，C2、Jukwaa 與 contract pointer 一致。

## Troubleshooting

- GUI IDE 找不到 `aws`/`kubectl`：GUI 的 PATH 與 shell 不同，設定 extension executable path 並確認 default shell。
- 無法載入 namespaces：確認 kubeconfig 使用 AWS profile，不要混用 deprecated Teleport config。
- Apple Silicon attach 問題：安裝 Dev Containers 並使用正確 shell。

## Related

- [[Apache Thrift RPC]]
- [[Kubernetes Architecture and Debugging]]
- [[C2-Jukwaa Web Module]]
- [[C2]]
- [[Jukwaa]]
- [[Houzz]]

## References

- `Row notes/2023-12-21 - C2 development.md`
- `Row notes/2025-03-21 - Codepath.md`
