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
| `Row notes/2026-05-10 - 5-10 weekly updates.md` | `8d48291e0924c4b89b5b1308b8a483c79b87f83cbca9db84186970a329f6ecc2` | 2026-07-16 | imported | [[Travel Split App]], [[Travel Split Backend Migration]], [[Expense App]], [[Expense App Authentication]], [[Expense Receipt AI Pipeline]], [[Google Cloud Functions]], [[Google Cloud Storage]], [[Supabase]], [[Supabase JWKS]] |
| `Row notes/2026-05-17 - 5-17 weekly update.md` | `62d52884287334657a0e428c6a6de5187fb5b1b4d4b6923031f8bd11f47c71eb` | 2026-07-15 | imported | [[Expense App]], [[Expense Receipt AI Pipeline]] |
| `Row notes/2026-05-24 - 5-24 weekly update.md` | `c50d149fda923c9da07b4f541b3a4ac9726e230644d071406fe5bf9025ec47ed` | 2026-07-14 | imported | [[Trie]], [[AI Agent Architecture]], [[Distributed Transactions - 2PC and Saga]] |
| `Row notes/2026-05-31 - 5-31 weekly update.md` | `45fe5988ac34e2c2376b67f91aa64080e99145917bc361319f9419e66b08a12c` | 2026-07-14 | imported | [[Graph Traversal - DFS and BFS]], [[Interview Preparation with AI]] |
| `Row notes/2026-06-07 - 6-7 weekly update.md` | `3835e311da9455af0bfdf3226b14430474bca100fac09187bec3f99d3f4751c7` | 2026-07-15 | imported | [[Wedding Table Service]], [[Firebase Realtime App Architecture]], [[SQL vs NoSQL]], [[System Design Foundations]] |
| `Row notes/2026-06-14 - 6-14 weekly update.md` | `2f17dafdc803b8e65bed16f9746a2521a8d92bd198fa00a3e328290fbcd8df30` | 2026-07-14 | imported | [[Back-of-the-envelope Estimation]], [[Remote Codex with Self-hosted Runner]] |
| `Row notes/2026-07-12 - 7-12 weekly update.md` | `c96f927f92dca083a10f4beadd8e3e83f6156d81cbfbd3336f320bb2a4aa7923` | 2026-07-14 | imported | [[SQL vs NoSQL]], [[System Design Foundations]] |
| `Row notes/2022-10-21 - Thrift.md` | `e6d695cbce75bd7bacc6a71a741895bb2b808a776b1c358a0888b1e3681c82c1` | 2026-07-14 | imported | [[Apache Thrift RPC]] |
| `Row notes/2022-11-11 - Fork Ref.md` | `3943aea77309254c165a19ec85ecc41993ccb0dd41764764dc719a29c6025baa` | 2026-07-14 | imported | [[React Ref Composition]] |
| `Row notes/2022-12-08 - About AB test.md` | `76537d60433d20a60bcd1eafd06f2509b23a248153416c145f9fcdaaf500e999` | 2026-07-14 | imported | [[A-B Testing Rollout]] |
| `Row notes/2022-12-14 - Image lazy loading.md` | `52ce91c8a2acc53ed70df4fec607d9101d84896c3acb53d8828a39e26940edf7` | 2026-07-14 | imported | [[Web Rendering Best Practices]] |
| `Row notes/2023-01-09 - Omnilog.md` | `94837c17bf4787c84d706bdb382681617fd18f15a007c5fb485fb4877ff3640a` | 2026-07-14 | imported | [[Frontend Analytics Instrumentation]] |
| `Row notes/2023-04-19 - IntersectionObserver.md` | `a2d4bbece700eb8a86e7f327b206705946084a268a1b4da3ad2595d94bca1c07` | 2026-07-14 | imported | [[Intersection Observer]] |
| `Row notes/2023-05-25 - About Link.md` | `4038d11561b01f441d538f3e517e27951f72748f79f5436b319f72afec9b7fb7` | 2026-07-14 | imported | [[Web Rendering Best Practices]] |
| `Row notes/2023-08-07 - 08-06 Tech sharing.md` | `032e7c965a3a5ab478662a35b92c94b42fd89f20f77542d987a80608991b818f` | 2026-07-15 | imported | [[Houzz Marketplace]], [[Marketplace Growth Experiments]] |
| `Row notes/2023-08-31 - Houzz stack overflow (development node).md` | `dddbd62096b6ee3c01c3dec537624af7f37ff7f347c7522508ce53e0e68c81b5` | 2026-07-14 | imported | [[C2 Thrift Socket Troubleshooting]] |
| `Row notes/2023-09-08 - Image best practice.md` | `ceb215822cfd1e355ec26f1e437181ced3189c93e32552f238ce09209bc47bd1` | 2026-07-14 | imported | [[Web Rendering Best Practices]] |
| `Row notes/2023-11-24 - Prismic.md` | `6601ab503bf7c031f6fc8c027ff1e3f396674fb0ac1dbea06eb524f0e6ab81ff` | 2026-07-15 | imported | [[Jukwaa]], [[Prismic CMS Development]] |
| `Row notes/2023-12-21 - C2 development.md` | `31a401a9e6901a512e9a310d0cda6b8d0d14530f717dee8ae21804a73262b509` | 2026-07-15 | imported | [[C2]], [[C2 Development on Kubernetes]], [[Apache Thrift RPC]] |
| `Row notes/2024-08-14 - Shop houzz.md` | `85f8c07d848512330e9880cbc90fbe3fe871aced76964ee26469cacb24918f1d` | 2026-07-14 | reviewed-no-content | — |
| `Row notes/2024-11-08 - Coralogix.md` | `26fcdabfaaf2cbbf6559970f426feaa22df01c2da3c070fef572502619e3d98a` | 2026-07-14 | reviewed-no-content | — |
| `Row notes/2024-11-08 - Houzz email.md` | `68a1dc86bfc28548dd92c52b203e6d9fe42290e3b6d2193d9c69a54c111f7cb1` | 2026-07-14 | imported | [[Email Template Testing]] |
| `Row notes/2024-11-14 - Houzz Users Manager.md` | `6337ab3d2981069190f75700082bc3dd042c72d9169c1de3c6ca7d0c9eb3dd9e` | 2026-07-14 | imported | [[User Impersonation Safety]] |
| `Row notes/2024-11-15 - Redis.md` | `04357be03172bb18946b9bb84db53536918610a7cae1e6dd0ca87233611ffaa8` | 2026-07-14 | imported | [[Redis Debugging]] |
| `Row notes/2024-12-03 - Batch and Database.md` | `58b1b855b717da6e79fc94354054fe54dd245b209cd4823f3ef1686e0bdf7142` | 2026-07-15 | imported | [[C2]], [[Batch Jobs and Database Migrations]] |
| `Row notes/2024-12-16 - C2 Web module.md` | `96aa8b09e9c32dd42566442da97a80eb21e25b84f7730096f437f25035d246da` | 2026-07-15 | imported | [[C2]], [[Jukwaa]], [[C2-Jukwaa Web Module]] |
| `Row notes/2024-12-27 - Airflow knowledge sharing note.md` | `6ca877841f213e6852f16b750a0cd0e7ddc71966a47664cf7bc5cbb49e14e4e3` | 2026-07-14 | imported | [[Airflow Data Pipelines]] |
| `Row notes/2025-02-26 - Hugedeer ELB.md` | `e738dae399c7d70d446d21ac8430160a2874661caa576f1b2f5dcd04d1c89b63` | 2026-07-14 | imported | [[Service Routing and Load Balancing]] |
| `Row notes/2025-03-21 - Codepath.md` | `e834594539a34f0b83fb91f16044102e6df57f24c3f87ffdee3e301a5b9a98a9` | 2026-07-14 | imported | [[C2 Development on Kubernetes]] |
| `Row notes/2025-03-21 - Houzz L10n.md` | `f909d25110282bc22e296239d550f491301bd3096115de8f7d64b5eb1ca8d81b` | 2026-07-15 | imported | [[Jukwaa]], [[Houzz Localization Lang Files]] |
| `Row notes/2025-06-03 - Set new route.md` | `63f039976093b3cea67e2ecd5039469179af5e76a672fa2f638d7ee464b3549e` | 2026-07-14 | imported | [[Prismic CMS Development]], [[Service Routing and Load Balancing]] |
| `Row notes/2025-06-15 - Webhook.md` | `380eb5949da64fbb9da4bc02718dc54c01b127bd21d9f5fdd7a5efa60b359977` | 2026-07-14 | imported | [[Webhook Architecture]] |
| `Row notes/2025-10-09 - 3D Clipper.md` | `f9cb65645680d356ac5e6450d703bf5dfa57575b036c040e20a89548880dee7d` | 2026-07-15 | imported | [[Houzz Marketplace]], [[3D Product Conversion Workflow]] |
| `Row notes/2025-10-29 - Oauth.md` | `5807dfcb00d953caede0b096f5f296ed904c7ca3569ddf55afe9da66774531b6` | 2026-07-16 | imported | [[Expense App Authentication]], [[OAuth for Browser Apps]], [[Supabase]] |
| `Row notes/2025-12-10 - Converted model.md` | `c8a049cce6d20ef3045a824a08475516e2dcace6fa2160425e047756417e8f61` | 2026-07-15 | imported | [[Houzz Marketplace]], [[3D Product Conversion Workflow]] |
| `Row notes/2025-12-19 - Houzz K8S.md` | `e32a6b3474536598f6803b0f59fbc98e33863997c59ac534d6bbe53f9ccf77f1` | 2026-07-14 | imported | [[Kubernetes Architecture and Debugging]] |
| `Row notes/2026-01-09 - React query.md` | `3bb034d511b45699342aefee1283c371dece16a2af50ca228d6297fb29425842` | 2026-07-14 | imported | [[TanStack Query Server State]] |
