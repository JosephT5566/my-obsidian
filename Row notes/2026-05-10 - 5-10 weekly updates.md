---
title: "5/10 weekly updates"
date: 2026-05-10
tags: ["Interview 2026"]
source: "https://app.notion.com/p/35f6ed462a268027a48ddfa315b5d732"
notion_id: "35f6ed462a268027a48ddfa315b5d732"
---

## Travel split
**把 GAS 的後端遷移到 GCF**
第一次開通 Google cloud function ，AI 分析是這樣的流量不會花到錢。
變成 GCF 做為後端，然後原本的 clf proxy 調整為 token issuer
全部都透過 HttpOnly, secure 來管理 token cookie，所以也把 [localhost](http://localhost) 開發調整成 same site
## Expense App
**Improve design**
改成使用 Shadcn lib 來提供 shared component。
本來是打算用 Shadcn Drawer 來改善 UX，但後來發現太大的 form 不太適合 Drawer，於是就改用 Shadcn Dialog。
System style 同時也是被 Shadcn 調整到了，所以花了一點時間調整
**完成 Search page**
這部分就蠻快的，先用 Shadcn 的 Date range selector 改善了日期選擇，reuse 既有的 expense row component 顯示搜尋結果，query 的部分則是已經準備好了，使用 `ILIKE` 進行資料庫搜尋。
**嘗試加入 AI 圖片分析**
跟 Gemini 討論後覺得可行，需要準備
- 另一個 GCF for api
	- 不擴充沿用先前 travel split 開的 GCF，一方面是用途不同所以分開管理，另一方面是配置的硬體也不太同
	- 另一個是考慮 travel split 的帳戶是一般帳戶，而另一個帳戶是有 Google pro，可能有多一點 token 可以用
- GCS (google cloud storage)
	- 做為暫存要分析的圖片，一方面能降低呼叫 GCF 使用的流量，同時也提供 image url 給 ai studio 使用
- Google ai studio
	- API key
互動方式：
1. app → GCF → GCS: 發行 GCS 的 signed url
	- App 跟 GCF 之間的認證都是單純透過 supabase 的 JWT。但發現本來的 **legacy JWT** 是**對稱**加密，還需要把 secret key 設給 GCF 來使用。現在出了新的 **JWKS** (JSON web key set) **非對稱**加密，我們可以直接使用 supabase 提供的 url 來取得公鑰，並進行 JWT 認證。
	- GCS 上傳資料的方式是透過使用 signed url，其有效期限為 10 分鐘，這部分就是需要透過指定的 GCF 來發行。
		- 先把 GCF 的對應帳號加到 GCS 的允許存取名單
		- 提供 GCS 的 bucket name 給 GCF
		- GCF 的 IAM 權限開通 `service account token creator` and `storage bucket viewer` （才能讀到 bucket image）
		- 準備要上傳的 file name 跟 file type
2. App → GCS
	- 這部分就是直接使用上一步取得的 signed url，使用 `PUT` 然後將檔案放在 body 送出就好。上傳完成也會回完成的 uploaded image url
3. App → GCF ↔ ai studio
	- 我們會先需要打開 google ai studio 取得 api token，然後再存到 GCF 提供注入
	- 在 GCF 裡規劃好 prompt，等待 app 呼叫 api，body 會帶上先前得到的 url，然後一起呼叫 ai api，ai 完成執行再回傳 json object string。

