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
 (SELECT round(sum(total_price)) FROM shopify_orders,win WHERE cancelled_at IS NULL AND created_at::date BETWEEN (ms-interval '1 month')::date AND (me-interval '1 month')::date) r_lastmonth;
```
> ⚠️ **Q1 已冇車房**。2026-09-04 起車房營收改由 garage-system 出（見 Q3b），呢度只剩零售。
> 呢條 Q1 跑喺 Retail Dashboard，**唔好順手把車房查詢加返落嚟**。

## Q1c —（僅過渡期，可選）舊制 BC 車房月度參考

**唔准入主行、唔准做 ▲▼ 比較、唔准同 Q3b 相加。** 只可以放喺 #車房 腳註，標明「舊制(BC)參考」。
窗口硬 clamp 喺 `2026-08-31`：BC pipeline 一旦修好補回 9 月，呢條都唔可能同 Q3b 雙計。
**2026-10-01 之後刪咗成段。**

```sql
-- Retail Dashboard myrangmxyjamsupbxbba
SELECT to_char(invoice_date,'YYYY-MM') m, count(*) n, round(sum(total_amount_incl_tax)) amt
FROM bc_sales_invoices
WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled'
  AND invoice_date <= '2026-08-31'::date            -- 硬閘，防同 Q3b 雙計
  AND invoice_date >= date_trunc('month',current_date - interval '3 month')::date
