---
type: concept
status: growing
topics: [supabase, jwks, jwt, authentication, asymmetric-cryptography]
created: "2026-07-15"
---

# Supabase JWKS

## Summary

JWKS（JSON Web Key Set）是發布 JWT verification keys 的標準 JSON 格式。[[Supabase]] 可透過公開 JWKS endpoint 提供 asymmetric JWT 的 public keys，讓 [[Google Cloud Functions]] 驗證 Access Token，而不必持有 signing private key 或 Legacy symmetric secret。

## Why It Matters

Legacy symmetric JWT 由同一個 secret 負責簽署與驗證；把 secret 複製給每個 backend，代表任何驗證端一旦外洩，也可能被用來偽造 token。Asymmetric JWT 由 private key 簽署、public key 驗證，能把「發行 token」與「驗證 token」的權限分開。

## How It Works

1. [[Expense App]] 將 Supabase Access Token 放在呼叫 [[Expense Receipt AI Pipeline]] endpoint 的 request 中。
2. GCF 讀取 JWT header 的 Key ID（`kid`）。
3. GCF 從 Supabase JWKS endpoint 取得並快取相符的 public key。
4. GCF 驗證 signature，以及 `iss`、`aud`、`exp` 等 claims。
5. 驗證成功後，Function 才能發行 [[Google Cloud Storage]] Signed URL 或執行 AI analysis。

```text
Supabase private key → sign JWT → Expense App
Supabase JWKS endpoint → public key → GCF verifies JWT
```

## Tradeoffs

- Public key 可以安全散布，降低跨服務共享 signing secret 的風險。
- 驗證端需要處理 Key ID、cache、refresh 與 key rotation；找不到 `kid` 時應重新取得 JWKS，但不能無限重試。
- Signature 有效不代表 request 一定可授權，仍需驗證 claims 與 application-level permissions。

## Related

- [[Supabase]]
- [[Google Cloud Functions]]
- [[Expense Receipt AI Pipeline]]
- [[OAuth for Browser Apps]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
