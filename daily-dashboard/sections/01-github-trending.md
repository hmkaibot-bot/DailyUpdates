# 區塊 1：🔥 GitHub 熱門項目

**目標**：每天挑出與使用者技術棧/業務相關的 3-5 個 GitHub 熱門 repo，附一句「為什麼值得看」。

## 做法

1. `WebFetch` 抓 `https://github.com/trending`（可加 `?since=daily`）。需要特定語言時用
   `https://github.com/trending/python?since=daily` 等。
2. 對照 `config.sections.github_trending.interests` 與 `languages` 過濾、排序。
3. 取前 `top_n`（預設 5）個。

## 每項輸出

- `[repo 名稱](連結)` — 一句話它在做什麼
- ⭐ 今日新增 star（若頁面有）
- 💡 一句「對你/公司的關聯」：例如可借鑑的自動化做法、可整合進頭盔王系統的工具

## 降級

抓不到 trending 頁面時，改用 `WebSearch` 找「本週 GitHub 熱門 AI / 自動化 repo」，標註為搜尋結果而非官方 trending。
