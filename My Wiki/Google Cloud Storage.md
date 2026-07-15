---
type: tool-or-library
status: active
topics: [gcp, object-storage, signed-url, image-upload]
created: "2026-07-15"
---

# Google Cloud Storage

## Summary

Google Cloud Storage（GCS）是 Google Cloud 的 Object Storage。[[Expense App]] 在 [[Expense Receipt AI Pipeline]] 中用它暫存待分析的收據圖片，讓 Client 能直接上傳，不需讓圖片流量穿過 App backend 或 [[Google Cloud Functions|GCF]]。

## Use Cases

- 保存圖片、影片、備份或其他 Object。
- 透過短效 Signed URL 讓未持有 Cloud credential 的 Client 執行受限上傳或下載。
- 作為 AI API 可讀取的暫存圖片來源。

## Configuration

Expense Receipt flow 會先讓 [[Google Cloud Functions]] 建立約 10 分鐘有效的 Signed URL。Function 需要知道 Bucket name，並由對應 Service Account 取得簽署與必要的 Storage 權限。Client 準備 File name 與 Content type 後，以 `PUT` 將檔案放進 request body。

上傳授權應限制 Object path、Content type、Size 與有效時間；暫存圖片則應設定 Lifecycle deletion，避免資料與成本持續累積。

## Strengths

- 圖片不經 App server，可降低流量、記憶體用量與系統耦合。
- Signed URL 將操作、Object 與期限縮小到單次需求。
- Bucket lifecycle 能自動清除暫存資料。

## Limitations

- Signed URL 本身在到期前是 bearer credential，外洩後可能被他人使用。
- CORS、IAM、Content type 或簽章設定錯誤都可能導致 Client 上傳失敗。
- Object lifecycle、存取紀錄與敏感資料政策仍需另外設定。

## Related

- [[Expense Receipt AI Pipeline]]
- [[Google Cloud Functions]]
- [[System Design Foundations]]

## References

- [[2026-05-10 - 5-10 weekly updates]]
