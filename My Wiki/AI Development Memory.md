---
type: concept
status: growing
topics: [ai, agent, documentation, knowledge-management, architecture]
created: "2026-08-03"
---

# AI Development Memory

## Summary

AI-assisted development 的長期記憶應由 version-controlled documentation 提供，而不是依賴 chat history。將知識分成 Repo Memory、Shared Project Memory 與 Session Memory，能讓 [[AI Agent Architecture|Agent]] 依任務逐層載入最小必要 context，並維持清楚的 source of truth。

## Why It Matters

聊天內容混合已確認事實、臨時推測、debug 過程與過期 todo；全部永久保存會增加 context noise，也容易讓舊資訊覆蓋 code 的現況。真正可維護的記憶需要清楚的 ownership、lifecycle 與更新條件。

## How It Works

### Repo Memory

回答「這個 repository 如何運作」，跟著 code 放在同一 repository 並接受 version control：

```text
AGENTS.md
docs/
├── architecture.md
├── domain.md
├── conventions.md
├── decisions/
└── current-work.md
```

長期保存 repo responsibility、folder structure、architecture、domain model、coding conventions、confirmed ADRs 與 known pitfalls。只有規則、架構或重要決策改變時更新，不為每次對話追加內容。

### Shared Project Memory

回答「多個 repositories 如何合作」。只有真的出現 cross-repo flow 時才建立獨立的 project-memory repository，保存 repository map、system overview、cross-repo API contracts、event flow、shared glossary 與跨 repo decisions。

### Session Memory

回答「這次工作做到哪裡」，只保存 goal、completed、pending 與 next step。功能完成後，把仍有長期價值的內容整理進正式 Repo／Shared Memory，其餘 session notes 刪除或封存，避免把 debug log 和已完成 todo 當成永久知識。

### Context Loading Order

```text
收到任務
→ 讀目前 Repo Memory
→ 任務需要跨 Repo？
→ 讀 Shared Project Memory
→ 必要時才讀其他 Repo
```

Chat history 位於 source-of-truth hierarchy 的最下層：

```text
Code
→ Repo Memory
→ Shared Project Memory
→ Session Memory
→ Chat History
```

## Example

導入順序應由小到大：先在每個 repo 建立精簡 `AGENTS.md`，再補 architecture 與 ADR；直到功能真的跨越多個 repos，才建立 shared memory。每次工作結束只問：repo 規則是否改變、cross-repo flow 是否改變、或只是本次進度；答案決定應更新哪一層。

## Tradeoffs

- Version-controlled memory 可 review、追蹤與隨 code 演進，但需要明確 owner 主動維護。
- Shared repository 提供跨 repo 全貌，也可能與各 repo 文件重複；應只保存 relationships 與 shared contracts。
- Session memory 降低中斷成本，但若不在完成後清理，會快速變成過期 backlog。
- Progressive context loading 節省 context window，代價是 repository map 與文件入口必須容易找到。
- 文件是 code 的解釋，不應取代對實際 implementation、tests 與 runtime state 的驗證。

## Related

- [[AI Agent Architecture]]
- [[Remote Codex with Self-hosted Runner]]
- [[Side Projects]]
- [[Backend Service Production Readiness]]

## References

- [[2026-08-02 weekly updates]]
