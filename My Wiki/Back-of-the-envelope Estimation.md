---
type: how-to
status: growing
topics: [system-design, estimation, capacity-planning, interview]
created: "2026-07-14"
---

# Back-of-the-envelope Estimation

## Goal

用數量級估算將流量、延遲、儲存與可用性需求轉成架構決策。

## Prerequisites

- 基本單位換算：KB、MB、GB、TB、PB
- 每日秒數約為 86,400
- 對產品的 DAU、操作頻率、資料大小與 Retention period 做明確假設

## Steps

1. **寫下假設**：使用者數、每日操作次數、讀寫比例、每筆大小與保存期限。
2. **估算平均流量**：`daily requests ÷ 86,400 = average QPS`。
3. **估算尖峰流量**：以產品特性選擇 Peak multiplier，不要只沿用固定倍數。
4. **拆分操作**：分別估算 Read、Write、Search、Upload 與 Background job。
5. **估算儲存**：`records/day × bytes/record × retention days`。
6. **加入額外容量**：Replication、Backup、Index、Thumbnail、Transcoding 與安全餘裕。
7. **檢查延遲階層**：CPU cache、RAM、SSD、Datacenter network、Cross-region network。
8. **換算 SLA**：
   - 99.9% 約為每年 8.76 小時停機。
   - 99.99% 約為每年 52.6 分鐘停機。
   - 99.999% 約為每年 5.26 分鐘停機。
9. **連回架構決策**：說明估算結果為何需要 Cache、CDN、Queue、Sharding 或 Object storage。

## Verification

- 每一行計算都有單位。
- 結果的數量級合理，沒有混用每日與每秒。
- 同時提供 Average 與 Peak。
- 架構元件可以追溯到某個容量或可靠性需求。

## Troubleshooting

- 缺少產品數字：使用範圍或保守假設，並清楚標記。
- 心算太複雜：用 `100,000 ÷ 10` 近似 `99,987 ÷ 9.1`。
- 得到龐大儲存量：確認是否重複加入 Retention，並分開 Raw data 與 Replicated capacity。
- SLA 看似不可能：檢查所有串聯依賴，因為整體可用性會低於單一元件。

## Related

- [[System Design Foundations]]
- [[SQL vs NoSQL]]

## References

- [[2026-06-14 - 6-14 weekly update]]
- [ByteByteGo: Back-of-the-envelope Estimation](https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation)

