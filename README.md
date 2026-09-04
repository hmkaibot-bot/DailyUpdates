# 頭盔王 Daily Dashboard

每天早上 **09:30** 自動彙整一份儀表板，發到 Slack 的 `daily` 頻道。由 Claude Code 排程（cron）觸發，執行 `.claude/skills/daily-dashboard` skill。

## 六大區塊

| # | 區塊 | 來源 |
|---|------|------|
| 1 | 🔥 GitHub 熱門項目 | github.com/trending（WebFetch） |
| 2 | 🚀 生產力建議（個人＋公司） | AI 綜合當日 context 產生 |
| 3 | 🛠️ 建立 Skill 的工作流程 | 每日輪播指南 |
| 4 | ⏳ 停滯工作提醒 | GitHub issues / PR |
| 5 | 🪖 頭盔王生意報表 | Supabase（零售 / 車房 / BC） |
| 6 | 💬 社群熱門（AI + 電單車） | WebSearch（Reddit 直連被擋，改用搜尋歸納） |

## 專案結構

```
config/dashboard.config.json      ← 所有來源、頻道、門檻、開關（改這裡就好）
.claude/skills/daily-dashboard/   ← 主 skill（cron 呼叫的對象）
daily-dashboard/
  ├── slack-template.md           ← Slack 訊息格式
  └── sections/                   ← 五個區塊各自的產生說明
```

## 資料來源對應（頭盔王）

| 業務線 | 權威來源 | project_id |
|--------|----------|-----------|
| 零售（Helmet King Shopify） | Retail Dashboard | `myrangmxyjamsupbxbba` |
| 車房營收 + 毛利（2026-09-01 起） | **garage-system 交車工單**（`v_job_order_billing` + `job_task_parts.unit_cost`） | `qxxegmvwtndoosqrhyar` |
| 車房營運（預約/工單） | garage-system | `qxxegmvwtndoosqrhyar` |

> 📋 **2026-09-04 起車房營收唯一權威 = garage-system 交車工單**（老闆決定不再以 BC 為準）。
> BC GARAGE 只作 ≤2026-08-31 的歷史腳註參考，**絕不可與 garage-system 相加**。
> 不要用 `invoice_summary`（BC 鏡像、凍結 2026-08-31）/ `daily_cash_reports`（空表）/ `job_orders.final_price`（全 NULL）。
> 詳見 `DATA-GAPS.md` 與 `daily-dashboard/queries.md` Q3b。

## 怎麼用

- **手動跑**：對 Claude 說「跑 daily dashboard」或 `/daily-dashboard`。首次建議用草稿模式確認格式。
- **自動跑**：cron 每天 09:30（`Asia/Hong_Kong`）觸發，直接發送到 daily 頻道。
- **調整內容**：編輯 `config/dashboard.config.json`（開關區塊、改門檻、加 repo、改時間）。

## 已知限制 / 待確認

1. **Slack `daily` 頻道**：目前搜尋不到同名頻道，需確認正確頻道名/ID（填入 config `delivery.channel_id`），或由本人建立。找不到時 skill 會 fallback 發 DM。
2. **GitHub 授權範圍**：目前僅 `hmkaibot-bot/dailyupdates`。停滯工作偵測只覆蓋此 repo；要監測更多 repo 需擴大授權。
3. **賣車 / BC 對應**：依資料表結構推斷，待最終確認。

## ⚠️ 安全提醒（Supabase RLS）

掃描兩個 Supabase 專案時發現多張資料表 **停用了 Row Level Security (RLS)**，會暴露給 anon key（任何持有 anon key 的人可讀寫）：

- **Retail Dashboard**：24 張表（多為 `recon_*`、`bc_*` ledger、`invoice_*`、`competitor_*`、`fx_rates`）
- **garage-system**：18 張表（含 `credit_memos`、`inventory_adjustments`、`fifo_cost_layers`、`part_*_master` 等）

此 dashboard 只做唯讀查詢，不影響也不修復這個問題，但**建議另行為這些表加上 RLS 與適當 policy**。詳見 Supabase 文件：https://supabase.com/docs/guides/database/postgres/row-level-security
