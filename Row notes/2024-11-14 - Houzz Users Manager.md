---
title: "Houzz Users Manager"
date: 2024-11-14
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/13e6ed462a2680c691d2c17628cce989"
notion_id: "13e6ed462a2680c691d2c17628cce989"
---

# User Manager tools
- Stg: [https://www.stghouzz.com/usersManager](https://www.stghouzz.com/usersManager)
- Prod: [https://www.houzz.com/usersManager](https://www.houzz.com/usersManager)
輸入 user name，可以查詢各個 user 的狀態
# Impersonate
可以切換成其他人的帳號登入，之前在查 seller central 的時候可以用 ([thread](slackMessage://houzz.slack.com/C05G5UCEFFZ/1723459650.986299/1723459650.986299))
***How to impersonate:***
- login to [https://www.houzz.com/usersManager](https://www.houzz.com/usersManager)
- type the user-id
- Clicks “impersonate” on the actions toolbar
- Clicks “Sign In As User” at the bottom
- If you want to use the same auth cookie on local, may use Go Jukwaa! to copy/paste
or
- go to any user profile with __private suffix, like [https://www.houzz.com/user/](https://www.houzz.com/user/)[\<username\>/__private](https://www.houzz.com/user/%3Cusername%3E/__private)
- Click the “impersonate this user” button on the admin toolbar
***How to un-impersonate:***
- clicks the “Undo impersonation” on the top-right dropdown
- always REMEMBER to un-impersonate the user after debugging!!!
- **DO NOT** do any action on the user account, just reviewing the page is fine

by [@Tom Huang](slackUser://houzz.slack.com/U02ADLYPZ29) 大大
1. 目前 Staging 上不能用 impersonate，但 Houzz2 可以
2. 另一種方法可以登進staging：在 [usersManager](https://www.stghouzz.com/usersManager) 把某個 seller 的email 改成自己的，就能直接用 email/pwd 登入
	1. 可用的 seller name: lightingnewyork
	2. email example: [joseph.tseng+stgtest@houzz.com](mailto:joseph.tseng+stgtest@houzz.com)，再用忘記密碼來 update pwd
3. 或是直接用 set password 改密碼
4. 只有 3rd party 的 seller 才會有 seller central

其他可嘗試的 user id:
Here's a list of test staging and pro accounts for future reference [https://houzz.slack.com/lists/T0258GPEP/F08178AHDPE](https://houzz.slack.com/lists/T0258GPEP/F08178AHDPE)
seller
- gdf studio
Pro user
- webuser_824671075
	- origin email: protes@houzz.com
	- password: protest1234
## Sign in Cookies
根據 dev tool 的 [code](https://github.com/Houzz/houzz-chrome-tools/blob/master/public/background/contextMenus.js#L60-L76)，在登入時會複製這些 cookies
```plain text
[ 'h0', 'h1', 'h2', 'h3', 'h4', 'h5', 'v', 'hzt', 'hzua1', 'hzua2', 'ivy_jukwaa_key', '_ivy_session_key', 'abts', 'ivydbex', 'cak' ]
```
後續透過 Chrome extension: **Cookie-Editor** 的 export 可以得到 cookies 的 array，再寫下簡單的 JS
```javascript
// 過濾出以上指定的 cookies 並調整 domain
const keys = [ 'h0', 'h1', 'h2', 'h3', 'h4', 'h5', 'v', 'hzt', 'hzua1', 'hzua2', 'ivy_jukwaa_key', '_ivy_session_key', 'abts', 'ivydbex', 'cak' ];
stgcookies.filter(c => keys.includes(c.name)).map(c => ({...c, domain: ".www.houzzdev.com"}))
```
然後同樣用 **Cookie-Editor** 的 import 就能匯入到 local dev 了
# Trade pro
在解這個 **Remove Request Quote for Trade Pros **[ticket](https://houzz.atlassian.net/browse/MPG-2228) 的時候遇到的，在 wish list 時，somehow trade pro 會使用 jukwaa page，一般 user 則是使用 c2 page
- trade pro account:  [https://engwiki.houzz.tools/doc/trade-testing-Yqh8gJKJSS](https://engwiki.houzz.tools/doc/trade-testing-Yqh8gJKJSS)
