---
type: how-to
status: growing
topics: [localization, webpack, frontend]
created: "2026-07-14"
---

# Houzz Localization Lang Files

## Goal

建立符合 Houzz L10n pipeline 的 lang file，確保 `_author` 與 `_id` 都存在。

## Steps

1. 建立 lang file 並填入 `_author` 與翻譯 keys。
2. 讓現有 JavaScript module import 該 lang file。
3. 執行 `NODE_ENV=dev npm run build-apps -- -a <app-name>`。
4. Webpack lang-loader 會以 filepath 與 timestamp 產生 MD5 `_id`。

## Verification

- Build 成功，輸出包含 `_author` 與 `_id`。
- UI 在目標 locale 顯示正確文字，沒有 fallback key。
- 不手動重用其他檔案的 `_id`。

## Related

- [[C2-Jukwaa Web Module]]
- [[Web Rendering Best Practices]]

## References

- `Row notes/2025-03-21 - Houzz L10n.md`
