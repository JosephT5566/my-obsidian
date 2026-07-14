---
title: "Batch and Database"
date: 2024-12-03
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/1516ed462a2680da84e7f23436c3cfda"
notion_id: "1516ed462a2680da84e7f23436c3cfda"
---

# Batch
## How to run a batch
之前 Tom 有寫過一篇 doc: **How to Generate a batch of promotional coupons**
[https://docs.google.com/document/d/1UN3Y-Mz_a83SDT3EjKkNSA1Ip12Jjk3pElt8C1ldljU/edit?tab=t.0](https://docs.google.com/document/d/1UN3Y-Mz_a83SDT3EjKkNSA1Ip12Jjk3pElt8C1ldljU/edit?tab=t.0)
我們可以用 staging 的 rq 來測試
[https://rq.stghouzz.com/mp-batch-general-v2](https://rq.stghouzz.com/mp-batch-general-v2)
# Database
## Modify database
舉例來說，假如我要新增 column 到某個 table，做法會是用 sql 指令，新增 sql file 到 c2/sql/07XXX_your_db_change，檔案名就是接續現有的檔案
eg: PR - Add columns for indexation limitations [https://github.com/Houzz/c2/commit/37eb3f34da331193389537c15f9784088ccc787d](https://github.com/Houzz/c2/commit/37eb3f34da331193389537c15f9784088ccc787d)
**RDS schema change with AWX (Ansible): **[https://engwiki.houzz.tools/doc/rds-schema-change-with-awx-ansible-gFO0iJnqGO](https://engwiki.houzz.tools/doc/rds-schema-change-with-awx-ansible-gFO0iJnqGO)
## Related logic
我的問題是：我們會在 C2 的 code 中寫 query 來取得 data 來處理嗎
以 Order 來說，我們會在 c2/platform/C2Order.php 做 DB integration，然後再到 c2/platform/OrderManager.php 執行 Business logic
## Testing
eg: added test cases and core implementation for breadcrumb [https://github.com/Houzz/c2/pull/37895](https://github.com/Houzz/c2/pull/37895)
