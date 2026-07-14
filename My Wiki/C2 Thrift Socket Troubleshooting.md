---
type: troubleshooting
status: growing
technology: C2 Thrift
created: "2026-07-14"
---

# C2 Thrift Socket Troubleshooting

## Symptoms

Development node 顯示 `fsockopen(): Unable to connect to unix:///opt/sock/hzsock/sl.sock`，指出預期的 Unix socket 不存在。

## Environment

- Runtime: C2 PHP / Thrift `TUNIXSocket`
- Relevant configuration: development node 的 local service/socket provisioning

## Investigation

1. 確認 error path 與實際 filesystem socket 是否一致。
2. 確認負責建立 socket 的 service/process 是否啟動。
3. 檢查目前 attach 的 container、cluster 與 codepath 是否正確。
4. 對照 C2/Thrift configuration，確認此環境應使用 Unix socket 還是 network endpoint。

## Root Cause

目前來源只記錄症狀與內部討論連結，沒有足夠證據確認單一 root cause；最直接的含義是 client 指向的 socket file 未建立或不可見。

## Verification

重新執行同一 request，不再出現 missing socket，且 Thrift handler 能回應。

## Related

- [[Apache Thrift RPC]]
- [[C2 Development on Kubernetes]]

## References

- `Row notes/2023-08-31 - Houzz stack overflow (development node).md`
