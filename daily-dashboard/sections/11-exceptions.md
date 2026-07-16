# 區塊（管理）：🚨 例外 / 風險清單（management by exception）

只列**需要今日處理的異常**。全部正常時顯示「✅ 今日無異常」。

## 檢查項與查詢

**賣車（26king `kpvakfbxpachnotjvnyu`）**
```sql
WITH lcd AS (SELECT (current_date-1) d, date_trunc('month',current_date)::date ms)
SELECT
  -- 蝕本成交（買斷/新車，本月）：⚠️ 必須售價非空，否則空白售價會被當 $0 誤報蝕本
  (SELECT count(*) FROM vehicles,lcd WHERE sold_at::date BETWEEN ms AND d
     AND acquisition_type<>'consignment' AND coalesce(total_cost,0)>0
     AND coalesce(final_sale_price,actual_sale_price) IS NOT NULL
     AND coalesce(final_sale_price,actual_sale_price) < total_cost) AS neg_margin,
  -- 成交但售價未入（資料缺口，非蝕本）：另開一條提示，唔好混入蝕本
  (SELECT count(*) FROM vehicles,lcd WHERE sold_at::date BETWEEN ms AND d
     AND coalesce(final_sale_price,actual_sale_price) IS NULL) AS sold_missing_price,
  -- 壓貨 >90 日
  (SELECT count(*) FROM vehicles WHERE sold_at IS NULL AND coalesce(is_archived,false)=false
     AND lifecycle_status NOT IN ('sold','completed')
     AND coalesce(intake_at::date,listed_at::date,created_at::date) <= current_date-90) AS stuck_90,
  -- 保險續保：14 日內到期未跟進 / 已逾期
  (SELECT count(*) FROM costgo_renewal_cases WHERE expiry_date BETWEEN current_date AND current_date+14
     AND coalesce(status,'') NOT IN ('completed','legacy_completed','declined','cancelled')) AS ins_expiring_14d,
  (SELECT count(*) FROM costgo_renewal_cases WHERE expiry_date < current_date
     AND coalesce(status,'') NOT IN ('completed','legacy_completed','declined','cancelled')) AS ins_overdue;
-- 驗證：真蝕本(售價非空且<成本) 0；成交但售價未入 = 2（SUZUKI DR-Z4SM、YAMAHA BW'S X125）→ 屬資料缺口非蝕本
-- （舊邏輯把售價空白當 $0，會誤報呢兩架為「蝕本 -$49,650 / -$19,000」，已修正）
```
**車房（garage-system `qxxegmvwtndoosqrhyar`）**
```sql
SELECT
  (SELECT count(*) FROM job_orders WHERE coalesce(is_long_stay,false)=true
     AND overall_status::text NOT IN ('已交車','完成','已完成,待交車')) AS long_stay,
  (SELECT count(*) FROM appointments WHERE parts_eta < current_date AND coalesce(parts_arrived,false)=false) AS parts_overdue,
  (SELECT count(*) FROM daily_cash_reports WHERE report_date=current_date-1 AND coalesce(cash_variance,0)<>0) AS cash_variance_flag;
```
**車房未來預約不足（garage-system）** — 產能閒置預警，及早宣傳
```sql
-- config: garage_booking_alert.lookahead_days=7, low_booking_threshold_per_day=4
SELECT scheduled_at::date d, count(*) appts
FROM appointments
WHERE status NOT IN ('cancelled','rejected')
  AND scheduled_at::date BETWEEN current_date AND current_date + 6
GROUP BY 1 ORDER BY 1;
-- 某日 appts < 門檻(4) → flag『未來預約不足』並列出該日。全部達標則不列。
-- 驗證(7/16)：未來7日共13個；7/18(六)=1、7/22(三)=1、7/20/7/21=2 → 多日低於門檻 → 觸發。
```
> 觸發後可接 `garage-fill-slots` skill（偵測→草擬催約訊息+到期保養客名單→人手批准發送）。

**資料新鮮度（信任度）**
```sql
-- Retail Dashboard：SELECT source, max(finished_at) FROM etl_sync_log GROUP BY 1;（表可能空）
-- garage：SELECT * FROM bc_sync_state;（singleton）
```
若任一主來源資料非昨日，列「⚠️ X 資料截至 YYYY-MM-DD（可能不準）」。

## 輸出（Slack 行，每個非零才列）
```
🚨 例外（需今日處理）
• 保險續保 14 日內到期未跟：2 ｜ 已逾期：1 → 今日致電
• 賣車壓貨>90日：{k}（若有，點名）
• 蝕本成交：{k}（真蝕本＝售價非空且<成本；若有列車型+售價/成本）
• ⚠️ 成交但售價未入：{k}（資料缺口，非蝕本；列車型提醒補價）
• ⚠️ 車房未來 7 日預約不足：{列 <門檻 嘅日子} → 建議即時宣傳催約
• 車房滯場：{k} ｜ 零件到貨逾期：{k}
• ⚠️ {來源} 資料停更
（全部 0 → 「✅ 今日無異常」）
```
👉 這區是「行動清單」，management 掃一眼就知今日要 chase 咩。

## 降級
個別查詢失敗只略過該項，不中斷整區。
