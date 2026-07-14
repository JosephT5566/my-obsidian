---
title: "Set new route"
date: 2025-06-03
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/2076ed462a2680fca4d5f72fcee14823"
notion_id: "2076ed462a2680fca4d5f72fcee14823"
---

# Description
Superman contests page, 我們需要設定 url 給 prismic-cms，需要動到哪些設定？
vote slice url: `houzz/contests/fortress-of-solitude-vote`
landing page url:
# Prismic cms
（Jukwaa 也同理）
1. 我們需要設定對應的 apps 中的 AppConfig
# Local dev
根據 <mention-page url="https://app.notion.com/p/d304ce1046ba497a8938bb14384a1e98"/>
> 更新 c2ubuntu ，c2ubuntu 中的 `/dev_environment/nginx-houzz-docker/configs/https/mp_web_apps.conf` 會決定哪些 url 要連線到哪個 service
如果要在 local 開啟 /contests，我們需要在 `/dev_environment/nginx-houzz-docker/configs/https/prismic_cms.conf.tmpl` 加上
```javascript
# pro contest
location ~* ^/contests(/.*)?$ {
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header Host $host;
    proxy_pass https://${NODE_ADDRESS}:4014;
}
```
並在 c2ubuntu 下執行 `./dev_environment/start_local_dev.sh` ([Prismic CMS local setup](https://docs.google.com/document/d/1b5s__yq3ttWN1XVpF5MG0Ytlez7FbUmcadNJWCeI78c/edit?tab=t.0#heading=h.2xjoljyohm74))
後續在跑完 server-hot-reload 就可以在 local 連到了
（**不加也沒差**，就是上到 staging 再來測）
如 `/houzz-cms/prismic-cms/apps/cms/AppConfig.js` 中的 `CMSMarketplaceDepartmentLanding` 就不在 c2ubuntu 上，所以就都直接上到 staging
```javascript
CMSMarketplaceDepartmentLanding: {
    mapping: [
        '/products/furniture',
        '/products/kitchen-and-dining',
        '/products/bath-products',
```
# Staging
- how to add new route: [mp-growth-tw thread](slackMessage://houzz.slack.com/C05G5UCEFFZ/1731680981.415869/1731680972.888879)
- example ticket [https://houzz.atlassian.net/browse/MPG-2500](https://houzz.atlassian.net/browse/MPG-2500)
- Load Balancer Change Request [Wiki](https://engwiki.houzz.tools/doc/load-balancer-change-request-developers-bPm41cAIOS)
	1. Apply the change to staging main LB and verify if the new url works as expected ([link](https://engwiki.houzz.tools/doc/load-balancer-change-request-for-routing-different-paths-bPm41cAIOS#h-apply-the-change-to-staging-main-lb-and-verify-if-the-new-url-works-as-expected-1))
	2. **awx-playbooks** is not necessary now. (only kube-atlas PR is needed)
