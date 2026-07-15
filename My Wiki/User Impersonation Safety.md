---
type: how-to
status: growing
topics: [houzz, c2, authentication, debugging, user-safety]
created: "2026-07-14"
---

# User Impersonation Safety

## Goal

以最小風險檢查特定使用者在 [[Houzz]]／[[C2]] UI 看到的權限或資料狀態。

## Prerequisites

- 取得 Users Manager 的授權。
- 優先使用 staging/test account；只有必要時才在 production impersonate。

## Steps

1. 在 Users Manager 以 user ID/name 找到帳號。
2. 選擇 Impersonate / Sign In As User，或從 private profile 的 admin toolbar 進入。
3. 僅檢視重現問題所需頁面，不執行寫入、下單、發訊息或修改設定。
4. 完成後立即使用 `Undo impersonation`，再確認已回到自己的帳號。

## Verification

- Debug 過程沒有對使用者資料造成修改。
- 工作結束時已解除 impersonation。
- Local 若需 session，僅複製必要 cookies，並限制到 local domain。

## Related

- [[OAuth for Browser Apps]]
- [[C2 Development on Kubernetes]]
- [[C2]]
- [[Houzz]]

## References

- `Row notes/2024-11-14 - Houzz Users Manager.md`
