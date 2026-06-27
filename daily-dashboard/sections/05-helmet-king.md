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

零售當日營業額（Retail Dashboard）：
```sql
SELECT date_trunc('day', created_at) AS d,
       count(*) AS orders,
       sum(total_price) AS revenue
FROM shopify_orders
WHERE created_at >= current_date - interval '8 days'
GROUP BY 1 ORDER BY 1 DESC;
```

車房當日營收（權威：Retail Dashboard，BC GARAGE 維度）：
```sql
SELECT invoice_date AS d, count(*) AS invoices, sum(total_amount_incl_tax) AS revenue
FROM bc_sales_invoices
WHERE dimension1_code = 'GARAGE' AND number LIKE 'SI-%' AND status <> 'Canceled'
  AND invoice_date >= current_date - interval '8 days'
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

```
🪖 頭盔王生意報表（資料截至 YYYY-MM-DD）
• 零售：$X（N 單）　▲/▼ vs 昨日 Y%　｜ Top: 商品A
• 車房：N 張工單　今日預約 M 個　現金日結 $Z
• BC：$X（N 張發票）
👉 一句洞察 / 提醒（例如：零售連兩日下滑，建議檢查廣告投放）
```

## 降級

任一業務線查詢失敗 → 該行標「⚠️ 今日無法取得（原因）」，其餘照常輸出。
