---
type: concept
status: growing
topics: [houzz, rpc, thrift, service-architecture]
created: "2026-07-14"
---

# Apache Thrift RPC

## Summary

Apache Thrift 是跨語言的 RPC framework，用 Interface Definition Language（IDL）定義 service contract，再產生 client/server code，讓不同語言的服務像呼叫本地函式一樣溝通。

## Why It Matters

在 [[Houzz]] 架構中，GraphQL 是 [[Jukwaa]] 等前端使用的 query layer，Thrift 則是 service-to-service layer：React → GraphQL resolver → Thrift client → [[C2]] PHP 或其他 Thrift service → database。

## How It Works

1. 在 `c2thrift` 定義 request、response 與 service method，作為 protocol 的 single source of truth。
2. 更新 [[C2]]／[[Jukwaa]] 的 git submodule pointer，執行 generator 產生語言專用型別。
3. 在 C2 `services/handlers` 實作 business logic。
4. 以 Mocha 或本機 Thrift client 當 client，測試 server response。

```text
GraphQL resolver → generated Thrift client → transport/protocol → service handler
```

## Tradeoffs

- 優點：強型別 contract、跨語言、適合內部服務呼叫。
- 成本：IDL、generated code 與多個 repository 的 submodule pointer 必須同步。
- Thrift endpoint 不是一般 HTTP API；用 `curl` 連線後停住，不代表 service 一定故障。

## Related

- [[C2 Development on Kubernetes]]
- [[C2-Jukwaa Web Module]]
- [[C2]]
- [[Jukwaa]]
- [[Kubernetes Architecture and Debugging]]

## References

- `Row notes/2022-10-21 - Thrift.md`
- `Row notes/2023-12-21 - C2 development.md`
