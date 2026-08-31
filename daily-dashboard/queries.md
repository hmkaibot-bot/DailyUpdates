# 鎖定查詢（LOCKED SQL）— skill 必須逐條照跑，不得即興改寫

目的：令每日（含 autonomous routine）run 出嘅數字**一致、可審計**。所有日期用 `current_date` 動態計，任何日子啱用。

## 通用定義
- `LCD`（最後完整日）= `current_date - 1`
- **報告月**：若今日係 1 號（`date_part('day',current_date)=1`）→ 報**上一個完整月**；否則報**本月至今**。各查詢用 `ms`（月初）、`me`（LCD）、`me_mtd`（含今日）表示，見各段 CTE。
- **兩個月尾（重要）**：
  - `me` = LCD = `current_date-1` → **流量/單日指標**用（昨日營收、日比較、保險宗數）。「營收報最後完整日」呢條規則只適用呢類。
  - `me_mtd` = `current_date` → **累計/MTD 指標**用（成交台數、收入、毛利、淨利、現金流），同 26KING app「本月至今」對齊。用 LCD 會令月底最後一日少計一單，老闆對住 app 就覺得數唔準；亦會出現「8/31 報到 8/30、9/1 報到 8/31」同一個月無故變數。
- 賣車淨利 = 毛利 × 0.9（減 10% 員工佣）。⚠️ **26KING app headline「已實現利潤」係未計佣金 = 毛利**，所以報表兩個都要出，唔好淨報一個。

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

**已對齊 26KING app「總覽 / 本月至今」**。2026-08-31 實測完全重現 app 全部 headline：
`sold_cnt 26 · revenue 1,098,914 · gross_profit 139,455 · own_gross 127,375 · consign_gross 12,080 · own_cost 946,416 · reserved 16 · pipeline 420,110 · pend_balance 805,500 · pend_deposit 205,000`。
**改口徑前先睇下面「口徑依據」，唔好還原成舊版。**

