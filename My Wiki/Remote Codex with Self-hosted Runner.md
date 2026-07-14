---
type: how-to
status: draft
topics: [codex, github-actions, self-hosted-runner, remote-access]
created: "2026-07-14"
---

# Remote Codex with Self-hosted Runner

## Goal

在不攜帶 MacBook 的情況下，透過 GitHub Actions self-hosted runner 觸發本機 CLI 工作流程。

## Prerequisites

- 一台保持開機、連網且不自動睡眠的 Mac
- GitHub repository 與 Self-hosted runner
- 已在本機安裝並驗證可用的 CLI 工作流程
- 對 Workflow dispatch、Secret 與 Runner security 的基本理解

## Steps

1. 在專用 Repository 設定 Self-hosted runner，並限制可觸發 Workflow 的人員與 Branch。
2. 在 Mac 上安裝 Runner service，使用低權限的專用帳號執行。
3. 建立手動 `workflow_dispatch`，先只允許固定、白名單內的命令。
4. 若需要週期執行，再使用 GitHub Actions schedule、macOS `launchd` 或其他排程機制。
5. 關閉系統自動睡眠，並在離開前從外部網路做一次完整測試。
6. 保存執行 Log、Exit code 與產出位置，讓失敗時可以遠端診斷。

## Verification

- 從另一台裝置成功觸發 Workflow。
- Runner 能收到任務並回報結果。
- 螢幕關閉一段時間後再次測試，確認主機沒有進入睡眠。
- Workflow 無法執行白名單以外的任意 Shell input。

## Troubleshooting

- Runner 顯示 Offline：檢查 Mac 是否睡眠、網路是否中斷、Runner service 是否仍在執行。
- CLI 能力和 Desktop App 不一致：先確認工作流程是否依賴只存在於 App 的 Plugin、登入 Session 或互動式 Approval。
- 遠端任務卡住：避免需要鍵盤確認的命令，並替每個步驟設定 Timeout。
- Chrome automation 不穩：遠端 CLI 未必具備 Desktop App 的 Browser state，應先在離開前驗證實際執行環境。

## Related

- [[AI Agent Architecture]]
- [[System Design Foundations]]

## References

- [[2026-06-14 - 6-14 weekly update]]
- [my-actions-runner](https://github.com/JosephT5566/my-actions-runner)

