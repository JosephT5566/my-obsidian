---
type: concept
status: growing
topics: [houzz, airflow, data-pipeline, dag]
created: "2026-07-14"
---

# Airflow Data Pipelines

## Summary

Apache Airflow 用 Directed Acyclic Graph（DAG）描述有依賴關係的資料工作，提供 scheduling、retry、觀測與流程視覺化。[[Houzz]] 的方向是把長期偏離 upstream 的 Luigi workflows 遷移到仍活躍維護的 Airflow；單純的 application batch 工作則可先參考 [[Batch Jobs and Database Migrations]]。

## How It Works

- Data Infrastructure（DI）負責 database、AWS、query engine、FTP、S3 等平台能力。
- Data Engineer（DE）設計與部署複雜或週期性的 pipelines。
- Application engineers 定義產品需求，簡單 pipeline 可自行完成，複雜流程與 DE 合作。
- Data Scientist（DS）確認資料欄位、品質與分析用途。

Pipeline 的主要難點常在跨團隊界面：例如先從第三方取得資料、整理成 CSV、與 DS 確認欄位，再由 DI/DE 安排 deployment。技術實作之外，要預留 schema agreement、data validation 與協作 lead time。

## Related

- [[Batch Jobs and Database Migrations]]
- [[Distributed Transactions - 2PC and Saga]]
- [[Houzz]]

## References

- `Row notes/2024-12-27 - Airflow knowledge sharing note.md`
