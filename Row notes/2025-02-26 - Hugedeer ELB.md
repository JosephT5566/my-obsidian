---
title: "Hugedeer ELB"
date: 2025-02-26
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/1a66ed462a2680c09e2aefefcfc6f2d3"
notion_id: "1a66ed462a2680c09e2aefefcfc6f2d3"
---

# Purpose
Yifan 說，hugedeer service 針對一些明確的類別, 會有獨立的 ELB, 方便 debug<br>e.g. product: [https://github.com/Houzz/c2/blob/b6d1da46ceb643d21b573e2cd98ceb66729ab823/config/C2ConfigHouzz2.php/#L4339](https://github.com/Houzz/c2/blob/b6d1da46ceb643d21b573e2cd98ceb66729ab823/config/C2ConfigHouzz2.php/#L4339)
因此希望針對 Search 可以打到不同的 ELB
# ELB
ELB: Elastic Load Balancer
**主要負責流量分發**，它通常會放在 API 服務的入口，確保請求能夠正確地轉發到後端 Server Group
Hugedeer 的 ELB 是由 AWS 的 load balancer 生成，再導到後續的 EC2，看起來是採用手動管理。
流程步驟：
1. 使用者請求
2. 經過 c2 service / spinnker service 等 API 服務的處理
3. \[ ELB（AWS獨立管理）\]
4. \[ Target Group（記錄後端目標）\]
5. \[ EC2 / Kubernetes Pods（處理 GraphQL / Thrift 請求）\]

BTW，目前 Houzz 的服務，ELB 的使用分配是由 spinnker 來管理的 (By Tom Lin)
以 `pro-ai-assistant-service` 為例，存在著這些的 load balancers（不是所有的 service 都有配置 load balancer）
![](https://prod-files-secure.s3.us-west-2.amazonaws.com/300ece61-c38a-44db-b7d8-e63f52d8b52e/33e2c609-2cc9-4856-b1de-94bf6323614b/%E6%88%AA%E5%9C%96_2025-02-26_%E6%99%9A%E4%B8%8A7.05.09.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB466WGBPGFDD%2F20260714%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260714T080834Z&X-Amz-Expires=300&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEEgaCXVzLXdlc3QtMiJHMEUCIQC6UwpAnukVHgecvnc2tjwS7hCD9o8m1xLgP3p47I8P8QIgVT9UwLBd81RF63aJCm3UTUyUIuJKrXW9BakcmRDx1SMq%2FwMIERAAGgw2Mzc0MjMxODM4MDUiDEN8RSdlGkNdlx5VzyrcA772RrtvgufhD%2FvKOLDBw%2F%2FsfN2Pqj7yWXOawtfmg77mkYln%2FGJBEwV5eFu0%2BKr2Jv8W0bISi0drDL7Op3sk%2FFrgChhfi8d8ZYYpv9rjJPDEZy%2FJDrxU6N2iTPqLJFDgzJ%2BKIL9iBzcQOtp3KX5qlMHVin5N5mg%2FNDhFWJrWRqu4Mw%2BIy1bcveQZm3mlONXr9bVAstXJZNb%2B0U5D0GWHyHPAhmcTN5JzQ5obdCdWs4mUXx87NDt3jFPZdE9zy0vR3uT%2Bw7Ti4PlHXrHfkFxokGzILlRxD72sO2whPhNI5dhCdPfzTBVbwI4QumRPrlmVEyGbCthc%2Fs2kpDISC1dfgTO51c%2BJA40%2BKEZN0bQmMoosrbFSlWJuBClOTu%2BkOpHxITFQZLLzHlq7GwjpBL%2FgEdt1ZomZuNuUmAhePxnz8Z82yF2d8Ay%2BufRBpBJS%2FH6zrXQA%2BDavt5sFJ8GyLeoA0n7GQ%2Bhr0dCjcSVKmxfUTkpNR0GIJ2MbE0bugoefGfinu%2BP6rf7XVo4cuHlQmJe6HCGuZ2JTKdoFCtcsFGnv26D3rwQu%2F8ms%2F0Yg0hwBn92wDm%2BZ3pEb3duYFYkm0Un2b4cX5u1aq8AqDtq%2BExONfNLvm85fxH5I0RaOFZqjMObK19IGOqUBI7MxC%2FGQmAbmiK1AfecIX%2FxvQeb2zRbN64a%2F5iM3cEBlckIuDMAwC%2FFU4uvj6Kh2ukWttkbq9ykm%2FaqukO8dA2hCL%2BFD%2BNcWwf2noNH1Lrtzhy4%2BUJSPHlN6Cib7jD9163eYyhYqffshSdpfKFNavg9mmKuSZxh%2BZ%2FbMCWnks7PGxAeu8mJuw9GcKAuelmVFX1UtXj1DBfr%2F6rh8g89ua%2FTShi0E&X-Amz-Signature=47a616261c9fa3b343998c665691ad88fbf9d77ab4e9fc653c7a4a57b675d41e&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
# Solutions
## C2
如上面提到的例子 [https://github.com/Houzz/c2/blob/b6d1da46ceb643d21b573e2cd98ceb66729ab823/config/C2ConfigHouzz2.php/#L4339](https://github.com/Houzz/c2/blob/b6d1da46ceb643d21b573e2cd98ceb66729ab823/config/C2ConfigHouzz2.php/#L4339)
C2 service (hugedeer_wrapper) 中會根據不同的 pool 選擇對應的 ELB endpoint（雖然看起來 dev, staging, houzz2 用的都是**一樣**的 ELB，但 prod 還是有區分）
這樣做法就會是：
1. 開一個新的 api 到 hugedeer_wrapper
2. 新增新的 ELB endpoint 到對應的 pool
3. 修改 Jukwaa 的 GQL resolver 改成使用 hugedeer_wrapper
## GQL
### 想法一：新的 service
發現 jukwaa/config-override/service 中有定義 Hugedeer service 使用的 endpoint。
或許我們可以再開一個新的 service `HugedeerQueryIntent`，然後給對應的 ELB endpoint 讓 service 去使用
但這麼做好像怪怪的，因為 ELB 就是在 reverse proxy 中被實現 ([ref](https://www.cloudflare.com/zh-tw/learning/cdn/glossary/reverse-proxy/))，應該被包在服務裡面被分配，而不是直接露出來被使用
另外就是再開一個 service 有點麻煩
1. 我們需要在 c2svc/src/go/src/houzz 中開一個新的資料夾，以及相關 handler (eg: hugedeer 的 [service_handler.go](https://github.com/Houzz/c2svc/blob/master/src/go/src/houzz/hugedeer/service/service_handler.go))
2. 接著要再到 c2thrift/definitions/backends 新增對應的介面接口 (eg: hugedeer 的 service [thrift](https://github.com/Houzz/c2thrift/blob/master/definitions/backends/hugedeer.thrift))
3. 在 jukwaa graph 中實作 resolver，對應的 schema，service registry 等等

### 想法二：在 Hugedeer service 中實現 pool 分配 ELB
從 AWS staging 的 load balancers 中去找，發現在 c2svc 的 [service-dashboard](https://github.com/Houzz/c2svc/blob/master/src/java/service-dashboard/src/main/resources/prod/service-dashboard.yml#L24-L36) 有列出不同的 ELB endpoints 並對應不同的 key
# Reference
- Graphouzz guide: [https://engwiki.houzz.tools/doc/graphouzz-yYKlWoYqGX](https://engwiki.houzz.tools/doc/graphouzz-yYKlWoYqGX)
	- 裡面提到了 jukwaa 中的 graph 架構，如何設置 schema, resolver, service registry 以及 directives, decorators 語法等等
