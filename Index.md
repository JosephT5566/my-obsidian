# Index

## Side Projects

- [[Side Projects]] — 個人專案的總覽，將產品脈絡、Architecture Decision 與可重用技術頁分開並交互連結。
  - [[Expense App]] — 支出管理 App 的功能、Authentication、UI 演進與 Receipt AI 子系統入口。
    - [[Expense App Authentication]] — Supabase Auth 負責登入與 session，GCF AI endpoint 透過 Supabase JWKS 驗證 Access Token 的責任分界。
    - [[Expense Receipt AI Pipeline]] — Expense App 的收據圖片 Upload、AI Analysis 與 Summary pipeline。
  - [[Travel Split App]] — 旅行費用分帳 App，以及 Cloudflare、Google OAuth、GAS／GCF 與 Google Sheets 的 authentication architecture。
    - [[Travel Split Backend Migration]] — 為解決 full-proxy latency，從 Cloudflare proxy + Workers KV + GAS 遷移到 token issuer + cookie + GCF verification。
  - [[Wedding Table Service]] — 婚禮現場多裝置報到 App 的目標、限制與技術選擇。
    - [[Firebase Realtime App Architecture]] — Wedding Table Service 採用 Firebase Hosting、Firestore、Authentication 與 Realtime listener 的架構決策。

## Houzz Engineering

- [[Houzz]] — Houzz 工作知識入口，依產品平台、Workflow／Decision 與可重用技術組織內容。
  - [[C2]] — Legacy web application 與 backend service platform，以及相關開發、batch、email 與 troubleshooting workflows。
    - [[C2 Development on Kubernetes]] — 使用 AWS SSO、EKS、Codepath pod 與 VS Code 調試 C2／C2Thrift。
      - [[C2 Thrift Socket Troubleshooting]] — 調查 C2 development environment 找不到 Thrift Unix socket 的檢查路徑。
    - [[Email Template Testing]] — 在 staging RQ debug pod 修改與發送 C2 order email 測試。
    - [[Batch Jobs and Database Migrations]] — 安全測試 C2／RQ batch job，並以版本化 SQL／AWX 執行 schema change。
    - [[User Impersonation Safety]] — 在 Houzz Users Manager 以最小風險 impersonate 使用者並安全解除。
  - [[Jukwaa]] — React／GraphQL frontend platform，以及與 C2、CMS、Localization 和 analytics 的整合入口。
    - [[C2-Jukwaa Web Module]] — Legacy C2 page 嵌入 Jukwaa React module 並串接 GraphQL／Thrift 的整合流程。
    - [[Prismic CMS Development]] — 在 Jukwaa 建立 Type、Doc、Slice、GraphQL fragment、component 與 route 的流程。
    - [[Houzz Localization Lang Files]] — 建立 Houzz lang file 並透過 Webpack 產生 `_id` 的流程。
  - [[Houzz Marketplace]] — Marketplace growth、content／SEO、catalog 與 3D product workflows 的產品領域入口。
    - [[Marketplace Growth Experiments]] — Marketplace differentiation、top-of-funnel、SEO、推薦與 email capture 專案。
    - [[3D Product Conversion Workflow]] — Catalog、ConvertedModel status、WebSocket、review flow 與 Pro Clipper 的產品流程。

## Algorithms

- [[Graph Traversal - DFS and BFS]] — 比較 DFS 與 BFS 的搜尋策略、常見題型、模板和複雜度取捨。
- [[Dynamic Programming]] — 說明 DP 的重疊子問題、最佳子結構、狀態定義與兩種實作方向。
- [[Trie]] — 介紹 Prefix tree 的資料結構、前綴查詢方式與 JavaScript 實作。

## AI and Career

- [[AI Agent Architecture]] — 整理 System prompt、Function calling、Agent loop、MCP 與 Skill 的角色。
- [[Interview Preparation with AI]] — 用 STAR Stories bank、JD 調整與 Transcript 回饋建立面試練習循環。
- [[Remote Codex with Self-hosted Runner]] — 透過 GitHub Actions self-hosted runner 遠端觸發本機 CLI 工作流程。

## Frontend Engineering

- [[React Ref Composition]] — 用 `forwardRef` 與 forked ref 同時支援 parent ref 和 component 內部 ref。
- [[Web Rendering Best Practices]] — 整理圖片尺寸、lazy loading、格式、alt text、圖片服務與 link attributes。
- [[Intersection Observer]] — 說明元素 visibility observer 的 root、rootMargin、threshold 與使用情境。
- [[Frontend Analytics Instrumentation]] — 以 Experience、Section、Container、Component 建立一致的 UI tracking taxonomy。
- [[TanStack Query Server State]] — 說明 query cache、staleTime、gcTime、persistence 與 auth lifecycle 的界線。

## Backend and Platform Engineering

- [[Apache Thrift RPC]] — 介紹跨語言 RPC contract、generated client/server code 與 Houzz GraphQL-to-C2 路徑。
- [[Redis Debugging]] — 從 staging 或 production debug pod 唯讀查詢 Redis Cluster。
- [[Airflow Data Pipelines]] — 說明 DAG workflow、Luigi migration 與 DI、DE、DS、應用工程的合作界面。
- [[Service Routing and Load Balancing]] — 整理 ELB、target group、traffic pool 與 application/infra route ownership。
- [[Webhook Architecture]] — 說明 event push、輕量前置層、n8n、驗證、重試與 idempotency。
- [[Kubernetes Architecture and Debugging]] — 串起 Cluster、Node Pool、Pod、Deployment、Service、DNS、rollout 與 debug。
- [[Google Cloud Functions]] — Travel Split 驗證 Cloudflare-issued credential、Expense App 驗證 Supabase JWT 並承接 AI operations 的 serverless backend。
- [[Google Cloud Storage]] — 以短效 Signed URL 讓 Client 直接上傳待分析收據圖片的 Object Storage。

## Security and Identity

- [[OAuth for Browser Apps]] — 區分 authentication、application session 與 delegated API authorization，並整理 Browser token models 與安全邊界。
- [[Supabase JWKS]] — 用公開 JWKS key 驗證 Supabase asymmetric JWT，避免讓驗證端持有 signing secret。

## Product and Experimentation

- [[A-B Testing Rollout]] — 說明穩定分桶、ramp up/down、metrics 與 experiment isolation。

## Data and System Design

- [[SQL vs NoSQL]] — 從資料關係、Access pattern、Transaction、Scale 與維護成本比較資料庫模型。
- [[Supabase]] — Expense App 使用的 PostgreSQL-based Backend as a Service，以及其 Auth、JWT 與資料存取邊界。
- [[System Design Foundations]] — 從單機擴展到 Load balancing、Cache、Queue、Replication、Statelessness 與 Sharding。
- [[Distributed Transactions - 2PC and Saga]] — 比較 2PC 與 Saga 的一致性、補償、協調方式與失敗處理。
- [[Back-of-the-envelope Estimation]] — 將 QPS、Peak、Storage、Latency 與 SLA 估算連回架構決策。
