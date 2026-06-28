# 區塊：🛵 租車（Rental）

租車是**車隊資產生意**——重點是**車隊使用率**與**未來檔期**，不是單日營業額。

**來源**：`rental project`（`xhyhfrxhvumskqzsoahk`）
- `admin_bookings`（訂單+billing，73 筆）、`fleet_vehicles`（在役 5 架）、`price_tiers`（rate card）

## ⚠️ 資料品質（先處理）
`admin_bookings.created_at` / `payment_date` 出現**未來日期（8–9 月）→ 疑為測試/種子資料**。
正式上線前，先用「真單」判定（例：排除明顯未來建立、或 `is_legacy=true` / 測試名單）再算，否則使用率與收入失真。

## 指標與查詢（rental project）
```sql
-- MTD 收入（按 pickup_date）
SELECT count(*) bookings, round(sum(total_fee)) rev_mtd
FROM admin_bookings
WHERE status IN ('confirmed','completed')
  AND pickup_date >= date_trunc('month',current_date)::date AND pickup_date < current_date;

-- 今日取車 / 還車
SELECT
  count(*) FILTER (WHERE pickup_date = current_date) AS pickups_today,
  count(*) FILTER (WHERE return_date = current_date) AS returns_today
FROM admin_bookings WHERE status IN ('confirmed','completed');

-- 車隊使用率（已租車-日 ÷ 在役車數 × 期間日數）。在役車數見 fleet_vehicles WHERE active。
WITH f AS (SELECT count(*) n FROM fleet_vehicles WHERE active),
days AS (SELECT (current_date - date_trunc('month',current_date)::date) d)
SELECT round(100.0 * (
  SELECT sum(LEAST(return_date,current_date) - GREATEST(pickup_date, date_trunc('month',current_date)::date))
  FROM admin_bookings
  WHERE status IN ('confirmed','completed') AND return_date >= date_trunc('month',current_date)::date AND pickup_date < current_date
) / NULLIF((SELECT n FROM f) * (SELECT d FROM days),0), 0) AS utilization_pct;

-- 押金在手 / 待收回（罰款+維修+隧道費 未付）
SELECT round(sum(deposit_amount)) deposits_held FROM admin_bookings WHERE status='confirmed';
```

## 月目標 + 追數
目標：**純利 $20,000/月**（config `monthly_targets.rental`）。淨利 = `total_fee − company_expense`（暫定；如有其他成本再加）。套用通用追數公式（見 `05-helmet-king.md`）。驗證值（至 6/27）：MTD 淨利 $3,640（7 單）→ 達成 18%、嚴重落後（受測試資料影響，需先清洗）。

## 輸出（Slack 行）
`🛵 租車：MTD 淨利 $X / 目標 2萬（{%}）｜ 車隊使用率 Y%（5 架）｜ 今日取車 A／還車 B ｜ 押金在手 $Z`

## 降級
查詢失敗或全為測試資料 → 標「⚠️ 租車資料待清洗」，不中斷整份報告。
