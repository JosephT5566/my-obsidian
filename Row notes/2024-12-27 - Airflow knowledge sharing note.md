---
title: "Airflow knowledge sharing note"
date: 2024-12-27
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/1696ed462a268039bac4c1edb24d9a4e"
notion_id: "1696ed462a268039bac4c1edb24d9a4e"
---

## Slide
[https://docs.google.com/presentation/d/17_mNRQR1Q9DyqJoJz8Wg_0cvkNRVKkG0h0q7k4r10bQ/edit#slide=id.p](https://docs.google.com/presentation/d/17_mNRQR1Q9DyqJoJz8Wg_0cvkNRVKkG0h0q7k4r10bQ/edit#slide=id.p)
## 分工
- DI: Data infra
	- 專注在資料庫本身或相關tools，如 AWS, Query engines, FTP, S3
- DE: Data Engineer
	- 專注在 data pipelines 的設置，如建立複雜的 pipeline 以定期執行等
- MP Eng
	- 我們，更專注在應用端，有 pipeline 的需求的話，就會去請求 DE，或是簡單的話也能自己完成
- DS: Data Scientist
## What is Airflow
目前我們在使用的是 Luigi，是 Spotify 建立的 pipeline 系統。本身是 Opensource，但我們已經很久沒更新，並也已經脫離社群版本太久
Airflow 是 Airbnb 的 pipeline 系統，目前依然有在維護，目標是將所有的 Luigi workflow 轉移到 Airflow 並 deprecate Luigi
Airflow 提供的 DAGs 可以視覺化完整的 pipeline 流程
## 開發過程中比較棘手的部分
Tom: Streaming MR 的過程中，可能會先要從 Awin 上面抓資料下來，存成 csv 後再轉給 DS 使用。<br>一般來說 Awin 會提供很多的參數，我會先把這些都整理成 csv 讓 DS 確認哪些需要，而這部分需要時間，DS 也要評估 Marketing 的使用狀況。再來我們完成程式碼後，會需要請 DI (Data infra/eng) 幫忙 deploy，這部分也需要時間，他們最近也在忙 migration，整個溝通合作過程最花時間
