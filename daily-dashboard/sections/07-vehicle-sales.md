# 區塊：🏍️ 賣車（Vehicle Sales）

高單價、低頻、有庫存與毛利、分三種收入模型——看**台數 + 純利 + pipeline + 在庫漏斗**，不看單日流水。

**來源（權威）**：`26king-trading`（`kpvakfbxpachnotjvnyu`）：`vehicles`（成交主表）、`dealer_orders`（新車）、`payment_records`（現金流）。
> ⚠️ BC `CARSHOP` 維度已**停在 2026-02-11**，僅歷史，不用作當前數字。

## 已界定算法（經老闆確認）

**已實現純利（已成交）** —— 認列以**成交日 `sold_at`** 為準（上月下訂、本月成交＝算本月）。本月至今 = `sold_at` ∈ [月初, 最後完整日]。
- 買斷 (buyback) / 新車 (dealer_order)：純利 = `coalesce(final_sale_price,actual_sale_price)` − `total_cost`
- 寄賣 (consignment)：**只計佣金** = `coalesce(consignment_sale_price,final_sale_price)` × `consignment_commission_rate/100`（車非我們所有，佣金才是真利潤）

**本月開單（pipeline / 潛在純利）** —— 本月**下訂但未成交**之單，成交時才轉「已實現」；本月下訂、下月成交則歸下月。
- 條件：`sold_at IS NULL` 且 (`lifecycle_status='reserved'` 或 `sale_status='deposit_paid'`)，且 `coalesce(reserved_at,customer_deposit_date)` ∈ 本月。
- 純利用同上公式（未成交用 `target_sale_price`/`estimated_sale_price` 估）。

**客人預留** = `vehicles` where `lifecycle_status='reserved'`。

## 主查詢（26king-trading）
```sql
WITH lcd AS (SELECT (current_date-1) d, date_trunc('month',current_date)::date ms)
SELECT
 (SELECT count(*) FROM vehicles,lcd WHERE sold_at::date BETWEEN ms AND d) AS sold_cnt,
 (SELECT round(sum(CASE WHEN acquisition_type='consignment'
        THEN coalesce(consignment_sale_price,final_sale_price,actual_sale_price,0)*coalesce(consignment_commission_rate,0)/100.0
        ELSE coalesce(final_sale_price,actual_sale_price,0)-coalesce(total_cost,0) END))
   FROM vehicles,lcd WHERE sold_at::date BETWEEN ms AND d) AS realized_profit,
 (SELECT round(sum(CASE WHEN acquisition_type='consignment'
        THEN coalesce(consignment_sale_price,final_sale_price,target_sale_price,0)*coalesce(consignment_commission_rate,0)/100.0
        ELSE coalesce(final_sale_price,target_sale_price,estimated_sale_price,0)-coalesce(total_cost,0) END))
   FROM vehicles,lcd WHERE sold_at IS NULL AND (lifecycle_status='reserved' OR sale_status='deposit_paid')
        AND coalesce(reserved_at::date,customer_deposit_date::date) BETWEEN ms AND d) AS pipeline_profit,
 (SELECT count(*) FROM vehicles WHERE lifecycle_status='reserved') AS reserved_now;
```
驗證值（2026-06，至 6/27）：成交 24 台、已實現純利 $196,824（買斷+新車 191,322＋寄賣佣金 5,502）、pipeline $34,650（3 台）、預留 4。

## 月目標 + 追數
目標：**純利 $250,000/月**（見 config `monthly_targets.vehicle_sales`）。套用通用追數公式（見 `05-helmet-king.md`）。

## 在庫漏斗
```sql
SELECT lifecycle_status, count(*) FROM vehicles WHERE coalesce(is_archived,false)=false GROUP BY 1 ORDER BY 2 DESC;
```

## 輸出（Slack 行）
`🏍️ 賣車：本月成交 {台} ｜ 已實現純利 ${realized} / 目標 25萬（{%}）距標 ${gap}，餘 {n} 日需 ${/日} ｜ 開單 pipeline ${pipeline} ｜ 預留 4`
