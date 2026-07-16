# 鎖定查詢（LOCKED SQL）— skill 必須逐條照跑，不得即興改寫

目的：令每日（含 autonomous routine）run 出嘅數字**一致、可審計**。所有日期用 `current_date` 動態計，任何日子啱用。

## 通用定義
- `LCD`（最後完整日）= `current_date - 1`
- **報告月**：若今日係 1 號（`date_part('day',current_date)=1`）→ 報**上一個完整月**；否則報**本月至今**（月初 → LCD）。各查詢用 `ms`（月初）、`me`（月尾/LCD）表示，見各段 CTE。
- 賣車淨利 = 毛利 × 0.9（減 10% 佣）；寄賣毛利只計佣金。營收只計 `< current_date`（排除今日）。

```sql
-- 共用月窗 CTE（貼喺每條查詢頭）
WITH win AS (
  SELECT
    CASE WHEN date_part('day',current_date)=1
         THEN (date_trunc('month',current_date)-interval '1 month')::date
         ELSE date_trunc('month',current_date)::date END AS ms,
    CASE WHEN date_part('day',current_date)=1
         THEN (date_trunc('month',current_date)-interval '1 day')::date   -- 上月最後一日
         ELSE (current_date-1) END AS me,
    (current_date-1) AS lcd
)
```

---
## Q1 — 零售 + 車房BC（Retail Dashboard `myrangmxyjamsupbxbba`）
```sql
WITH win AS (SELECT CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 month')::date ELSE date_trunc('month',current_date)::date END ms, CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date ELSE (current_date-1) END me)
SELECT
 -- 零售昨日/前日/上週同日
 (SELECT round(sum(total_price)) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date=me) r_yday,
 (SELECT count(*) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date=me) r_yday_ord,
 (SELECT round(sum(total_price)) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date=me-1) r_prev,
 (SELECT round(sum(total_price)) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date=me-7) r_wow,
 -- 零售 MTD + 上月同期
 (SELECT round(sum(total_price)) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date BETWEEN ms AND me) r_mtd,
 (SELECT count(*) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date BETWEEN ms AND me) r_mtd_ord,
 (SELECT round(sum(total_price)) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date BETWEEN (ms-interval '1 month')::date AND (me-interval '1 month')::date) r_lastmonth,
 -- 車房 BC GARAGE 昨日/上週/ MTD /上月同期
 (SELECT round(sum(total_amount_incl_tax)) FROM bc_sales_invoices,win WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date=me) g_yday,
 (SELECT count(*) FROM bc_sales_invoices,win WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date=me) g_yday_cnt,
 (SELECT round(sum(total_amount_incl_tax)) FROM bc_sales_invoices,win WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date=me-7) g_wow,
 (SELECT round(sum(total_amount_incl_tax)) FROM bc_sales_invoices,win WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date BETWEEN ms AND me) g_mtd,
 (SELECT count(*) FROM bc_sales_invoices,win WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date BETWEEN ms AND me) g_mtd_cnt,
 (SELECT round(sum(total_amount_incl_tax)) FROM bc_sales_invoices,win WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled' AND invoice_date BETWEEN (ms-interval '1 month')::date AND (me-interval '1 month')::date) g_lastmonth;
```

## Q1b — 零售 Top 賣品 + 低庫存暢銷（Retail Dashboard）
```sql
WITH win AS (SELECT CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 month')::date ELSE date_trunc('month',current_date)::date END ms, CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date ELSE (current_date-1) END me)
SELECT
 (SELECT json_agg(t) FROM (SELECT title, round(sum(line_total)) amt, sum(quantity) qty FROM shopify_order_lines,win WHERE created_at::date BETWEEN ms AND me GROUP BY title ORDER BY 2 DESC LIMIT 5) t) top_products,
 (SELECT json_agg(l) FROM (
   WITH sold AS (SELECT sku, sum(quantity) q FROM shopify_order_lines,win WHERE created_at::date BETWEEN ms AND me AND sku IS NOT NULL GROUP BY sku),
        inv AS (SELECT DISTINCT ON (sku) sku, product_title, inventory_quantity FROM shopify_inventory WHERE sku IS NOT NULL ORDER BY sku, snapshot_date DESC)
   SELECT i.product_title, i.inventory_quantity qty, s.q sold FROM inv i JOIN sold s ON s.sku=i.sku WHERE i.inventory_quantity<=3 AND i.product_title NOT ILIKE '%gift card%' ORDER BY s.q DESC LIMIT 5) l) low_stock;
```

