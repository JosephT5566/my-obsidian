---
title: "Houzz email"
date: 2024-11-08
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/1376ed462a268086b193c199728765a5"
notion_id: "1376ed462a268086b193c199728765a5"
---

# Guide
[Outline](https://engwiki.houzz.tools/doc/creating-and-testing-a-new-email-492msvEX54)
# Related code
php (for prod/stg):
[https://github.com/Houzz/c2/blob/master/web/email/OrderEPNEmail.php#L378-L389](https://www.google.com/url?q=https://github.com/Houzz/c2/blob/master/web/email/OrderEPNEmail.php%23L378-L389&sa=D&source=docs&ust=1731003202518973&usg=AOvVaw3qRTe3Uz0oTDutIPlDO43-)
mock data (for local testing):
[https://github.com/Houzz/c2/blob/058cf55bc279d032684d1d97e0ec9239ff9852a8/email-new/data/OrderConfirmationEmail.json#L371-L392](https://www.google.com/url?q=https://github.com/Houzz/c2/blob/058cf55bc279d032684d1d97e0ec9239ff9852a8/email-new/data/OrderConfirmationEmail.json%23L371-L392&sa=D&source=docs&ust=1731003202519006&usg=AOvVaw3Ut7_wo5pTIPgl8CgKeNYd)
# How to send email
## 1. RQ pod
1. 到 [https://rq.stghouzz.com/mp-batch-general-v2#readyJobs](https://rq.stghouzz.com/mp-batch-general-v2#readyJobs) 點擊 Debug pod button，然後 create Debug pod。他會自動建，好了會出現 pod name, like `rq-mp-general-2024-11-07-joseph.tseng-btxc4`
2. 到 vscode，我們需要切換 cluster，如同之前<mention-page url="https://app.notion.com/p/31c28ed845a84be08883e44c63dd5770"/> <br>ter 輸入 `tsh kube login stg-batch-eks` 就能切到 teleport.ivyco.net-stg-batch-eks cluster
3. Namespace 切到 `backend`
4. 一樣到 Workloads/Pods 裡找我們剛剛開的 pod name → right click → attach visual studio code → `rq worker container`
5. 進到 /home/clipu/c2 然後到我們改過的檔案，以這[PR](https://github.com/Houzz/c2/pull/39309/files)為例，我們會到 `web/email/OrderEPNEmail.php`
6. 修改完檔案後，再到 /batch 去執行裡面的 `/testOrderEmails.php` script: <br>其中的 case name, stg user id, email 都可以換
	```bash
php testOrderEmails.php -t order_delayed_new_eta -u 75914600 -e joseph.tseng@houzz.com -a 1773-1882-1035-1828 -n HOZ083609

php testOrderEmails.php -t order_delayed_new_eta -u 75606384 -e jing.su@houzz.com -a 1773-1882-1035-1828 -n HOZ083609
	```
Note: OrderEPNEmail 是所有 email template 的 parent，如果有改動，可以所有的 cases 都送一次
## 2. Manage Order
這部分就跟 RQ 的做法不一樣\\
進到 [https://www.stghouzz.com/manageOrder/1773-1882-1035-1828](https://www.stghouzz.com/manageOrder/1773-1882-1035-1828)
再從最下面 `Send Order Email` Button
