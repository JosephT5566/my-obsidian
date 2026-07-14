---
title: "About Link"
date: 2023-05-25
tags: ["Tech note"]
database: "Houzz Database"
source: "https://app.notion.com/p/4e139a663a984f7882f5e1de24a360a7"
notion_id: "4e139a663a984f7882f5e1de24a360a7"
---

# About Link
required props
- rel: `{null}`
	- Regarding the SEO team said, the page will navigate to the internal page. so we don't need `nofollow` and `noopener` ([Thread](slackMessage://houzz.slack.com/C04UFSN8RN3/1683621445.390389/1683593497.256199))
	- houzz-ui 會內建 rel=`{’nofollow nooper’}` ，所以需要自行設定 `{null}`
optional props:
- target: `"_blank”` → open on new tab