```sql
WITH win AS (
  SELECT
    CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 month')::date
         ELSE date_trunc('month',current_date)::date END ms,
    CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date
         ELSE (current_date-1) END me,          -- LCD：流量/單日指標用
    CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date
         ELSE current_date END me_mtd           -- 含今日：MTD 累計指標用（同 app 一致）
),
sold AS (   -- 已成交池
  SELECT v.*,
    coalesce(v.final_sale_price, v.actual_sale_price)                                      AS sp,
    coalesce(nullif(coalesce(v.consignment_sale_price,0),0), coalesce(v.purchase_price,0)) AS csp_eff,
    coalesce(v.consignment_commission_rate,7)                                              AS ccr_eff,
    coalesce(v.consignment_end_date, v.sold_at::date) - v.consign_start_date               AS rent_days
  FROM vehicles v, win w
  WHERE v.sold_at::date BETWEEN w.ms AND w.me_mtd
    AND v.lifecycle_status IN ('completed','sold')      -- 隔走 pending_prep / listed 等未終結狀態
),
soldp AS (
  SELECT s.*,
    CASE WHEN s.acquisition_type='consignment' THEN
           CASE WHEN coalesce(s.sp,0)<=0 THEN NULL                       -- 寄賣未定價 → 唔計利潤
                ELSE (s.sp - s.csp_eff)                                  -- 差價
                   + round(s.csp_eff*s.ccr_eff/100.0,2)                  -- 佣金（基數 = csp_eff）
                   + coalesce(s.rent_days,0)*coalesce(s.daily_rent,0) END -- 寄存租金 N日 × $70
         WHEN s.sp IS NULL THEN NULL                                     -- 自家車未入價 → 唔當蝕晒成本
         ELSE s.sp - coalesce(s.total_cost,0) END AS gp
  FROM sold s
),
pend AS (   -- 待埋單池：存量指標，唔設月窗
  SELECT v.*,
    coalesce(v.final_sale_price, v.actual_sale_price)                                      AS sp,
    coalesce(nullif(coalesce(v.consignment_sale_price,0),0), coalesce(v.purchase_price,0)) AS csp_eff,
    coalesce(v.consignment_commission_rate,7)                                              AS ccr_eff,
    coalesce(v.consignment_end_date, current_date) - v.consign_start_date                  AS rent_days
  FROM vehicles v
  WHERE v.sold_at IS NULL AND coalesce(v.is_archived,false)=false AND v.final_sale_price IS NOT NULL
    AND (v.lifecycle_status IN ('reserved','transferring','pending_delivery')
         OR v.sale_status IN ('deposit_paid','transfer_arranged'))
),
stock AS (  -- 真·可售在庫：剔走已落訂待埋單嗰批，避免同 pipeline 重複計
  SELECT * FROM vehicles
  WHERE sold_at IS NULL AND coalesce(is_archived,false)=false
    AND lifecycle_status NOT IN ('sold','completed','reserved','transferring','pending_delivery')
    AND coalesce(sale_status,'') NOT IN ('deposit_paid','transfer_arranged')
)
SELECT
 (SELECT count(*) FROM sold) sold_cnt,
 (SELECT round(sum(coalesce(sp,0))) FROM sold) revenue,
 (SELECT round(sum(gp)) FROM soldp) gross_profit,                                    -- 未計佣，= app headline
 (SELECT round(0.9*sum(gp)) FROM soldp) net_profit,                                  -- 扣 10% 員工佣
 (SELECT round(sum(gp)) FROM soldp WHERE acquisition_type<>'consignment') own_gross,
 (SELECT round(sum(gp)) FROM soldp WHERE acquisition_type='consignment') consign_gross,
 (SELECT round(sum(coalesce(total_cost,0))) FROM sold WHERE acquisition_type<>'consignment') own_cost,
 (SELECT count(*) FROM pend) reserved,
 (SELECT round(sum(CASE WHEN acquisition_type='consignment'
        THEN (sp-csp_eff)+round(csp_eff*ccr_eff/100.0,2)+coalesce(rent_days,0)*coalesce(daily_rent,0)
        ELSE sp-coalesce(total_cost,0) END)) FROM pend) pipeline,
 (SELECT round(sum(sp)-sum(coalesce(deposit_amount,0))) FROM pend) pend_balance,
 (SELECT round(sum(coalesce(deposit_amount,0))) FROM pend) pend_deposit,
 (SELECT count(*) FROM stock) stock_cnt,
 (SELECT round(sum(coalesce(total_cost,0))) FROM stock WHERE acquisition_type IS DISTINCT FROM 'consignment') stock_capital,
 (SELECT count(*) FROM stock WHERE coalesce(intake_at::date,listed_at::date,created_at::date)<=current_date-60) aged60,
 (SELECT count(*) FROM stock WHERE coalesce(intake_at::date,listed_at::date,created_at::date)<=current_date-90) aged90,
 (SELECT json_agg(o) FROM (SELECT brand||' '||model v, current_date-coalesce(intake_at::date,listed_at::date,created_at::date) d FROM stock ORDER BY 2 DESC LIMIT 3) o) oldest,
 (SELECT round(sum(amount) FILTER (WHERE direction='inflow'  AND payment_date BETWEEN ms AND me_mtd)) FROM payment_records,win) cash_in,
 (SELECT round(sum(amount) FILTER (WHERE direction='outflow' AND payment_date BETWEEN ms AND me_mtd)) FROM payment_records,win) cash_out,
 (SELECT round(sum(coalesce(dealer_balance,0))) FROM vehicles WHERE coalesce(dealer_balance,0)>0 AND dealer_balance_date IS NULL AND coalesce(is_archived,false)=false) ap,
 -- ⚠️ ar_* 口徑同 app 未對齊（見下），報表要標明
 (SELECT round(sum(coalesce(actual_sale_price,0)-coalesce(deposit_amount,0))) FROM vehicles
   WHERE sold_at IS NOT NULL AND coalesce(is_archived,false)=false AND customer_balance_paid_date IS NULL
     AND coalesce(actual_sale_price,0)-coalesce(deposit_amount,0)>0) ar_outstanding,
 (SELECT count(*) FROM vehicles
   WHERE sold_at IS NOT NULL AND coalesce(is_archived,false)=false AND customer_balance_paid_date IS NULL
     AND coalesce(actual_sale_price,0)-coalesce(deposit_amount,0)>0) ar_cnt,
 (SELECT count(*) FROM costgo_renewal_cases,win WHERE coalesce(payment_received_at,renewal_effective_date) BETWEEN ms AND me_mtd AND status IN ('completed','legacy_completed')) ins_cnt,
 (SELECT round(sum(coalesce(gross_premium,0))) FROM costgo_renewal_cases,win WHERE coalesce(payment_received_at,renewal_effective_date) BETWEEN ms AND me_mtd AND status IN ('completed','legacy_completed')) ins_gross,
 (SELECT count(*) FROM soldp WHERE acquisition_type<>'consignment' AND gp<0) neg_margin,
 (SELECT count(*) FROM soldp WHERE gp IS NULL) missing_price,
 (SELECT count(*) FROM vehicles,win WHERE sold_at::date BETWEEN ms AND me_mtd AND lifecycle_status NOT IN ('completed','sold')) excluded_status_cnt,
 (SELECT count(*) FROM costgo_renewal_cases WHERE expiry_date BETWEEN current_date AND current_date+14 AND coalesce(status,'') NOT IN ('completed','legacy_completed','declined','cancelled')) ins_expiring,
 (SELECT count(*) FROM costgo_renewal_cases WHERE expiry_date < current_date AND coalesce(status,'') NOT IN ('completed','legacy_completed','declined','cancelled')) ins_overdue;
```
> 保險佣金 = `ins_gross × 0.15`。

