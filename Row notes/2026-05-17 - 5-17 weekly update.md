---
title: "5/17 weekly update"
date: 2026-05-17
tags: ["Interview 2026"]
source: "https://app.notion.com/p/3666ed462a2680bd88e0f396c690ccdf"
notion_id: "3666ed462a2680bd88e0f396c690ccdf"
---

## Expense App
<mention-page url="https://app.notion.com/p/35f6ed462a26807db299ff938180ce59"/>
根據上週規劃好的 flow，這週花了幾天實現並測試完成。在接收到 ai api 的 response 後，針對其內容完成了額外的前端設計。
**Wizard flow** (**step-by-step flow):**
1. **Upload**: Upload image(s)
2. **Analysis**: Call ai api and analysis image(s), then display result. User can check and decide if we need to re-run the analysis.
3. **Summary**: Based on the received data, user can group/ungroup/ignore the items and fill up the missing fields, like category, split mode. At the end, convert to type `ExpenseRow[]` and save to DB.