GROUP BY 1 ORDER BY 1;
```
> 實測（2026-09-04）：6月 141/$343,503 · 7月 147/$320,791 · 8月 176/$363,665。

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

## Q3b — 車房營收 + 零件貢獻毛利（garage-system `qxxegmvwtndoosqrhyar`）★ **車房營收/毛利唯一權威**

2026-09-04 老闆決定：**不再以 BC 為準，全改用 garage-system**。呢條取代咗原本 Q1 入面嘅車房 BC 半段。
實測重現 26 維修部 app：`g_mtd_cnt 21 · g_mtd 29,887.00 · g_parts_cost 8,560.25 · g_gp 21,326.75 · g_gp_pct 71.4`。

> ⚠️ **`g_gp` 係「零件貢獻毛利」，唔係 all-in 毛利率。** 工時佔收入 57.5% 而公式**完全冇師傅人工成本**，
> 所以 71.4% 係上限值，唔可以攞嚟同舊制 48% 假設直接比較（48% 反而接近 all-in）。報表一律叫
> 「零件貢獻毛利（未計人工）」。

```sql
-- Q3b — 車房營收 + 零件貢獻毛利（garage-system qxxegmvwtndoosqrhyar）★ 車房營收/毛利唯一權威
-- 2026-09-04 實跑重現：g_mtd_cnt 21 · g_mtd 29887.00 · g_parts_cost 8560.25 · g_gp 21326.75 · g_gp_pct 71.4
-- 設計原則：
--   (1) 冇任何 hardcode 日期／金額。切換日由 min(delivered_at) 自己讀出（g_src_start），
--       所以 9/8、10/1、跨年通通自動啱，將來有數自動出，唔使改檔。
--   (2) 冇 fallback BC。歷史窗冇數 = NULL + cnt 0 + comparable=false，誠實報「未夠比較期」。
--   (3) winx LEFT JOIN billed ON true：billed 空表都保證回一行（唔會成段車房消失）。
--       因為有幽靈 NULL 行，所有計數一律 count(b.d)/count(*) FILTER，冇一個 bare count(*)。
WITH win AS (                       -- 通用月窗（同 queries.md 其他 Q 一致）
  SELECT
    CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 month')::date
         ELSE date_trunc('month',current_date)::date END AS ms,
    CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date
         ELSE (current_date-1) END AS me,        -- LCD：單日指標
    CASE WHEN date_part('day',current_date)=1 THEN (date_trunc('month',current_date)-interval '1 day')::date
         ELSE current_date END AS me_mtd         -- 含今日：MTD 累計（同 app 對齊）
),
winx AS (                           -- 上月同期窗 + 新來源起點（切換日，由資料讀出，非硬編）
  SELECT w.*,
    (w.ms - interval '1 month')::date AS pms,
    CASE WHEN w.me_mtd = (w.ms + interval '1 month' - interval '1 day')::date
         THEN (w.ms - 1)                                              -- 報整月 → 對足上個整月
         ELSE least((w.me_mtd - interval '1 month')::date, w.ms - 1)  -- 月中 → 對上月同 day-of-month，短月 clamp
    END AS pme,
    (SELECT min(delivered_at AT TIME ZONE 'Asia/Hong_Kong')::date
       FROM job_orders WHERE delivered_at IS NOT NULL) AS src_start
  FROM win w
),
cost AS (                           -- ⚠️ unit_cost 先係成本；unit_price 係賣俾客嘅價，撈亂毛利會由 71.4% 變 7.7%
  SELECT t.job_order_id,
         sum(p.quantity*coalesce(p.unit_cost,0))                         AS parts_cost,
         count(*)                                                        AS parts_lines,
         count(*) FILTER (WHERE p.unit_cost IS NULL)                     AS parts_lines_no_cost,
         sum(p.quantity*p.unit_price) FILTER (WHERE p.unit_cost IS NULL) AS no_cost_retail_amt
  FROM job_tasks t
  JOIN job_task_parts p ON p.task_id=t.id AND coalesce(p.status,'')<>'取消'  -- coalesce：NULL status 唔好靜靜剔走成行
  GROUP BY 1
),
qual AS (                           -- 未定價工時 = 收入被低報
  SELECT t.job_order_id, count(*) FILTER (WHERE coalesce(t.customer_price,0)=0) AS tasks_unpriced
  FROM job_tasks t GROUP BY 1
),
billed AS (                         -- 已交車 = 已出單，基準 job_orders.delivered_at（HKT）
  SELECT (j.delivered_at AT TIME ZONE 'Asia/Hong_Kong')::date AS d,
         b.grand_total                       AS rev,
         coalesce(c.parts_cost,0)            AS pcost,
         coalesce(c.parts_lines,0)           AS plines,
         coalesce(c.parts_lines_no_cost,0)   AS plines_nocost,
         coalesce(c.no_cost_retail_amt,0)    AS nocost_amt,
         coalesce(q.tasks_unpriced,0)        AS tasks_unpriced,
         coalesce(b.quotation_labour,0)      AS quo_labour,
         lower(coalesce(b.payment_method,'(未填)')) AS pm
  FROM v_job_order_billing b
  JOIN job_orders j ON j.id=b.job_order_id
  LEFT JOIN cost  c ON c.job_order_id=b.job_order_id
  LEFT JOIN qual  q ON q.job_order_id=b.job_order_id
  WHERE j.delivered_at IS NOT NULL
)
SELECT
 -- ===== 單日（LCD = me）。cnt=0 即「當日冇交車」，唔係 $0 生意 =====
 round(sum(b.rev) FILTER (WHERE b.d = w.me),2)                               AS g_yday,
 count(b.d)       FILTER (WHERE b.d = w.me)                                  AS g_yday_cnt,
 round(sum(b.rev) FILTER (WHERE b.d = w.me - 1),2)                           AS g_prev,
 count(b.d)       FILTER (WHERE b.d = w.me - 1)                              AS g_prev_cnt,
 round(sum(b.rev) FILTER (WHERE b.d = w.me - 7),2)                           AS g_wow,
 count(b.d)       FILTER (WHERE b.d = w.me - 7)                              AS g_wow_cnt,
 ((w.me - 7) >= w.src_start)                                                 AS g_wow_comparable,
 -- ===== MTD（me_mtd 含今日）=====
 round(sum(b.rev)           FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd),2)  AS g_mtd,
 count(b.d)                 FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd)     AS g_mtd_cnt,
 round(sum(b.pcost)         FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd),2)  AS g_parts_cost,
 round(sum(b.rev - b.pcost) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd),2)  AS g_gp,
 round(100.0 * sum(b.rev - b.pcost) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd)
       / nullif(sum(b.rev)          FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd),0),1) AS g_gp_pct,
 count(DISTINCT b.d)        FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd)     AS g_mtd_days,
 round(sum(b.rev) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd)
       / nullif(count(DISTINCT b.d) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd),0),2) AS g_rev_per_day,
 -- ===== 上月同期（切換日前必然 NULL；絕不 fallback BC）=====
 round(sum(b.rev)           FILTER (WHERE b.d BETWEEN w.pms AND w.pme),2)    AS g_lastmonth,
 count(b.d)                 FILTER (WHERE b.d BETWEEN w.pms AND w.pme)       AS g_lastmonth_cnt,
 round(sum(b.rev - b.pcost) FILTER (WHERE b.d BETWEEN w.pms AND w.pme),2)    AS g_gp_lastmonth,
 (w.pms >= w.src_start)                                                      AS g_lastmonth_comparable,
 -- ===== 資料鮮度 =====
 max(b.d)                                                                    AS g_maxdate,
 (current_date - max(b.d))                                                   AS g_stale_days,
 w.src_start                                                                 AS g_src_start,
 -- ===== 資料品質（毛利可信度，非零就要喺報表出註）=====
 count(*) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd AND b.plines = 0)          AS g_no_parts_cnt,        -- 純工時單，正常
 count(*) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd AND b.plines_nocost > 0)   AS g_missing_cost_cnt,    -- 零件冇入成本 → GP 偏高
 round(sum(b.nocost_amt) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd),2)         AS g_missing_cost_amt,    -- 該批零件嘅售價值
 count(*) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd AND b.tasks_unpriced > 0)  AS g_unpriced_task_jobs,  -- 有 $0 工時 → 收入低報
 count(*) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd AND b.quo_labour > 0)      AS g_quo_labour_jobs,     -- >0 → quotation 模組啟用，防工時雙計
 count(*) FILTER (WHERE b.d BETWEEN w.ms AND w.me_mtd AND b.pm = '(未填)')       AS g_missing_pm_cnt,      -- 收款方式未填
 -- ===== 窗口回聲（審計用）=====
 w.ms AS g_ms, w.me AS g_me, w.me_mtd AS g_me_mtd, w.pms AS g_pms, w.pme AS g_pme
