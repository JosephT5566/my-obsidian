---
type: concept
status: growing
topics: [houzz, engineering, system-navigation, architecture]
created: "2026-07-15"
---

# Houzz

## Summary

這是 Houzz 工作知識的導覽頁。內容依「產品或平台 → Workflow／Decision → 可重用技術」整理，讓內部系統脈絡與通用工程知識可以分開維護、互相連結。

## Systems and Product Areas

### [[C2]]

Legacy web application 與 backend service platform。相關內容包含 [[C2 Development on Kubernetes]]、[[Apache Thrift RPC]]、[[Batch Jobs and Database Migrations]]、[[Email Template Testing]] 與 [[C2 Thrift Socket Troubleshooting]]。

### [[Jukwaa]]

以 React 與 GraphQL 為主的 frontend application platform。它能透過 [[C2-Jukwaa Web Module]] 嵌入 C2 page，並使用 [[Prismic CMS Development]]、[[Houzz Localization Lang Files]]、[[Frontend Analytics Instrumentation]] 等 workflows。

### [[Houzz Marketplace]]

Marketplace product 與 growth work 的入口，串起 [[Marketplace Growth Experiments]]、[[A-B Testing Rollout]]、[[3D Product Conversion Workflow]] 與相關 frontend/data technologies。

## Shared Engineering Capabilities

- **Runtime and infrastructure**：[[Kubernetes Architecture and Debugging]]、[[Service Routing and Load Balancing]]、[[Redis Debugging]]。
- **Service communication**：[[Apache Thrift RPC]] 與 GraphQL-to-C2 request path。
- **Data operations**：[[Airflow Data Pipelines]]、[[Batch Jobs and Database Migrations]]。
- **Frontend engineering**：[[Web Rendering Best Practices]]、[[Intersection Observer]]、[[React Ref Composition]]、[[TanStack Query Server State]]。
- **Operational safety**：[[User Impersonation Safety]] 與 production read-only investigation。

## Related

- [[C2]]
- [[Jukwaa]]
- [[Houzz Marketplace]]
- [[System Design Foundations]]
