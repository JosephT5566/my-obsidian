---
type: concept
status: seed
topics: [houzz, marketplace, experimentation, rollout, analytics]
created: "2026-07-14"
---

# A-B Testing Rollout

## Summary

A/B test 透過穩定分桶把流量分配到 control 與 treatment；ramp up/down 是調整 bucket coverage，而不是改變同一使用者的隨機結果。

## Example

[[Houzz]] 舊有 bucket 設定範例：1% rollout 可使用 control `0–9`、treatment `500–509`；ramp down 到 0% 時清空兩側 bucket ranges。[[Houzz Marketplace]] 的 growth features 是需要同時規劃 experiment isolation、analytics 與 rollout strategy 的實際案例。

## Tradeoffs

- 同時執行會影響相同頁面或 funnel 的 tests，可能互相污染結果。
- Ramp percentage 只是流量配置，仍需明確 success metrics、guardrails 與停止條件。
- 清空 bucket 前應確認 default behavior，避免 0% 狀態產生未預期 fallback。

## Related

- [[Frontend Analytics Instrumentation]]
- [[Marketplace Growth Experiments]]
- [[Houzz Marketplace]]
- [[Houzz]]

## References

- `Row notes/2022-12-08 - About AB test.md`
