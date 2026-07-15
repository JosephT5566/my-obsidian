---
type: concept
status: growing
topics: [supabase, jwks, jwt, authentication, asymmetric-cryptography]
created: "2026-07-15"
---

# Supabase JWKS

## Summary

JWKS（JSON Web Key Set）是發布 JWT verification keys 的標準 JSON 格式。[[Supabase]] 可透過公開 JWKS endpoint 提供 asymmetric JWT 的 public keys，讓外部 verifier 驗證 Access Token，而不必持有 signing private key 或 Legacy symmetric secret。

## Why It Matters

Legacy symmetric JWT 由同一個 secret 負責簽署與驗證；把 secret 複製給每個 backend，代表任何驗證端一旦外洩，也可能被用來偽造 token。Asymmetric JWT 由 private key 簽署、public key 驗證，能把「發行 token」與「驗證 token」的權限分開。

## How It Works

1. Verifier 讀取 JWT header 的 Key ID（`kid`）與 algorithm。
2. Verifier 從 Supabase JWKS endpoint 取得並適度快取相符的 public key。
3. Verifier 驗證 signature，以及 `iss`、`aud`、`exp` 等 claims。
4. 驗證成功只代表 token authenticity；application 仍要檢查 user、role 與 operation-level permissions。

```text
Supabase private key → sign JWT → Client
Supabase JWKS endpoint → public key → External verifier
```

## Expense App Outcome

[[Expense App Authentication]] 已實際採用 JWKS 保護 [[Expense Receipt AI Pipeline]] 的 GCF endpoints。Client 將 Supabase Access Token 放在 `Authorization: Bearer`；GCF 透過 `https://{project_ref}.supabase.co/auth/v1/.well-known/jwks.json` 取得 signing key，以 `ES256` 和 `audience="authenticated"` decode token。`get_upload_url` 會進一步把 `sub` 放進 GCS object path，`analyze_receipt` 也會在執行 AI operation 前驗證 token。

目前 implementation 有兩個應追蹤的邊界：程式註解標示 `RS256`，但實際 decoder 只接受 `ES256`；另外 decoder 沒有明確傳入 expected issuer。這不改變「已採用 JWKS」的事實，但應讓註解、Supabase signing configuration 與完整 claims validation 保持一致。

## Tradeoffs

- Public key 可以安全散布，降低跨服務共享 signing secret 的風險。
- 驗證端需要處理 Key ID、cache、refresh 與 key rotation；找不到 `kid` 時應重新取得 JWKS，但不能無限重試。
- Signature 有效不代表 request 一定可授權，仍需驗證 claims 與 application-level permissions。

## Related

- [[Supabase]]
- [[Expense App Authentication]]
- [[OAuth for Browser Apps]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
- User clarification (2026-07-16)
- [Private implementation: google-ai-gcf/main.py](https://github.com/JosephT5566/google-ai-gcf/blob/main/main.py)
- [Supabase Auth: JSON Web Tokens](https://supabase.com/docs/guides/auth/jwts)
