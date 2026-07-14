# Index

## Algorithms

- [[Graph Traversal - DFS and BFS]] — 比較 DFS 與 BFS 的搜尋策略、常見題型、模板和複雜度取捨。
- [[Dynamic Programming]] — 說明 DP 的重疊子問題、最佳子結構、狀態定義與兩種實作方向。
- [[Trie]] — 介紹 Prefix tree 的資料結構、前綴查詢方式與 JavaScript 實作。

## AI and Career

- [[AI Agent Architecture]] — 整理 System prompt、Function calling、Agent loop、MCP 與 Skill 的角色。
- [[Interview Preparation with AI]] — 用 STAR Stories bank、JD 調整與 Transcript 回饋建立面試練習循環。
- [[Expense Receipt AI Pipeline]] — 記錄以 GCF、GCS Signed URL、JWKS 與 AI API 處理收據圖片的架構決策。
- [[Remote Codex with Self-hosted Runner]] — 透過 GitHub Actions self-hosted runner 遠端觸發本機 CLI 工作流程。

## Frontend Engineering

- [[React Ref Composition]] — 用 `forwardRef` 與 forked ref 同時支援 parent ref 和 component 內部 ref。
- [[Web Rendering Best Practices]] — 整理圖片尺寸、lazy loading、格式、alt text、圖片服務與 link attributes。
- [[Intersection Observer]] — 說明元素 visibility observer 的 root、rootMargin、threshold 與使用情境。
- [[Frontend Analytics Instrumentation]] — 以 Experience、Section、Container、Component 建立一致的 UI tracking taxonomy。
- [[Houzz Localization Lang Files]] — 記錄 lang file 必填欄位與透過 Webpack 產生 `_id` 的流程。
- [[TanStack Query Server State]] — 說明 query cache、staleTime、gcTime、persistence 與 auth lifecycle 的界線。

## Backend and Platform Engineering

- [[Apache Thrift RPC]] — 介紹跨語言 RPC contract、generated client/server code 與 Houzz GraphQL-to-C2 路徑。
- [[C2-Jukwaa Web Module]] — 說明 legacy C2 page 如何嵌入 Jukwaa React module 並串接 GraphQL/Thrift。
- [[Prismic CMS Development]] — 整理 Type、Doc、Slice、GraphQL fragment、component 與 route 的開發流程。
- [[C2 Development on Kubernetes]] — 使用 AWS SSO、EKS、Codepath pod 與 VS Code 調試 C2/C2Thrift。
- [[C2 Thrift Socket Troubleshooting]] — 調查 C2 development node 找不到 Thrift Unix socket 的檢查路徑。
- [[Email Template Testing]] — 在 staging RQ debug pod 修改與發送 C2 order email 測試。
- [[Redis Debugging]] — 從 staging 或 production debug pod 唯讀查詢 Redis Cluster。
- [[Batch Jobs and Database Migrations]] — 安全測試 RQ batch job 並以版本化 SQL/AWX 執行 schema change。
- [[Airflow Data Pipelines]] — 說明 DAG workflow、Luigi migration 與 DI、DE、DS、應用工程的合作界面。
- [[Service Routing and Load Balancing]] — 整理 ELB、target group、traffic pool 與 application/infra route ownership。
- [[Webhook Architecture]] — 說明 event push、輕量前置層、n8n、驗證、重試與 idempotency。
- [[Kubernetes Architecture and Debugging]] — 串起 Cluster、Node Pool、Pod、Deployment、Service、DNS、rollout 與 debug。

## Security and Identity

- [[OAuth for Browser Apps]] — 區分 ID/Access/Refresh Token，並整理 Google Identity、PKCE 與 OAuth proxy。
- [[User Impersonation Safety]] — 用最小風險 impersonate 使用者並確保完成後解除身分切換。

## Product and Experimentation

- [[A-B Testing Rollout]] — 說明穩定分桶、ramp up/down、metrics 與 experiment isolation。
- [[Marketplace Growth Experiments]] — 整理 marketplace differentiation、top-of-funnel、SEO、推薦與 email capture 專案。
- [[3D Product Conversion Workflow]] — 串起 Catalog、ConvertedModel status、WebSocket、review flow 與 Pro Clipper。

## Data and System Design

- [[SQL vs NoSQL]] — 從資料關係、Access pattern、Transaction、Scale 與維護成本比較資料庫模型。
- [[Firebase Realtime App Architecture]] — 記錄婚禮報到 App 選擇 Firebase Realtime stack 的背景、後果與 Security Rules。
- [[System Design Foundations]] — 從單機擴展到 Load balancing、Cache、Queue、Replication、Statelessness 與 Sharding。
- [[Distributed Transactions - 2PC and Saga]] — 比較 2PC 與 Saga 的一致性、補償、協調方式與失敗處理。
- [[Back-of-the-envelope Estimation]] — 將 QPS、Peak、Storage、Latency 與 SLA 估算連回架構決策。
