---
title: "IntersectionObserver"
date: 2023-04-19
tags: ["Tech note"]
database: "Houzz Database"
source: "https://app.notion.com/p/45f12843e89343e88888d33153889230"
notion_id: "45f12843e89343e88888d33153889230"
---

<unknown url="https://app.notion.com/p/45f12843e89343e88888d33153889230#f611db5d89494fdb9032e5bd474cea42" alt="bookmark"/>
`IntersectionObserver` 幫我們觀察的是元素的「相交（intersect）」變動，也就是元素與指定可視窗口的「相交與否」發生變動時觸發。
簡單來說就是頁面元素因捲動而進入到可視範圍中，或是離開了可視範圍時，`IntersectionObserver` 就會執行指定任務，所以我們可以利用它來偵測「某個元素是不是進入視窗中」了，而且還可以調整許多細微的偵測設定，相當強大。
和其他「觀察者」一樣，`IntersectionObserver` 為一個建構函示，需要使用 `new` 關鍵字來創建實體，並且需要傳入 Callback Function 作為參數，該 Callback 會獲得一個存放 IntersectionObserverEntry 的陣列以及「觀察者（observer）」自身實體
另外，IntersectionObserver 除了 Callback 之外還有一個可選的 `options` 參數可以設定：
- **root**： 這個屬性將決定要以哪個元素的可視窗口作為觀察依據，預設為 `null`，表示以 Viewport 作為判斷依據，也可以設定成其他元素。
- **rootMargin**： 這個屬性決定的是窗口的縮放，設定規則和 CSS 的 `margin`，可以給定一個值，也可以四邊各自設定，正值為外擴，負值為內縮。
- **threshold**： 這個屬性是設定觸發的**比例門檻**，當目標元素與可視範圍的相交範圍「經過」了這道門檻，Callback 就會被觸發，舉例來說：
	- 預設值 `0`： 當相交範圍的比例「開始大於 0%」或「開始小於 0%」 的瞬間會觸發。
	- 設定為 `1`： 當相交範圍的比例「開始大於 100%」或「開始小於 100%」 的瞬間會觸發。
	- 設定成陣列 `[0, 0.5, 1]`： 規則如上，但目標元素就會有三個觸發時機。
