# 區塊：🏍️ 賣車（Vehicle Sales）

高單價、低頻、有庫存與毛利、分「買斷 / 寄賣 / 新車(dealer order)」三種——看**台數 + 利潤 + 在庫漏斗**，不看單日流水。

**來源（權威）**：`26king-trading`（`kpvakfbxpachnotjvnyu`）的 `vehicles`（成交主表）、`dealer_orders`（新車）、`payment_records`（現金流）。
> ⚠️ BC `CARSHOP` 維度已**停在 2026-02-11**，僅作歷史，**不用作當前數字**。

## 已確認對齊（與老闆口徑一致）
- **今月成交台數** = `vehicles` where `sold_at` 落在本月 → 已驗證 = 25 ✅
- **客人預留** = `vehicles` where `lifecycle_status='reserved'` → 已驗證 = 4 ✅
- 成交按 `acquisition_type` 拆：買斷 buyback / 新車 dealer_order / 寄賣 consignment。

## ⏳ 待老闆確認的兩個金額口徑（不要猜）
老闆要看：**已實現利潤** 與 **本月開單**。系統內無對應 view/function，需照前端算法定義：
- `已實現利潤`：哪些單算「已實現」（已找尾數 `customer_balance_paid_date`？已過戶 `transfer_date`？），利潤 = 售價 − 哪些成本（`total_cost`？`purchase_price+prep_cost`？是否含 `licence_fee`/`first_reg_tax`？），範圍（本月 or 累計）。
- `本月開單`：開單 = 本月開出發票之單，是利潤還是金額，哪個子集（新車 dealer_order？訂金？開單未收？）。
> 參考：本月 sold_at 25 台「售價−total_cost」毛利 = $233,722（買斷13=120,322 / 新車10=79,800 / 寄賣2=33,600）；已找尾數(10 台)利潤 = $122,672。皆 ≠ 老闆數字 → 待定義後改這裡的 SQL。

## 在庫漏斗（lifecycle_status）
`pending_intake / pending_prep / pending_listing / listed / listed_new / reserved / transferring / sold / completed`
```sql
SELECT lifecycle_status, count(*) FROM vehicles WHERE coalesce(is_archived,false)=false GROUP BY 1 ORDER BY 2 DESC;
```

## 輸出（Slack 行，金額待口徑確認後填）
`🏍️ 賣車：本月成交 25 台 ｜ 已實現利潤 $208,564 ｜ 本月開單 $69,650 ｜ 客人預留 4 ｜ 在庫漏斗(待整備/已上架/已留訂)`

## 降級
查詢失敗 → 標「⚠️ 今日無法取得」，不中斷整份報告。