### 口徑依據（點解要咁寫）

1. **自家車未入價 → `gp = NULL`，唔係 0。** 舊版 `coalesce(fsp,asp,0) - total_cost` 令一架未入價嘅車變成「蝕晒成本」。2026-08 嘅 Yamaha XMAX（8/29 成交、`total_cost` 42,877）就係咁假蝕 42,877，係毛利由 139,455 跌到 89,258 嘅**最大單一來源**。`missing_price` 由「淨係報數」改為同分子用同一個定義（`gp IS NULL`），報幾多就真係排除咗幾多。
2. **寄賣公式三截**：`(sp − csp_eff) + round(csp_eff × rate%, 2) + rent_days × daily_rent`。
   - 頭兩截同 `csp_eff` 嘅 fallback，係**照抄 DB function `public.staff_commission_summary`** 嘅 `base`/`calc` CTE 原文（該 function 亦寫住 `WHEN actual_sale_price <= 0 THEN 0`，即係 app 排除未定價寄賣車嘅依據）。
   - 第三截寄存租金：`payment_records` 有 21 筆 `transaction_type='consignment_rental'`，notes 逐筆寫住「寄賣租金結算: 12 日 × HK$70 = HK$840 (2026-04-09 → 2026-04-21)」，日數係兩個日期**直接相減（唔 +1）**，單價 = `daily_rent`。
   - ⚠️ 用 `consign_start_date`，**唔係** `consignment_start_date`（後者全表 NULL）。
   - 舊版淨計 `fsp × rate%`（8月得 4,760），漏咗差價同租金，佣金基數亦用錯。
3. **`lifecycle_status IN ('completed','sold')`**：用 `IN` 唔用 `='completed'`，因為表入面真係有 2 行 `'sold'`（5 月，都有 sold_at）。呢個 filter 順便隔走 8/27 `pending_prep` 嘅 YZF-R6，27 → 26 台，同 app 一致。`excluded_status_cnt` 專門數呢啲「有 sold_at 但狀態未終結」嘅車，提醒同事補狀態。
4. **已成交池刻意唔加 `is_archived` filter**：5 月 31 行已售全部 `is_archived=true`，加咗會抹走成個月。`stock`/`pend` 就照舊要用（嗰啲係在庫車）。
5. **`reserved` / `pipeline` 改為存量口徑**：舊版 `reserved` 淨數 `lifecycle_status='reserved'`（冇隔 archived，多計 2 架）；舊版 `pipeline` 綁咗月窗 `reserved_at BETWEEN ms AND me`，漏晒上月落訂未埋單嘅單（實測得 262,500 / 9台）。改用同一個 `pend` 池後：16 台 / 420,110 / 805,500 / 205,000，四個數全部對正 app。
6. **`stock` 去重**：舊版 `stock_cnt` 55 台入面有 16 台就係 `pend` 嗰批，同時計兩次。`stock_capital` 亦排除寄賣車（寄賣車 `total_cost` 唔係公司資金）。

### 已知未對齊 / 待決（報表要標明，唔好靜靜當啱）
- **應收未收數**：本 SQL `ar_outstanding` = 1,872,714 / 45 台；app 顯示 1,818,342 / 32 台。純用 `vehicles` 數學上去唔到尾數「…342」（全表 售價−訂金 唯一非整百值係 1,514），所以 app 幾乎肯定混咗 `dealer_orders`。**冇夾數**，報表標「口徑同 app 未對齊」。
- **逾期應收**：`customer_balance_due_date` 跑出 652,500 / 14 台，app 係 294,014 / 6 台；app 嘅「7日內到期 2」亦重現唔到（全表冇一張 due_date 落喺未來 7 日）。暫時唔出呢兩項。
- **應付車行**：`vehicles.dealer_balance` = 1,068,800，但 `dealer_orders` 未找數尾數 = 2,428,300，差 1.36M。**要老闆決定邊個權威**，本次未改。
- **`aged60` 質素**：在庫車大量 `intake_at` 係 NULL 要 fallback 落 `created_at`，當中有未到貨嘅 `listed_new` 新車，會假性老化。數字可出但要標註。
- **資料錯誤（要人手修）**：Kawasaki ZX-4RR 兩架（8/2、8/14）嘅 `actual_sale_price` 同 `deposit_amount` 疑似對調輸入（8/2 嗰架售價 98,000 但訂金 99,514）。

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