FROM winx w LEFT JOIN billed b ON true
GROUP BY w.ms, w.me, w.me_mtd, w.pms, w.pme, w.src_start;
```

### 口徑依據
- **收入** = `v_job_order_billing.grand_total`，基準 `job_orders.delivered_at`（轉 HKT）。
  實測 21 張已交車工單 `payment_received_at` 100% 有值，兩個基準等價。
- **零件成本** = `job_task_parts.quantity × unit_cost`，排除 `status='取消'`（用 `coalesce(status,'')` 免得 NULL status 被靜靜剔走）。
  ⚠️ `unit_price` 係**賣俾客嘅價**、`unit_cost` 先係**成本**；撈亂毛利率會由 71.4% 變 7.7%。
- **切換日唔係硬編**：`g_src_start = min(delivered_at)::date`（今日 = 2026-09-01）。
  `g_wow_comparable` / `g_lastmonth_comparable` 由佢自動算，所以 9/8 有週比較、10/1 有月比較，**唔使改任何檔**。
- `winx LEFT JOIN billed ON true`：`billed` 空表都保證回一行，車房整段唔會喺報表消失。因為有幽靈 NULL 行，
  所有計數用 `count(b.d)` 或 `count(*) FILTER`，**唔准用 bare `count(*)`**。
- 時區用 `AT TIME ZONE 'Asia/Hong_Kong'`：DB session 係 UTC，將來有車喺 HKT 凌晨交會計錯日。
  同理 cron 09:30 HKT = 01:30 UTC 安全，**但唔好把 cron 搬到 08:00 HKT 之前**。

### 版面規則（report-layouts 必須跟）
| 情況 | 出咩 |
|---|---|
| `comparable = false` | 「—（新來源 {g_src_start} 上線，未夠比較期）」**絕對唔准出 ▼100% 或當 $0 計跌幅** |
| `comparable = true` 而 `cnt = 0` | 「$0（當日冇交車）」 |
兩者意思完全唔同，混為一談就會把「冇生意」報成「新系統」。

### 資料品質欄位（非零就要喺報表出註）
| 欄 | 意思 | 2026-09-04 實測 |
|---|---|---|
| `g_no_parts_cnt` | 純工時單（正常） | 3 |
| `g_missing_cost_cnt` / `g_missing_cost_amt` | 零件冇入 `unit_cost` → **毛利偏高** | 4 張 / 涉零件售價 $3,940 |
| `g_unpriced_task_jobs` | 有 `customer_price = 0` 嘅工時 → **收入低報** | 9 |
| `g_quo_labour_jobs` | >0 即 quotation 模組啟用 → **防工時雙計** | 0 |
| `g_missing_pm_cnt` | 收款方式未填 | 0 |
| `g_stale_days` | 資料幾多日冇更新 | 0 |

> 把 `g_missing_cost_cnt` 嗰批按有價零件嘅成本率（68.5%）補返，毛利會由 71.4% 跌到約 **62%**。
> 「同 26 維修部 app 分毫不差」只代表 app 有同一個窿，唔代表個數係 all-in。

### 為咗換來源而必須知嘅三個 `v_job_order_billing` 地雷
1. **工時冇 status filter** —— 零件有 `status <> '取消'`，但 labour 係 `sum(customer_price)` 冇任何條件。
   21 張已交車工單嘅 46 條 task 有 34 條 status 仲係「待開始」，佢哋嘅 $14,685 工時照樣入咗收入
   → task status 冇人維護，任何靠 status 嘅例外偵測（SLA）都信唔過。
2. **10/46 條 task `customer_price = 0`** —— 舊制唔影響報表，換來源之後**直接扣減營收**。
3. **`quotation_labour` 係加喺 `tasks_labour` 之上** —— 現時 `quotations` 表空所以 = 0，一旦啟用就會**工時雙計**。

### ⚠️ 兩個口徑量緊唔同嘢（切換前必讀）
BC 8 月 176 張 / $363,665 / 均價 $2,066；garage-system 9 月頭 4 日 21 張 / $29,887 / 均價 $1,423。
單量係 BC 嘅 78%，但**中位數只有 44%**（$597 vs $1,360）—— 唔係做少咗客，係少咗一類生意：
- BC 8 月 176 張入面 **57 張（32%）完全冇工時行 = 純貨品/配件單**，佔全月約 25% 收入。
- 21 張已交車工單實測：`appointment_id` **全部非空（0 張 walk-in）**、無 parts-only 單、無 26 Pack 使用。
→ 櫃檯落單／配件零售／26 Pack 預售**結構上入唔到 `job_orders`**。呢啲收入計唔計入「車房營收」要老闆定；
未定之前，`monthly_targets.garage` 嘅 $400k **唔可以攞嚟計達成%**（見 config `basis_note`）。

## 追數/GP 計法
見 `sections/05-helmet-king.md`（通用追數公式 + 集團總 GP 五+租屋 分項相加，必填）。目標見 `config.monthly_targets` / `group_profit`。

## 規則
- **逐條照跑，唔准即興改寫欄位/口徑**；發現查詢報錯先修此檔，唔好喺 run 時亂改。
- 跑完把結果套入 `report-layouts.md` 各版面。
