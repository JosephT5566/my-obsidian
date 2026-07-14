---
type: concept
status: growing
topics: [ai, agent, function-calling, mcp, skill]
created: "2026-07-14"
---

# AI Agent Architecture

## Summary

AI Agent 將 LLM 的推理能力與工具執行迴圈結合。System prompt 定義行為，Function calling 提供結構化工具介面，MCP 將工具服務標準化，而 Skill 讓專門指令與資源按需載入。

## Why It Matters

單純的 LLM 只能根據上下文產生回覆。要可靠地查資料、操作外部系統或執行多步驟任務，需要清楚區分模型、指令、工具協定與執行環境。

## How It Works

1. **System prompt**：提供角色、規則、背景與工作方式，和 User prompt 分開。
2. **Function calling**：用結構化 Schema 描述工具名稱與參數，降低模型輸出無效格式的機率。
3. **Agent loop**：模型判斷下一步，Client 執行工具，再把結果回傳給模型，直到任務完成。
4. **MCP**：透過 MCP Server 暴露工具或資源，由 MCP Client 使用一致介面呼叫。
5. **Skill**：先提供簡短 Metadata 供模型判斷相關性，命中後才讀取完整 `SKILL.md`、參考資料或腳本。

## Example

```text
User request
  → Agent reads relevant Skill
  → Model selects a tool
  → Client validates and executes the call
  → Tool result returns to the model
  → Model continues or produces the final answer
```

## Tradeoffs

- Function calling 改善格式可靠性，但仍需要參數驗證、權限控制與錯誤處理。
- MCP 降低多個 Client 重複整合工具的成本，也增加 Server lifecycle 與信任邊界。
- Skill 的 Progressive disclosure 節省 Context window，但 Metadata 品質會影響是否正確啟用。
- Agent 自主性越高，越需要清楚的 Scope、Approval 與 Verification。

## Related

- [[Expense Receipt AI Pipeline]]
- [[Interview Preparation with AI]]
- [[Remote Codex with Self-hosted Runner]]

## References

- [[2026-05-24 - 5-24 weekly update]]
- [Model Context Protocol](https://modelcontextprotocol.io/)

