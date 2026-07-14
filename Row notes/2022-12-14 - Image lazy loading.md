---
title: "Image lazy loading"
date: 2022-12-14
tags: ["Tech note"]
database: "Houzz Database"
source: "https://app.notion.com/p/ffea1da4f0ac4db0bdfd5b663313ed05"
notion_id: "ffea1da4f0ac4db0bdfd5b663313ed05"
---

# Image lazy loading
<unknown url="https://app.notion.com/p/ffea1da4f0ac4db0bdfd5b663313ed05#1c3c36cc994246f5a2bed967a09e6e2f" alt="bookmark"/>
image 必加 props:
- width: number → 撐開圖片載入完時所需的空間，避免載入完成時影響網頁的排版
- height: number → 撐開圖片載入完時所需的空間，避免載入完成時影響網頁的排版
	- 避免 CLS
		<unknown url="https://app.notion.com/p/ffea1da4f0ac4db0bdfd5b663313ed05#3f7bbb20a67d4cd295c91938a704e860" alt="bookmark"/>
	- All you need to do is add the images *native* width and height to the image.
		You can do this with the `width` and `height` attributes, defining the sizes in pixels.
		This then allows the browser calculate an aspect ratio (the ratio of the width to the height) and allocate enough space for the image before it loads.
		<unknown url="https://app.notion.com/p/ffea1da4f0ac4db0bdfd5b663313ed05#bd2c0d63ce804b159befcab571a2213f" alt="bookmark"/>
- loading: `“lazy”`
- alt: `“”`
