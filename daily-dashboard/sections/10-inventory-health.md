# 區塊（管理）：📦 賣車庫存健康（壓貨 / 鎖住資金）

賣車是集團壓最多資金的地方——management 要看**鎖住幾多錢、壓貨幾耐**，及早清滯銷、釋放現金。

## 來源：26king-trading `kpvakfbxpachnotjvnyu` `vehicles`
未售 = `sold_at IS NULL AND coalesce(is_archived,false)=false AND lifecycle_status NOT IN ('sold','completed')`
```sql
WITH stock AS (
  SELECT *, current_date - coalesce(intake_at::date, listed_at::date, created_at::date) AS days_in_stock
  FROM vehicles
  WHERE sold_at IS NULL AND coalesce(is_archived,false)=false AND lifecycle_status NOT IN ('sold','completed')
)
SELECT
  count(*) stock_cnt,
  round(sum(coalesce(total_cost,0))) capital_locked,
  count(*) FILTER (WHERE days_in_stock>=60) aged_60,
  count(*) FILTER (WHERE days_in_stock>=90) aged_90,
  round(avg(days_in_stock)) avg_days
FROM stock;
-- 驗證(6/27)：在庫 55 架、鎖住資金 $2,889,501、>60日 4、>90日 0、平均 21 日（健康）
```
最舊壓貨清單（給管理層點名清貨）：
```sql
SELECT brand, model, reg_no,
  current_date - coalesce(intake_at::date,listed_at::date,created_at::date) AS days,
  total_cost, target_sale_price, lifecycle_status
FROM vehicles
WHERE sold_at IS NULL AND coalesce(is_archived,false)=false AND lifecycle_status NOT IN ('sold','completed')
ORDER BY days DESC LIMIT 5;
```

## 輸出（Slack 行）
`📦 賣車庫存：在庫 {n} 架 ｜ 鎖住資金 $X ｜ 車齡>60日 {k} 架 ｜ 平均 {d} 日`
（若 aged_60>0，列出最舊 1-2 架點名）
👉 決策：邊幾架要劈價/推廣，避免資金與牌費長期鎖死。

## Tier 2（可後加）
- 零件/配件滯銷：garage `parts_inventory`、`dead_stock_reviews`、`bc_inventory`（28k）。
- 零售缺貨：Shopify 暢銷品低庫存/斷貨（`shopify_inventory`）。

## 降級
查詢失敗 → 標「⚠️ 無法取得」，不中斷。
