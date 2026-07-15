---
type: how-to
status: growing
topics: [houzz, c2, email, testing, rq-worker]
created: "2026-07-14"
---

# Email Template Testing

## Goal

在 staging RQ debug pod 修改 [[C2]] email template，使用測試 script 或 Manage Order 發送測試信。Pod attachment 與 cluster context 可沿用 [[C2 Development on Kubernetes]] 的操作方式。

## Steps

1. 在 staging RQ dashboard 為對應 queue 建立 debug pod。
2. 切換到 batch cluster 與 `backend` namespace，attach VS Code 到 `rq-worker-container`。
3. 在 `/home/clipu/c2` 修改目標 email class 或 mock data。
4. 進入 `/batch` 執行測試 script，明確指定 case、staging user、收件信箱、order number 等參數。
5. 若屬於 order email，也可從 staging Manage Order 的 `Send Order Email` 觸發。

## Verification

- 信件寄到指定測試信箱。
- Template parent class 有改動時，至少跑過所有受影響的 cases。
- 不使用 production user 或真實顧客收件地址測試。

## Related

- [[C2 Development on Kubernetes]]
- [[Batch Jobs and Database Migrations]]
- [[C2]]
- [[Houzz]]

## References

- `Row notes/2024-11-08 - Houzz email.md`
