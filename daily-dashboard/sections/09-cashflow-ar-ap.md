# 區塊（管理）：💵 現金流 & 應收應付

management 第一關心：今天現金夠不夠、要追/要找哪幾筆。

## 來源與算法
**賣車現金流（26king-trading `kpvakfbxpachnotjvnyu`，live）**
```sql
WITH lcd AS (SELECT (current_date-1) d, date_trunc('month',current_date)::date ms)
SELECT
  round(sum(amount) FILTER (WHERE direction='inflow'  AND payment_date BETWEEN ms AND d)) cash_in_mtd,
  round(sum(amount) FILTER (WHERE direction='outflow' AND payment_date BETWEEN ms AND d)) cash_out_mtd
FROM payment_records, lcd;
-- 驗證(6/27)：入 $254,900 / 出 $780,256（入貨期淨流出屬正常）
```
**應付（AP）— 欠車行尾數**
```sql
SELECT round(sum(coalesce(dealer_balance,0))) ap, count(*) n
FROM vehicles WHERE coalesce(dealer_balance,0)>0 AND dealer_balance_date IS NULL AND coalesce(is_archived,false)=false;
-- 驗證(6/27)：$446,600
```
**應收（AR）— ⚠️ 待確認**：`vehicles.customer_balance` 目前全空，AR 算不準。
替代算法（成交/預留但未找尾數）：`final_sale_price − coalesce(customer_deposit,0)`，條件 `customer_balance_paid_date IS NULL AND sale_status IN ('deposit_paid','completed')`。上線前請老闆確認尾數記在哪欄。

**其他線（次要 / 待補）**
- 保險 outstanding：`insurance_monthly_summary().customer_outstanding / qbe_outstanding`（insurance_policies 停更，現多為 0）。
- 車房按金在手：`appointments`/`job_orders` 的 `deposit_amount`（`deposit_status` 已收）。
- 零售（Shopify）：多為線上預付，AR 極少；`financial_status='pending'` 可視作待收。
- 車房現金日結：靠 `garage-daily-close` skill 人手埋數後才有。

## 輸出（Slack 行）
`💵 現金流(賣車)：本月 入 $X / 出 $X　｜ 應付車行 $X（N 筆）｜ 應收 $X（待確認）`
👉 決策：今日要追邊幾筆尾數、現金是否夠出糧/入貨。

## 降級
任一查詢失敗 → 該項標「⚠️ 無法取得」，不中斷。