## Q2 — 賣車全部（26king-trading `kpvakfbxpachnotjvnyu`）
```sql
WITH win AS (SELECT CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 month')::date ELSE date_trunc('month',current_date)::date END ms, CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date ELSE (current_date-1) END me)
SELECT
 (SELECT count(*) FROM vehicles,win WHERE sold_at::date BETWEEN ms AND me) sold_cnt,
 (SELECT round(0.9*sum(CASE WHEN acquisition_type='consignment' THEN coalesce(consignment_sale_price,final_sale_price,actual_sale_price,0)*coalesce(consignment_commission_rate,0)/100.0 ELSE coalesce(final_sale_price,actual_sale_price,0)-coalesce(total_cost,0) END)) FROM vehicles,win WHERE sold_at::date BETWEEN ms AND me) net_profit,
 (SELECT count(*) FROM vehicles WHERE lifecycle_status='reserved') reserved,
 (SELECT round(sum(CASE WHEN acquisition_type='consignment' THEN coalesce(consignment_sale_price,final_sale_price,target_sale_price,0)*coalesce(consignment_commission_rate,0)/100.0 ELSE coalesce(final_sale_price,target_sale_price,estimated_sale_price,0)-coalesce(total_cost,0) END)) FROM vehicles,win WHERE sold_at IS NULL AND (lifecycle_status='reserved' OR sale_status='deposit_paid') AND coalesce(reserved_at::date,customer_deposit_date::date) BETWEEN ms AND me) pipeline,
 (SELECT count(*) FROM vehicles WHERE sold_at IS NULL AND coalesce(is_archived,false)=false AND lifecycle_status NOT IN ('sold','completed')) stock_cnt,
 (SELECT round(sum(coalesce(total_cost,0))) FROM vehicles WHERE sold_at IS NULL AND coalesce(is_archived,false)=false AND lifecycle_status NOT IN ('sold','completed')) stock_capital,
 (SELECT count(*) FROM vehicles WHERE sold_at IS NULL AND coalesce(is_archived,false)=false AND lifecycle_status NOT IN ('sold','completed') AND coalesce(intake_at::date,listed_at::date,created_at::date)<=current_date-60) aged60,
 (SELECT round(sum(amount) FILTER (WHERE direction='inflow'  AND payment_date BETWEEN ms AND me)) FROM payment_records,win) cash_in,
 (SELECT round(sum(amount) FILTER (WHERE direction='outflow' AND payment_date BETWEEN ms AND me)) FROM payment_records,win) cash_out,
 (SELECT round(sum(coalesce(dealer_balance,0))) FROM vehicles WHERE coalesce(dealer_balance,0)>0 AND dealer_balance_date IS NULL AND coalesce(is_archived,false)=false) ap,
 (SELECT count(*) FROM costgo_renewal_cases,win WHERE coalesce(payment_received_at,renewal_effective_date) BETWEEN ms AND me AND status IN ('completed','legacy_completed')) ins_cnt,
 (SELECT round(sum(coalesce(gross_premium,0))) FROM costgo_renewal_cases,win WHERE coalesce(payment_received_at,renewal_effective_date) BETWEEN ms AND me AND status IN ('completed','legacy_completed')) ins_gross,
 (SELECT count(*) FROM vehicles,win WHERE sold_at::date BETWEEN ms AND me AND acquisition_type<>'consignment' AND coalesce(total_cost,0)>0 AND coalesce(final_sale_price,actual_sale_price) IS NOT NULL AND coalesce(final_sale_price,actual_sale_price)<total_cost) neg_margin,
 (SELECT count(*) FROM vehicles,win WHERE sold_at::date BETWEEN ms AND me AND coalesce(final_sale_price,actual_sale_price) IS NULL) missing_price,
 (SELECT count(*) FROM costgo_renewal_cases WHERE expiry_date BETWEEN current_date AND current_date+14 AND coalesce(status,'') NOT IN ('completed','legacy_completed','declined','cancelled')) ins_expiring;
```
> 保險佣金 = `ins_gross × 0.15`。

## Q3 — 車房營運 + 未來預約（garage-system `qxxegmvwtndoosqrhyar`）
```sql
SELECT
 (SELECT count(*) FROM appointments WHERE scheduled_at::date=current_date AND status NOT IN ('cancelled','rejected')) appt_today,
 (SELECT count(*) FROM appointments WHERE scheduled_at::date=current_date+1 AND status NOT IN ('cancelled','rejected')) appt_tmr,
 (SELECT count(*) FROM job_orders WHERE overall_status::text IN ('完成','待交車')) waiting_delivery,
 (SELECT count(*) FROM job_orders WHERE coalesce(is_long_stay,false)=true AND overall_status::text NOT IN ('已交車','完成','已完成,待交車')) long_stay,
 (SELECT count(*) FROM appointments WHERE parts_eta<current_date AND coalesce(parts_arrived,false)=false) parts_overdue,
 (SELECT count(*) FROM daily_cash_reports WHERE report_date=current_date-1) dcr_yday,
 (SELECT json_agg(f) FROM (SELECT scheduled_at::date d, to_char(scheduled_at,'MM/DD(Dy)') label, count(*) n FROM appointments WHERE status NOT IN ('cancelled','rejected') AND scheduled_at::date BETWEEN current_date AND current_date+6 GROUP BY 1,2 ORDER BY 1) f) next7_booking;
```
> 未來預約不足：`next7_booking` 內任何一日 `n < 4`（config `garage_booking_alert.low_booking_threshold_per_day`）→ flag 並列該日。

## 追數/GP 計法
見 `sections/05-helmet-king.md`（通用追數公式 + 集團總 GP 五+租屋 分項相加，必填）。目標見 `config.monthly_targets` / `group_profit`。

## 規則
- **逐條照跑，唔准即興改寫欄位/口徑**；發現查詢報錯先修此檔，唔好喺 run 時亂改。
- 跑完把結果套入 `report-layouts.md` 各版面。
