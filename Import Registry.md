# Import Registry

這份清單記錄 `Row notes` 的匯入狀態，供後續匯入流程判斷哪些來源是新增、未變更或內容已更新。

## 判斷規則

- Registry 沒有該路徑：視為新來源，需要匯入。
- 路徑相同且 SHA-256 相同：內容未變更，跳過匯入。
- 路徑相同但 SHA-256 不同：來源已更新，重新檢查並更新既有知識頁。
- 只有在 My Wiki、Index 與 Dialog 都成功更新後，才能新增或更新 Registry。
- `Output pages` 記錄此次來源建立或更新的 My Wiki 頁面；Registry 本身不是知識頁面，不列入 `Index.md`。

## Imported Sources

| Source | SHA-256 | Imported at | Status | Output pages |
|---|---|---|---|---|
| `Row notes/2026-04-27 - 4-27 進辦公室.md` | `1d027528e177a39dff10f2bd81ebbeefdb4898efeb19e5a202681583dd89ce28` | 2026-07-14 | imported | [[Graph Traversal - DFS and BFS]] |
| `Row notes/2026-04-30 - Dynamic programming.md` | `5c1cad825de6ef1be174a2df7b654145c501c81f267592a458fada3335de50e2` | 2026-07-14 | imported | [[Dynamic Programming]] |
| `Row notes/2026-05-10 - 5-10 weekly updates.md` | `8d48291e0924c4b89b5b1308b8a483c79b87f83cbca9db84186970a329f6ecc2` | 2026-07-14 | imported | [[Expense Receipt AI Pipeline]] |
| `Row notes/2026-05-17 - 5-17 weekly update.md` | `62d52884287334657a0e428c6a6de5187fb5b1b4d4b6923031f8bd11f47c71eb` | 2026-07-14 | imported | [[Expense Receipt AI Pipeline]] |
| `Row notes/2026-05-24 - 5-24 weekly update.md` | `c50d149fda923c9da07b4f541b3a4ac9726e230644d071406fe5bf9025ec47ed` | 2026-07-14 | imported | [[Trie]], [[AI Agent Architecture]], [[Distributed Transactions - 2PC and Saga]] |
| `Row notes/2026-05-31 - 5-31 weekly update.md` | `45fe5988ac34e2c2376b67f91aa64080e99145917bc361319f9419e66b08a12c` | 2026-07-14 | imported | [[Graph Traversal - DFS and BFS]], [[Interview Preparation with AI]] |
| `Row notes/2026-06-07 - 6-7 weekly update.md` | `3835e311da9455af0bfdf3226b14430474bca100fac09187bec3f99d3f4751c7` | 2026-07-14 | imported | [[SQL vs NoSQL]], [[Firebase Realtime App Architecture]], [[System Design Foundations]] |
| `Row notes/2026-06-14 - 6-14 weekly update.md` | `2f17dafdc803b8e65bed16f9746a2521a8d92bd198fa00a3e328290fbcd8df30` | 2026-07-14 | imported | [[Back-of-the-envelope Estimation]], [[Remote Codex with Self-hosted Runner]] |
| `Row notes/2026-07-12 - 7-12 weekly update.md` | `c96f927f92dca083a10f4beadd8e3e83f6156d81cbfbd3336f320bb2a4aa7923` | 2026-07-14 | imported | [[SQL vs NoSQL]], [[System Design Foundations]] |
