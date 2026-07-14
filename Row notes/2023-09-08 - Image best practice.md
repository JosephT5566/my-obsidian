---
title: "Image best practice"
date: 2023-09-08
tags: ["Work wiki"]
database: "Houzz Database"
source: "https://app.notion.com/p/ce5d4e5bae8c4d8e970a24ab5b904f55"
notion_id: "ce5d4e5bae8c4d8e970a24ab5b904f55"
---

# Image best practice
在 image 的使用上有許多眉眉角角，照片的上傳、URL的生成、設定的方法…
## Best practice
從 [Marketplace Image Rendering Guidelines](https://docs.google.com/document/d/11VaKtmguTIYV58RN4d7sGlatR5L0DqDWMwkUPIcrPZ4/edit#heading=h.ijyoorrgx9pf) 中擷取的
- Set the correct retina density for images
- Always set an image width/height in the viewport
- Use jpg over png, unless the image is primarily text
	- jpg 壓縮率比 png 好，但文字會變很醜，所以最好是圖片用 jpg，文字部分用 HTML/CSS 來處理
- Always render product images behind a white background
- Avoid adding image padding
- Always set alt text
- Have images URLs have a slug
## Upload images
我們目前有兩種方式可以上傳圖片：
1. ideabook
2. assetUpload
### Ideabook
透過 C2 method 處理，上傳到 s3 bucket ，並帶有一些編碼（external id）
### assetUpload
[Static Assets Upload](https://docs.google.com/document/d/1vdPzmfTkcg-xUBDqEt843-XEixBS-TEdg4t69Zllu1k/edit?usp=sharing)
透過 assetUpload ([prod](https://www.houzz.com/assetUpload)/[staging](https://www.stghouzz.com/assetUpload)) 直接上傳 static 檔案 ，檔案格式不限，甚至能傳 CSV，大小上限 30MB
> If you are using the assetUpload tool, it only uploads the image directly to the s3 bucket [s3://houzz-assets-production/](s3://houzz-assets-production/) **without** any process. Like [@tkyung](slackUser://houzz.slack.com/U0258GPJT)  said, you could upload the image through the upload page like project slideshow of your user account.<br>— Johnny Tseng ([thread](slackMessage://houzz.slack.com/C01TADHUSCX/1669790669.427059/1669790669.427059))
## Delete image
要刪除檔案的話，要到對應的 imageCache ([prod](https://www.houzz.com/imageCache)/[staging](https://www.stghouzz.com/imageCache)) 手動刪除 (To purge the URL from the caching layer.)
## simg and fimg
根據 [image rendering guidelines](https://docs.google.com/document/d/11VaKtmguTIYV58RN4d7sGlatR5L0DqDWMwkUPIcrPZ4/edit#heading=h.p93oquugrgnj) 中說：
We have two types of image rendering servers: simgs and fimgs
- simg (static image) - are older image rendering server that have an enumerated list of dimensions (stored in S3, uses one of the predefined sizes mostly used for high-quality images)
- fimgs (flexible image) - dynamically rendered images based on **parameters** set in the URL (generated on-the-fly by the image server from simgs)<br>從 simgs 取出的過程中生成的
其他一些特色：
- fimgs 可以再進行一些後處理動態生成我們想要的樣子，simgs 則是當初存多大就會取得多大
- fimgs 因為 url 有含 slug 所以有利於 SEO
- fimg 不支援 1200x1200 以上的大尺寸圖片
結論
- fimgs 用於 thumbnails
- simgs 用於高解析圖片
## Generate image URL
### By static path
```javascript
import ImageUtils from '@houzz/jukwaa-core/image';
ImageUtils.getLocalImageUrl('project/image.jpg');
```
```css
background-image: url('/jpics/project/image.jpg');
```
這兩種是直接用 static 路徑取得圖片，僅限於用 assetUpload 上傳的圖片
或是
```javascript
import C2Utils from '@houzz/jukwaa-core/utils/C2Utils';
C2Utils.getStaticResUrl('vt/designs/unified-upload/furniture.jpg')
```
### By external id
有透過 simgs server 的話就能取得一組 external id，透過 external id ，根據我們上面提到的需求，可以選擇要用 simgs 或是 fimgs 的 url
```javascript
// simgs
// 透過 utils.generateThumbnailUrl() 這個方法
import ImageUtils from '@houzz/jukwaa-core/image';
ImageUtils.getUserImageThumbnailUrl(...);
ImageUtils.getThumbnailUrl(...);
ImageUtils.getThumbnailUrlFromParams(...);
```
```javascript
// fimgs
// 透過 utils.generateDynamicThumbUrl() 這個方法
import ImageUtils from '@houzz/jukwaa-core/image';
ImageUtils.getUserDynamicThumbUrl(...);
ImageUtils.getImageUrlWithDimension(...)
```
Note: `getImageUrlWithDimension` 用 externalId 也是從 imageStore 拿，所以要先拿到存在 imageStore 才行。相較之下 getImagesByExternalIds 是 data fatcher，會是直接用 external id 拿image
## photo id, image id and external id
當我們用 idealbook 上傳照片後，可以獲得一個 image url ，後方編碼就是 **photo id**
我們可以用 **photo id** 在 imageManager ([prod](https://www.houzz.com/imagesManager)/[staging](https://www.stghouzz.com/imagesManager)) 搜尋到我們傳的圖片
搜尋後取得的資訊中，可以看到 **image id** 與 **external id**，點擊圖片也可以看到 url 是 simgs
不確定 image id 的用途，但 external id 是 amazon s3 service 中的 key
（external id 也可以用來在imageManager 查找，但 image id 則不行）
如同這張圖：[https://st.hzcdn.com/simgs/c56184f703d991e7_7-2483/home-design.jpg](https://st.hzcdn.com/simgs/c56184f703d991e7_7-2483/home-design.jpg)
- home-design.jpg is default appended
- 7 is thumb size
- c56xxx is external id (存在 s3 中的 key)
- 2383 is timestamp
## Space id
space id 指的是上面提到的 photo id，space id 只能透過 GQL 取得 external id
```javascript
query getPhotosBySpaceIds($spaceIds: [Int]) {
    photos: getPhotosByIds(ids: $spaceIds) {
        id
        type
        title
        image {
            externalId
            contentModified
            imageId
            width
            height
        }
    }
}
```
接著我們就有了 imageInfo，就能繼續用 getImageUrlWithDimension 拿到 url 了
## 解析度
假設我們有一個 100x100 的格子來放圖片
那麼理想的圖片會是 200x200，在更高的 2x 解析度會是 300x300
但在 fimg 生成圖片時，不可使用超過原始圖片的尺寸，超過的話會補上白色區塊
## References
[Image Service - FAQs](https://docs.google.com/document/d/1r4kQLNZKD735BIUM_n9YGgFYB8mw6x2n9OY7ZBfja7M/edit?usp=sharing)
[Image Knowledge transfer](https://docs.google.com/presentation/d/1LTyIH3npv8jcRFALsxxgJryQ25qIBivoJ6xtUCK8zXY/edit?usp=sharing)
