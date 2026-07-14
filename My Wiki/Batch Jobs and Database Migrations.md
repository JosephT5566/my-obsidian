---
type: how-to
status: growing
topics: [batch, database, migration]
created: "2026-07-14"
---

# Batch Jobs and Database Migrations

## Goal

安全測試 batch job，並以可追蹤的 schema migration 更新資料庫結構。

## Steps

### Batch job

1. 在 staging RQ 對應 queue 建立/執行 job。
2. 使用 debug pod 觀察執行環境、輸入與 log。
3. 以小範圍資料先驗證，再規劃 production rollout 與重跑策略。

### Database migration

1. 在 C2 的 migration 目錄依序號新增 SQL file，不直接手動更改 production schema。
2. 將 database integration 放在 platform/data layer，business logic 放在 manager/service layer。
3. 補上受 schema 或 logic 影響的 tests。
4. 依 RDS schema change process 透過 AWX/Ansible 執行與稽核。

## Verification

- Migration 可重現、版本化且有 rollout/rollback 考量。
- Batch job 可安全重跑，或明確說明不具 idempotency。
- Application code 與 schema deployment 順序相容。

## Related

- [[Airflow Data Pipelines]]
- [[SQL vs NoSQL]]
- [[Email Template Testing]]

## References

- `Row notes/2024-12-03 - Batch and Database.md`
