# 區塊 5：🪖 頭盔王生意報表

**目標**：每天彙整頭盔王集團三條業務線的關鍵數字，給一份「老闆早上看一眼就懂」的摘要。

## 三條業務線（來源 = Supabase）

| 業務線 | 權威來源 | project_id | 重點表 |
|--------|----------|-----------|--------|
| 零售（Helmet King Shopify） | Retail Dashboard | `myrangmxyjamsupbxbba` | `shopify_orders`, `shopify_order_lines`, `shopify_products`, `meta_ad_insights` |
| **車房營收（Helmet King + 26King）** | **Retail Dashboard 的 BC GARAGE 維度** | `myrangmxyjamsupbxbba` | **`bc_sales_invoices` WHERE `dimension1_code='GARAGE'`**, `bc_invoice_lines` |
| 車房營運（預約/工單） | garage-system | `qxxegmvwtndoosqrhyar` | `appointments`, `job_orders`（只用狀態/數量，**不用 final_price**） |

> ⚠️ **車房營收一律取 BC GARAGE 維度**，不要用 garage-system 的 `invoice_summary`（停在 2026-04-08）、
> `daily_cash_reports`（空表）、或 `job_orders.final_price`（全 NULL）——詳見 `/DATA-GAPS.md` 調查。
> 這三張是車房 app 的營運表、目前未維護，拿來算營收會嚴重失真。

## ⏰ 重要：以「最後一個完整日」為準

dashboard 在早上跑，**當日（current_date）尚未結束，數字不準**。所以：
- 營收與一切比較，**一律取「最後一個完整日」** = 最近一個 `< current_date` 且有資料的日子（通常是昨天）。
- **今日（current_date）數字不要當績效**：可選擇性附一行「今日進行中 $X（未結算，僅參考）」，但**不得**用於 ▲▼ 比較。
- 比較基準：last_complete_day vs 前一個有資料日、vs 上週同一天（−7 天）。
- 報表標題的「資料截至」= last_complete_day（不是 current_date）。

## 做法

1. **先驗證欄位**：每張表第一次使用前，用
   `SELECT column_name, data_type FROM information_schema.columns WHERE table_name = '<表>'`
   確認欄位名（日期欄、金額欄、狀態欄），不要猜欄位名。
2. **查資料新鮮度**：先看 `etl_sync_log`（Retail Dashboard）與 `bc_sync_state`（garage-system）
   確認資料更新到哪天。若資料不是今天的，在區塊標明「資料截至 YYYY-MM-DD」。
3. **跑當日彙總**，每條業務線抓：
   - 當日營業額 / 訂單（或工單）數
   - 對比：vs 前一日、vs 上週同一天
   - Top 商品 / 服務
4. 全部用 `mcp__Supabase__execute_sql`（唯讀 SELECT），**絕不執行寫入/DDL**。

## 範例查詢（執行前先用 information_schema 核對欄位名）

零售營業額（最後一個完整日為準，排除今日 current_date）：
```sql
SELECT created_at::date AS d, count(*) AS orders, sum(total_price) AS revenue
FROM shopify_orders
WHERE cancelled_at IS NULL
  AND created_at::date < current_date            -- 只取已結束的日子
  AND created_at >= current_date - interval '9 days'
GROUP BY 1 ORDER BY 1 DESC;
-- 取第 1 列 = last_complete_day；第 2 列 = 前一日；找 d = last_complete_day - 7 = 上週同日。
-- 今日(current_date)若要附參考，另跑一條並標「未結算」，不參與比較。
```

車房營收（權威：BC GARAGE 維度，最後一個完整日為準，排除今日）：
```sql
SELECT invoice_date AS d, count(*) AS invoices, sum(total_amount_incl_tax) AS revenue
FROM bc_sales_invoices
WHERE dimension1_code = 'GARAGE' AND number LIKE 'SI-%' AND status <> 'Canceled'
  AND invoice_date < current_date                -- 只取已結束的日子
  AND invoice_date >= current_date - interval '9 days'
GROUP BY 1 ORDER BY 1 DESC;
```

車房當日營運（garage-system，只取數量/狀態與預約）：
```sql
-- 今日預約
SELECT count(*) FILTER (WHERE scheduled_at::date = current_date) AS today_appts
FROM appointments;
-- 今日新工單（數量，不用 final_price）
SELECT count(*) AS jobs_today FROM job_orders WHERE created_at::date = current_date;
```

## 輸出格式

「資料截至」= **last_complete_day**（最後一個完整日，非今日）。
```
🪖 頭盔王生意報表（資料截至 {last_complete_day}）
• 零售：$X（N 單）　▲/▼ vs 前一日　▲/▼ vs 上週同日　｜ Top: 商品A
• 車房 (BC GARAGE)：$X（N 張發票）
• 車房營運：今日預約 M 個 ｜ 未來預約 K 個（前瞻，可用今日）
• （選填）今日進行中：零售 $Y — _未結算，僅參考_
👉 一句洞察（例如：零售單日走弱屬波動 / 連跌兩日建議查廣告）
```
> 預約是前瞻資訊，用 current_date 是對的；營收用 last_complete_day。

## 降級

任一業務線查詢失敗 → 該行標「⚠️ 今日無法取得（原因）」，其餘照常輸出。
