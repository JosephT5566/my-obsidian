---
type: concept
status: growing
topics: [ai, structured-output, validation, prompt-injection, human-in-the-loop, security]
created: "2026-07-28"
---

# AI Structured Output Validation

## Summary

AI structured output 讓 model 依 JSON Schema 產生固定形狀的候選資料，但只保證 output shape，不保證內容真實、符合業務規則或具有操作權限。安全的 workflow 會把 model output 視為 untrusted input，經過 Server validation、least-privilege isolation 與必要的 human review 後，才形成正式資料或 side effect。

## Why It Matters

[[Expense Receipt AI Pipeline]] 使用 AI 從收據提取 merchant、date、currency、total 與 items。即使 provider strict mode 能產生 schema-compatible JSON，模型仍可能讀錯金額、接受 receipt 內的 prompt injection，或產生不屬於目前 user／tenant 的 identifier。

核心原則是：

> AI 只產生 candidate data，不直接決定 canonical business state。

## How It Works

### Separate Instructions from Untrusted Data

System instruction 應明確說明任務，OCR、文件或 user content 則作為 untrusted data 放入獨立欄位。JSON、XML tag 或 delimiter 能幫助 model 理解邊界，但不是 security boundary；prompt injection 仍可能影響 output。

真正的 blast-radius control 來自：

- AI service 不持有不必要的 database write permission。
- Model 不能自行決定 `user_id`、`tenant_id` 或 authorization role。
- Identity 與 tenant scope 由 authenticated session 提供。
- Tool allowlist、parameter validation 與 authorization 在 handler 重新執行。
- 高風險或不可逆操作要求 human approval。

這與 [[AI Agent Architecture]] 的 tool boundary 相同：Schema 改善格式可靠性，權限與執行正確性仍由 deterministic code 負責。

### Strict Output Allowlist

Schema 只接受業務需要的欄位，拒絕未知欄位，並限制 enum、長度、範圍與 array size：

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["merchant", "currency", "total", "items"],
  "properties": {
    "merchant": {"type": "string", "maxLength": 200},
    "currency": {"type": "string", "enum": ["TWD", "USD", "JPY"]},
    "total": {"type": "number", "minimum": 0, "maximum": 1000000},
    "items": {"type": "array", "maxItems": 100}
  }
}
```

Authorization fields、database owner、tenant ID 與 internal status 不應由 model output 提供。即使 provider 宣稱 strict schema adherence，Backend 仍要用自己控制的 schema 再次解析和驗證，以處理 provider error、版本差異與 application-specific constraints。

### Three Validation Layers

1. **Structural validation**
   - Required fields、type、enum、date format、unknown fields、size limits。
2. **Business validation**
   - Amount 非負、date 不明顯在未來、line items 與 total 合理、currency 受支援。
3. **Authorization and provenance**
   - Draft 與 source file 屬於目前 actor。
   - `user_id`／`tenant_id` 只來自 trusted session。
   - 來源 object path 與 authenticated scope 相符。

Schema-valid 不等於 factually correct。`total = 10000` 可以是合法 number，仍可能把收據上的 `1000` 辨識錯誤；這類問題需要 domain rule、原始證據與 human review。

### Human-in-the-loop Draft

Model output 先建立 draft，使用者對照原始資料檢查、修改，再主動 confirm。Frontend validation 改善操作體驗，但 PATCH 與 confirm endpoint 都必須重新執行 Server validation，因為 Client 可以繞過 UI。

Draft 建議同時保存：

- `raw_extraction`：不可修改的 model output。
- `draft_value`：目前可編輯的候選資料。
- `schema_version`、`model_version`、`prompt_version`。
- `validation_result` 與 user-modified fields。
- Source file、owner、timestamps 與 terminal resource ID。

保留 raw 與 edited value 能支援 audit、debug、model comparison 與品質指標。若要把使用者修正用於訓練，還需另外處理 consent、privacy 與 data governance。

具體的 draft state、optimistic concurrency 與 confirm transaction 見 [[Expense Draft Review Workflow]]。

## Example

```text
Receipt / OCR (untrusted)
→ AI structured output
→ Server schema validation
→ Business validation
→ Owner and source validation
→ Draft
→ Human review and edit
→ Revalidation
→ Confirm canonical expense
```

模型被 prompt injection 影響時，最壞結果應是產生被拒絕或等待修正的 candidate，而不是直接修改正式 expense、跨 tenant 讀取資料或呼叫高權限工具。

## Tradeoffs

- Strict schema 降低 parsing ambiguity，但 schema 越複雜，provider compatibility 與 version migration 成本越高。
- Human review 提高正確性與安全性，也增加 completion latency 和 user effort。
- 保存 raw output 改善 audit 與 model evaluation，代價是 storage、PII 與 retention governance。
- Prompt separation 是 defense in depth，不是 prompt injection 的完整解法。
- Model confidence 可以協助排序 review，不應單獨成為跳過 validation 或 authorization 的依據。

## Related

- [[Expense Receipt AI Pipeline]]
- [[Expense Draft Review Workflow]]
- [[AI Agent Architecture]]
- [[API Contract Design]]
- [[Backend Service Production Readiness]]

## References

- [[2026-07-26 weekly updates]]
- [[2026-08-02 weekly updates]]
- [OpenAI API: Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [OWASP: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
