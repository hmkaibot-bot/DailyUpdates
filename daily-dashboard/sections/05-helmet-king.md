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

本月至今 MTD（月初 → 最後完整日，並與上月同期比較）：
```sql
WITH b AS (SELECT date_trunc('month',current_date)::date ms,
                  (date_trunc('month',current_date)-interval '1 month')::date lms,
                  (current_date - date_trunc('month',current_date)::date) AS days_elapsed)
SELECT
  (SELECT round(sum(total_price)) FROM shopify_orders,b WHERE cancelled_at IS NULL AND created_at::date>=ms AND created_at::date<current_date) AS retail_mtd,
  (SELECT round(sum(total_price)) FROM shopify_orders,b WHERE cancelled_at IS NULL AND created_at::date>=lms AND created_at::date<lms+days_elapsed) AS retail_lastmonth_same,
  (SELECT round(sum(total_amount_incl_tax)) FROM bc_sales_invoices,b WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date>=ms AND invoice_date<current_date) AS garage_mtd,
  (SELECT round(sum(total_amount_incl_tax)) FROM bc_sales_invoices,b WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date>=lms AND invoice_date<lms+days_elapsed) AS garage_lastmonth_same
FROM b;
-- MTD 區間 = [月初, current_date)；上月同期 = [上月初, 上月初+已過天數)，確保同日數可比。
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
【昨日 / 最後完整日】
• 零售：$X（N 單）　▲/▼ vs 前一日　▲/▼ vs 上週同日
• 車房 (BC GARAGE)：$X（N 張發票）　▲/▼ vs 前一日　▲/▼ vs 上週同日
【本月至今 MTD（{month_start}→{last_complete_day}）】
• 零售：$X（N 單）　▲/▼ vs 上月同期
• 車房：$X（N 張）　▲/▼ vs 上月同期
【營運】今日預約 M ｜ 未來預約 K ｜（選填）今日進行中 $Y _未結算_
👉 一句洞察（例如：零售單日走弱屬波動 / 連跌兩日建議查廣告）
```
> 預約是前瞻資訊，用 current_date 是對的；營收用 last_complete_day。

## 月目標追數（所有四線）

設定見 config `monthly_targets`（零售 150萬營 / 車房 40萬營 / 賣車 25萬純利 / 租車 2萬純利）。
通用公式（以最後完整日 LCD 計）：
```
LCD          = current_date - 1
elapsed      = day-of-month(LCD)                 例：LCD=15號 → elapsed=15
days_in_month= 當月總日數                         例：6月=30
達成%        = MTD ÷ 目標
應達%        = elapsed ÷ days_in_month            例：15/30 = 50%
距標         = 目標 − MTD
餘下日       = days_in_month − elapsed
每日需追     = 距標 ÷ 餘下日
預估埋月     = MTD ÷ elapsed × days_in_month
```
- 零售/車房 用「MTD 營業額」對營業額目標；賣車用「已實現純利」對純利目標；租車用「淨利」對純利目標。
- 落後（達成% < 應達%）以 ⚠️ 標示。

## 集團總利潤（所有部組相加）

設定見 config `group_profit`。**注意：零售/車房 DB 無成本(COGS)**，故用毛利率假設估算；賣車/租車為真實淨利。
```
集團總利潤(MTD) ≈ 零售營業額×零售毛利率(假設)
                + 車房營業額×車房毛利率(假設)
                + 賣車已實現純利(真實)
                + 租車淨利(真實)
```
> 必附一句 caveat：「零售/車房利潤為毛利率估算（DB 無 COGS）」，避免誤當精算。毛利率待老闆提供實際值。

## 輸出格式（集團追數版）
```
🪖 集團生意（資料截至 {LCD}｜本月 {elapsed}/{days_in_month} 日）
線別            MTD          目標     達成%   距標      餘{n}日需/日
零售(營)        $1.21M       $1.5M    81%     $289k     $96k
車房(營)        $306k        $400k    76%     $94k      $31k
賣車(純利)      $197k        $250k    79%     $53k      $18k
租車(純利)      $3.6k        $20k     18%     $16k      $5.5k
— 集團總利潤(估)  $XXX,XXX     （零售/車房為毛利率估算）
👉 跨線洞察（哪條最落後、最該追）
```

## 降級

任一業務線查詢失敗 → 該行標「⚠️ 今日無法取得（原因）」，其餘照常輸出。
