---
type: concept
status: growing
topics: [backend, production-readiness, reliability, security, observability, slo]
created: "2026-08-03"
---

# Backend Service Production Readiness

## Summary

Backend service readiness 不能用單一完成百分比判斷。較實用的模型是：MVP scope 定義這版做多少功能，mandatory release gates 決定能否安全上線，readiness scorecard 衡量成熟度，SLO 與 metrics 驗證上線後是否持續健康。

## Why It Matters

MVP 不等於 minimum safe product。即使是 internal service，也不能省略 authentication、authorization、input validation、failure handling 與 observability；可以依 service tier 降低公開網路防護、HA 或 support level，但「內網」本身不是信任邊界。

[[Expense Receipt AI Pipeline]] 的 GCS／AI processing flow 提供具體案例：圖片、object path、metadata 與 model output 都是外部輸入，必須先驗證 caller 與 resource ownership，再執行下載、provider call 或 persistence。

## How It Works

### Define Contract and Trust Boundaries First

開發前先定義 use case、request／response contract、domain model、resource owner、untrusted inputs 與 server-owned control parameters。Provider-specific output 應依序經過 SDK parsing、Application schema validation 與 normalization，只有 validated domain model 能進入 business logic。

推薦 runtime 順序：

```text
Parse request
→ Validate shape and limits
→ Authenticate caller
→ Authorize resource ownership
→ Inspect metadata
→ Load content
→ Execute expensive provider call
→ Validate and normalize result
```

Authentication 只證明 caller identity，不代表 request 中的 user ID、object ID 或 storage path 已獲授權；難猜的 UUID 也不是 authorization。

### Stable Success and Error Contracts

API 應提供 versioned response、stable status 與 error code，區分 caller 應修改輸入、停止 retry、重新上傳或稍後 retry。Raw provider wording 與任意 payload 不應直接成為 public contract；invalid provider output 也不能繼續進入 normalization 或 persistence。

### Security Is a Cross-Cutting Gate

在 AI integration 中，server-owned model、system instruction、schema、tools 與 generation settings 是 control plane；圖片、OCR 或使用者文字是 data plane。不可信內容可以影響合法資料值，但不能改寫 control plane。這與 [[AI Structured Output Validation]] 的原則一致：schema 降低格式錯誤，但不能取代 authorization、business validation 與 least privilege。

### Test Failure Paths and Side Effects

Tests 除了檢查 happy path、boundary values 與錯誤 status，也要驗證 rejected input 不會觸發 download、provider call、normalization 或 persistence。Negative test 必須證明 pipeline 已 short-circuit，而不只是 response 看起來正確。

### Four Readiness Mechanisms

| Mechanism | Question |
|---|---|
| MVP scope | 這一版需要哪些核心功能？ |
| Release gate | 哪些條件未完成就不能上線？ |
| Readiness scorecard | Service 各面向成熟到什麼程度？ |
| SLO and metrics | 上線後是否持續健康？ |

典型 release blockers 包含 cross-user access、sensitive-data leakage、untrusted input 能控制副作用、缺少 payload／file／concurrency limits，以及 critical failure 無法被發現或安全處理。Scorecard 的平均分數不能抵銷任何一項 blocker。

### Service Tier and Operations

- Internal Experimental：少數開發者測試，可採 best effort 與 business-hours support。
- Internal Shared：多團隊正式使用，需要 stable contract、service identity、caller allowlist、limits、alerts、rollback、owner、runbook 與基本 SLO。
- Internal Critical：阻塞財務或核心營運，進一步需要 HA、on-call、DR 與更嚴格 SLO。

Internal caller 代表 end user 操作資源時，必須分開驗證 caller service identity 與 end-user ownership context，不能信任 request 任意帶入的 `user_id`。

## Example

Receipt service 的 2026-08-02 readiness 記錄顯示，canonical schema、Gemini structured response、versioned normalized result、GCS ownership／image-input hardening，以及 prompt-injection control-plane protection 已有對應實作與 tests。仍待補的 production 能力包括 rate／concurrency limit、per-caller quota、cost protection、metrics／alerts／correlation tracing、real provider smoke test、retention policy、deployment rollback、owner 與 runbook。

可追蹤的指標包括：

- Service health：availability、p50／p95／p99 latency、error rate、saturation 與 dependency health。
- Business／AI quality：extraction success、needs-review、retryable-failure、人工修正率與 model／prompt／schema version 差異。
- Security signals：cross-user rejection、oversized upload、MIME mismatch、schema-invalid output 與 configuration-injection attempt。

SLO 應由 use case、provider latency、成本與 caller tolerance 校準；例如「99.5% 合法 receipt requests 在 10 秒內取得非 provider-failure 結果」只是一個待驗證的起點。

## Tradeoffs

- MVP 可以省略進階報表或高度自動化，但不能把 authorization 與 safe failure 當作日後加分項。
- Readiness scorecard 方便呈現成熟度，卻不能取代 binary release gates。
- Internal service 可以降低部分 public-facing requirements，但多一層 caller／end-user identity 和 operational ownership。
- Strict validation、limits 與 least privilege 增加開發工作，換來較小的 blast radius、成本風險與 incident ambiguity。
- SLO 過嚴會增加成本，過鬆則無法代表 caller 真正需要的可靠度。

## Related

- [[Expense Receipt AI Pipeline]]
- [[AI Structured Output Validation]]
- [[API Contract Design]]
- [[Reliable Background Job Processing]]
- [[System Design Foundations]]
- [[AI Development Memory]]

## References

- [[2026-08-02 weekly updates]]
- [google-ai-gcf PR #18](https://github.com/JosephT5566/google-ai-gcf/pull/18)
- [google-ai-gcf PR #19](https://github.com/JosephT5566/google-ai-gcf/pull/19)
- [google-ai-gcf PR #20](https://github.com/JosephT5566/google-ai-gcf/pull/20)
- [google-ai-gcf PR #21](https://github.com/JosephT5566/google-ai-gcf/pull/21)
- [google-ai-gcf PR #22](https://github.com/JosephT5566/google-ai-gcf/pull/22)
