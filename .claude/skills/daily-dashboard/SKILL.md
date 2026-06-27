---
name: daily-dashboard
description: 產生頭盔王每日 dashboard 並發送到 Slack daily 頻道。當使用者要求「跑每日 dashboard」「daily update」「每日報告」，或由 cron 每天 09:30 自動觸發時使用。彙整 GitHub 熱門項目、生產力建議、skill 建立工作流程、GitHub 停滯工作提醒、頭盔王生意報表五大區塊。
---

# Daily Dashboard（頭盔王每日儀表板）

每天彙整 5 個區塊，組成一則 Slack 訊息發到使用者的 daily 頻道。設定來源：`config/dashboard.config.json`。

## 執行流程（總覽）

0. **車房發票同步**（每日先跑，把 BC GARAGE 新發票補進 garage-system `invoice_summary`，見下方「附：同步步驟」）。失敗不影響報表，記錄後繼續。
1. 讀 `config/dashboard.config.json` 取得來源、目標頻道、各區塊開關。
2. 依序產生 5 個區塊（見 `daily-dashboard/sections/`）。任一區塊抓取失敗 → 不中斷，該區塊標註「⚠️ 今日無法取得」並附原因，其餘照常。
3. 用 `daily-dashboard/slack-template.md` 的格式組成完整訊息（繁體中文）。
4. 發送到 Slack：
   - 先用 `slack_search_channels` 找 `delivery.channel_name`（預設 `daily`）。找到就用該 `channel_id`。
   - 找不到頻道 → 發 DM 給 `delivery.fallback_dm_user_id`，並在訊息開頭提醒「找不到 daily 頻道，已改發 DM，請建立或告知頻道」。
   - **預設先建草稿（`slack_send_message_draft`）還是直接發（`slack_send_message`）？** cron 自動執行時直接發；人手測試時先發草稿讓使用者過目。
5. 把當日摘要（日期 + 各區塊一句話結論）寫回 Supabase `PM` 專案的 `daily_dashboard_log` 表（若不存在則略過，不報錯），方便累積歷史與隔日「停滯工作」比對。

## 五大區塊

| # | 區塊 | 來源 | 說明檔 |
|---|------|------|--------|
| 1 | 🔥 GitHub 熱門項目 | 網頁 github.com/trending | `sections/01-github-trending.md` |
| 2 | 🚀 生產力建議 | AI 綜合產生 | `sections/02-productivity.md` |
| 3 | 🛠️ Skill 工作流程 | 輪播指南 | `sections/03-skill-workflow.md` |
| 4 | ⏳ 停滯工作提醒 | GitHub issues / PR | `sections/04-stalled-work.md` |
| 5 | 🪖 頭盔王生意報表 | Supabase（零售 / 賣車 / BC） | `sections/05-helmet-king.md` |
| 6 | 💬 社群熱門（AI + 電單車） | WebSearch | `sections/06-community-trends.md` |

## 原則

- **簡潔可行動**：每個區塊控制在 3-6 行，重點是「我今天該做什麼」，不是資料傾倒。
- **失敗不中斷**：任一資料源掛掉就降級該區塊，整份 dashboard 照樣送出。
- **不外洩機密**：Slack 連結勿放敏感 query string；生意數字只發到指定頻道。
- **可調**：所有來源、門檻、開關都在 config，不要把這些寫死在邏輯裡。

## 附：車房發票同步步驟（步驟 0 細節）

把 Retail Dashboard 的 BC GARAGE 發票，補進 garage-system `invoice_summary`（修復「停在 4/8」的根因——詳見 `/DATA-GAPS.md`）。在每日 session 內用 Supabase MCP 對兩個專案各跑一次，免額外憑證：

1. 在 garage-system 取目前最新日期：`SELECT max(invoice_date) FROM invoice_summary;`
2. 在 Retail Dashboard 產生待補列（`format(%L)` 安全跳脫），窗 = (上述最新日期, 今天]：
   ```sql
   SELECT string_agg(format('(%L,%L,%L,%L::date,%s,%s)',
     customer_number, customer_name, number, invoice_date::text,
     total_amount_incl_tax::text,
     (SELECT count(*) FROM bc_invoice_lines l WHERE l.invoice_number=i.number)::text), E',\n')
   FROM bc_sales_invoices i
   WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled'
     AND invoice_date > :last_date AND invoice_date <= current_date;
   ```
3. 在 garage-system `INSERT ... SELECT FROM (VALUES …) AS v(bc_customer_no,customer_name,invoice_number,invoice_date,total_amount,line_count)`，
   customer_id 以 `customer_profiles.bc_customer_no` 子查詢對應，並 `WHERE NOT EXISTS (… invoice_number)` 防重覆（可安全重跑）。

> 更穩健的長期做法（需憑證、非必要）：在 garage-system 設 `postgres_fdw` 連 Retail Dashboard，
> 或部署一支 edge function + pg_cron 定時同步。需要 Retail Dashboard 的 service key 作 secret。

## 手動測試

直接呼叫此 skill 即可產生並（依設定）發送或建立草稿。第一次建議用草稿模式確認格式。
