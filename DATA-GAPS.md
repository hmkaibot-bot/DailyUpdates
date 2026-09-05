# 頭盔王資料缺口調查報告（2026-06-27）

唯讀調查（6 個 agent、150 次查詢）＋對抗式驗證，後續經使用者核准執行修復。
專案：garage-system = `qxxegmvwtndoosqrhyar`、Retail Dashboard = `myrangmxyjamsupbxbba`。

## ✅ 執行狀態（2026-06-27，經使用者核准）

| 缺口 | 動作 | 狀態 |
|---|---|---|
| 1 invoice_summary | 回填 2026-04-09~06-26 共 **399 列 / $910,129** 進 garage-system | ✅ 已完成（總列 6293→6692、最新日期→06-26、重複 0） |
| 1 持續同步 | 同步步驟內建進 `daily-dashboard` skill「步驟 0」（每日跑、免憑證） | ✅ 已建（隨每日 dashboard 執行） |
| 2 final_price | 套用 trigger `trg_job_orders_compute_final_price`（結案且全任務已定價才寫、不覆蓋、不阻擋） | ✅ 已套用 garage-system |
| 3 daily_cash_reports | 建 `garage-daily-close` skill（往後人手埋數） | ✅ 已建（人手觸發） |

> 回填採 `NOT EXISTS(invoice_number)` 去重，可安全重跑。final_price trigger 可一鍵移除（見該段 SQL 註解）。

## 最重要結論

車房真正的營收**權威來源**是 Retail Dashboard 的 `bc_sales_invoices`（`dimension1_code='GARAGE'`），它**完整且更新到昨天**。
`invoice_summary` / `daily_cash_reports` / `job_orders.final_price` 是車房 app 自己的營運表——
**dashboard 報表不應依賴它們**，應直接取 BC GARAGE。已據此調整 `daily-dashboard/sections/05-helmet-king.md`。

---

## 缺口 1：invoice_summary 停在 2026-04-08 ✅ 可安全回填

- **根因**：整張表 6293 列的 `created_at` 全落在 2026-04-08 12:54:05–10（5 秒內）＝一次性人手批次匯入；無 trigger、無 cron、無 edge function 維護它，所以匯完就停了。
- **真實來源**：`bc_sales_invoices`（GARAGE、`number LIKE 'SI-%'`），明細在 `bc_invoice_lines`。
- **缺口窗 2026-04-09 ~ 06-26**：399 列（排除 1 列 Canceled）、**910,129 HKD**、1,796 明細行。
- **對抗驗證（未被推翻）**：
  - `invoice_number` 唯一（6293/6293 distinct），既有表無任何 ≥ `SI-24-36280` 之列 → 用 `NOT EXISTS(invoice_number)` 防重覆，重跑不會重複。
  - 金額採 `total_amount_incl_tax`：以唯一一筆 incl≠excl 的歷史發票 `SI-24-17013`（既有表存 351 = incl）證實原匯入用含稅，選擇正確。
  - 重疊樣本逐筆吻合（`SI-24-36245`=1180/2行、`SI-24-36251`=3210/6行）。
  - 客戶對應：190 個客戶中 187 個可對到 `customer_profiles.id`；3 個新客戶（C022167/C022234/C022239）→ `customer_id` 設 NULL（欄位允許）。
- **限制**：來源在另一專案，無法單一 SQL 跨專案 JOIN。需「從 A 讀 → 組 INSERT 寫進 B」兩步（或設 postgres_fdw/dblink）。回填只補短期缺口，**根因（無自動同步）未除**，要另設定期同步否則會再脫節。

### 回填計劃（待你核准，尚未執行）

步驟一 — 在 **Retail Dashboard** 執行（唯讀取數）：
```sql
SELECT i.number AS invoice_number, i.invoice_date,
       i.customer_number AS bc_customer_no, i.customer_name,
       i.total_amount_incl_tax AS total_amount,
       (SELECT COUNT(*) FROM public.bc_invoice_lines l WHERE l.invoice_number = i.number) AS line_count
FROM public.bc_sales_invoices i
WHERE i.dimension1_code = 'GARAGE' AND i.number LIKE 'SI-%'
  AND i.status <> 'Canceled'
  AND i.invoice_date BETWEEN DATE '2026-04-09' AND DATE '2026-06-26'
ORDER BY i.invoice_date, i.number;
```

步驟二 — 在 **garage-system** 逐列 INSERT（以 `customer_profiles` 對 `customer_id`、`NOT EXISTS` 防重覆）：
```sql
INSERT INTO public.invoice_summary
  (id, bc_customer_no, customer_id, customer_name, invoice_number, invoice_date, total_amount, line_count, created_at)
SELECT gen_random_uuid(), :bc_customer_no,
       (SELECT p.id FROM public.customer_profiles p WHERE p.bc_customer_no = :bc_customer_no),
       :customer_name, :invoice_number, :invoice_date, :total_amount, :line_count, now()
WHERE NOT EXISTS (SELECT 1 FROM public.invoice_summary s WHERE s.invoice_number = :invoice_number);
```

---

## 缺口 2：job_orders.final_price 全為 NULL ❌ 不能可靠回填

- **根因**：86 列 `final_price` 全空（含已交車/完成單）。沒有任何 trigger/cron/function 會算它；員工結算時也沒手動填。計價資料分散在 `job_tasks.customer_price`（半數為 0＝未定價）與 `job_task_parts`，`quotations` 模組未啟用、`bc_sales_order` 全 NULL。
- **對抗驗證（推翻回填）**：
  - 用推導公式只有 30 列「安全子集」、合計 **46,694 HKD**；但 BC GARAGE 同期 = 2026-05 **392,579** + 2026-06 **292,164 HKD** → job_orders 只是極小且不對應的片段，量級差 10 倍以上。
  - `final_price` 語意未定（是否扣 `deposit`？安全子集 9 列 deposit 等於整單金額）。
  - 與 BC 無 join key，無法逐筆對帳；20 筆結案單有 17 筆連 `delivered_at` 都空。
- **正確修法（流程，非回填）**：
  1. 員工在 UI 補錄各任務金額後再結算；
  2. 加 BEFORE UPDATE trigger / 在結算 edge function：狀態轉「完成/已交車」時自動 `final_price = 工時+零件+傷損−折扣`，並擋下仍有未定價任務的結案。
  3. 先釐清 `final_price` 是「工單總額」還是「淨應收（扣訂金）」。

---

## 缺口 3：daily_cash_reports 空表 ❌ 不能回填（流程補）

- **根因**：2026-05-18 migration 只建了空表＋即時彙總 view `v_daily_cash_summary`，預期前端「埋數」UI 每日由店員 INSERT 並輸入實點現金——但該流程從未被使用（0 列），亦無任何後端自動寫入。
- **對抗驗證（推翻回填）**：
  - `payment_receipts` 只有 7 列（3 筆供應商付款、3 筆退款、1 筆遞延現金），**不是**車房收款來源；真實收款在 BC GARAGE（同期 153 張 / 365,682 HKD）。
  - 用 payment_receipts 回填，net_cash 會錯 2–3 個數量級；唯一一筆「入帳」還是 `is_deferred` 預付。
  - `physical_cash_count` / `cash_variance` 是人手實點，系統無記錄、**無法重建**。
- **正確修法（流程）**：往後每日由人手埋數（可做成「車房每日結算」skill），輸入實點現金才寫入正規列；只需歷史摘要的話，直接查 `v_daily_cash_summary`，不要寫假日結。

---

## 我能做 vs 需要你

| 缺口 | 我能直接修？ | 需要你 |
|---|---|---|
| invoice_summary | ✅ 可回填 399 列（已驗證） | 核准寫入；選跨專案方式（腳本逐列 / 設 fdw）；是否一併建定期同步 |
| final_price | ❌ 只能改流程 | 決定語意；要不要我寫「結案自動計價」trigger（往後生效，不回填歷史） |
| daily_cash_reports | ❌ 只能改流程 | 要不要我做「車房每日結算」skill（往後人手埋數） |
| **dashboard 報表** | ✅ 已改用 BC GARAGE 權威來源 | 無 |

---

# 更新（2026-09-04）：車房營收/毛利來源已切換

## ✅ 缺口 2/3 實質解決 —— 車房有真實毛利了

garage-system 嘅**交車工單流程 2026-09-01 上線**（最早 `job_orders.delivered_at` = 9/1），
而家可以直接算出**真實毛利**，唔使再靠「營業額 × 48% 假設」：

```
收入   = v_job_order_billing.grand_total（基準 delivered_at）
零件成本 = sum(job_task_parts.quantity × unit_cost)，排除 status='取消'
毛利   = 收入 − 零件成本
```

2026-09-04 實測 **21 單 / 收入 $29,887.00 / 零件成本 $8,560.25 / 毛利 $21,326.75 / 71.4%**，
同 26 維修部 app 儀表板分毫不差。鎖定 SQL 見 `daily-dashboard/queries.md` Q3b。

> ⚠️ `job_task_parts.unit_price` 係**賣俾客嘅價**，`unit_cost` 先係**入貨成本**。撈亂會令毛利變 0。
> ⚠️ `invoices` / `invoice_lines` 兩張表係**空**嘅，所以 `v_invoice_gross_profit` 查唔到嘢——唔好用嗰個 view。

## 🚨 新缺口：BC 同步 2026-08-31 起中斷

`bc_sales_invoices` **全表**（唔止 GARAGE 維度）`max(invoice_date)` 同 `max(created_at)` 都停喺 **2026-08-31**，
到 9/4 已經斷咗 4 日。呢個唔係以往嗰種一兩日 lag，係同步壞咗，**要修**。

影響：2026-09 之後 BC 完全冇車房數。暫時靠 garage-system 工單頂住，但：
- BC 亦係**零售以外**其他 BC 維度數據嘅來源，斷咗會影響其他報表；
- 9 月嗰 21 張工單 `bc_sales_order` / `bc_pushed_at` **全部 NULL**，即係仲未推去 BC。

## ⚠️ 兩個來源絕不可相加

| | 6月 | 7月 | 8月 | 9月 |
|---|---|---|---|---|
| BC GARAGE `bc_sales_invoices` | 141 / $343,503 | 147 / $320,791 | 176 / $363,665 | **0** |
| garage-system `job_orders` | — | — | — | 21 / $29,887 |

時間上係**接力**唔係重疊。現行規則：**2026-09 起以 garage-system 為準**，8 月及之前用 BC。
一旦 BC 修好並補回 9 月，同一筆生意會兩邊都有 —— **只可揀一個來源**。
跨越切換點做月比較（vs 上月）時，報表必須標明「來源已切換，非同口徑」。

---

# 決定（2026-09-04）：不再以 BC 為準，車房全改用 garage-system

老闆指示：**「現在開始不再以 bc 為準，全改用 garage-system」**。已全面切換，逐檔改動見 commit。

## 切換後嘅唯一權威
```
車房營收 = v_job_order_billing.grand_total，基準 job_orders.delivered_at（HKT）
零件成本 = job_task_parts.quantity × unit_cost，排除 status='取消'
毛利    = 收入 − 零件成本  ← 叫「零件貢獻毛利（未計師傅人工）」
```
鎖定 SQL = `daily-dashboard/queries.md` **Q3b**。BC 只剩 `Q1c` 一條 ≤2026-08-31 嘅月度腳註參考，
窗口硬 clamp，2026-10-01 後可刪。

## 歷史比較：誠實留白，唔補數
garage-system 冇 2026-09-01 之前嘅營收歷史（工單流程嗰日先上線）。已查證**冇任何非 BC 嘅替代來源**：

| 候選 | 否決理由 |
|---|---|
| `invoice_summary` | **就係 BC 鏡像**（用咗等於冇切換）；漏數，8 月逐日對數 7 日短數，**最爛嗰日 08-28 錄 1 張 $150、BC 實際 8 張 $10,723（短 98.6%）**；冇成本欄位 |
| `_archive_*_20260831` | 337 張入面 177 張 `delivered_at` 全部同一日（批量回填造假）、`out_date` 100% NULL、`_archive_job_task_parts` **根本冇 `unit_cost` 欄**、重建收入只覆蓋同期 BC 嘅 **14.5%** |
| `payment_receipts` / `v_daily_cash_summary` | 得 7 行、停 2026-06-23、內容係供應商付款＋退款 |
| `v_simple_pl` | 每行 revenue = 0（建喺空嘅 `invoices` 上） |
| `quotations` / `customer_package_usages` / `daily_cash_reports` | 空表 |

**做法**：`Q3b` 用 `g_src_start = min(delivered_at)`（**唔係硬編**）自動算出
`g_wow_comparable` / `g_lastmonth_comparable`。false 就出「未夠比較期」，
**絕不 fallback BC、絕不當 $0 計跌幅**。`g_wow` 2026-09-08 自動有數、`g_lastmonth` 2026-10-01 自動有數，唔使改檔。

> 附帶一提：2026-09-04 係星期五，上週同日 = **08-28**，正正就係鏡像最爛嗰一日。
> 用 `invoice_summary` 補呢欄會印出「vs 上週同日 **+8,526%**」。

## 🔴 三個必須向老闆提嘅風險

**R1 — $400k 月目標喺新口徑下已經失效。** $400k 按 BC 訂（8月 176張/$363,665/均$2,066）；
garage-system 均價 $1,423、4 日 $7,472/日 → 埋月推算約 $194k–$249k，**最樂觀都只係目標六成**。
而且呢 4 日樣本（9/1二–9/4五）**完全冇星期六**。→ 未重校前 #車房 唔准出「達成%／落後」。

**R2 — 兩個口徑根本唔係量同一樣嘢。** 單量差距細（5.25/日 vs 6.77/日 = 78%）但**中位數只有 44%**
（$597 vs $1,360）→ 唔係做少咗客，係少咗一類生意。BC 8 月 176 張入面 **57 張（32%）完全冇工時行
= 純貨品/配件單**，佔全月約 25% 收入。而 21 張已交車工單 `appointment_id` **全部非空（0 張 walk-in）**。
→ 櫃檯落單／配件零售／26 Pack 預售**結構上入唔到 `job_orders`**。呢三條計唔計入「車房營收」要老闆定。

**R3 — 71.4% 唔係 all-in 毛利率。**
(a) **工時完全零成本**：工時 $17,185 佔收入 57.5%，公式冇任何師傅人工 → 等於當 8 個師傅 100% 毛利。
    舊嗰個 48% 假設反而接近 all-in，**兩個數放埋一齊比較係錯嘅**。
(b) 4 張單零件冇入 `unit_cost`（涉售價 $3,940）；按有價零件成本率 68.5% 補返，毛利率跌到約 **62%**。
(c) **會遮住營收暴跌**：舊法 8/30 快照 gp_garage $175,558；新法若營收跌 38%，GP 分項只跌 8.5%
    → 老闆睇 GP 行**睇唔出出事**。$1M 集團 GP 目標當初就係按「BC $365k × 48% ≈ $175k」校出，**分母一換目標本身失效**。

## 🟠 其他要跟嘅事
- **Routine prompt 唔喺 repo 入面**，仲寫死住「車房(garage-system + Retail Dashboard 的 BC GARAGE)」，
  而且 routine clone default branch → **未 merge 入 main 嘅改動唔會生效**。要用 `update_trigger` 改（唔好 delete+recreate，會失去 run history）。
- **BC 其實冇死，死嘅係一條 sync**：garage-system 自己嗰條 `bc_invoice_status` 仲有 9/1–9/3 嘅新發票（102 行），
  而 Retail Dashboard 嘅 `bc_sales_invoices` 停喺 8/31。即係 **Retail Dashboard 嗰條 pipeline 壞咗**，可以修。
  建議照修 —— 零售報表仲要用，`bc_purchase_invoices`（供應商發票／真實入貨成本）亦已斷供，
  而佢正正係將來想補返「真實零件／人工成本」嘅唯一來源。
- **切走車房之後冇人再會發現 BC 死咗**。前車可鑑：`bc_sales_invoices` 嘅 **CARSHOP 維度已經死咗 7 個月**
  （停 2026-02）一直冇人 flag。→ `11-exceptions` 應加一條獨立嘅 BC 停更例外。
- **兩個現有「資料新鮮度」檢查全部係盲嘅**：`etl_sync_log` 0 行；`bc_sync_state` 根本唔係入庫監控
  （係另一條 AI 對帳 job，日日報 success）。所以斷咗 4 日先發現。
- **`v_job_order_billing` 三個計數地雷**：工時冇 status filter（46 條 task 有 34 條仲係「待開始」但 $14,685 工時照入收入）；
  10/46 條 `customer_price = 0`（收入低報）；`quotation_labour` 加喺 `tasks_labour` 之上，
  `quotations` 一旦啟用就**工時雙計**。Q3b 已加 `g_unpriced_task_jobs` / `g_quo_labour_jobs` 預警。
- **`daily_dashboard_log` 冇 schema_version / source**，車房 key 名亦已亂（garage / garage_revenue / garage_bc …），
  加上今次口徑切換冇標記 → 歷史值將來會唔可解讀。趁得 8 行加欄好平（要人手 `apply_migration`，唔喺 skill 唯讀範圍）。

## ✅ 老闆拍板（2026-09-04，同日稍後）—— 三個未決問題已有答案

**1. 車房月目標：$400k → `$250,000`**（`monthly_targets.garage.amount`）
「暫時預」= 先用住，觀察一兩個月後可再校。目標而家同數字**同一口徑**（都係 garage-system 工單），
所以報表**可以正常出達成%**，之前嗰條「唔准出達成%」嘅限制取消。

**2. 櫃檯落單／配件零售／26 Pack —— 唔計入車房營收**
即係維持 garage-system 工單口徑，唔使為咗追返 BC 嗰 32% 純貨品單而改系統。
老闆補充：**26 Pack 套票收入係 realize（客人使用）之後先確認**，所以工單口徑本身就會少計 ——
呢個係**已知並接受**嘅特性，唔係 bug，唔使補數。
→ 上面 R2 從此唔再係「未決風險」，係已接受嘅口徑定義。

**3. 師傅人工唔喺 Gross Profit 內處理**
公司會計口徑：師傅人工屬**營運開支**，唔係 COGS。所以「收入 − 零件成本」**已經係完整嘅毛利**，
報表直接叫「**毛利**」，唔使加「未計人工」但書。
→ 上面 R3(a) 撤回 —— 71.4% 唔係「假嘅高」，係正常口徑。
→ R3(b) **仍然有效**：部分零件未入 `unit_cost`（實測 4 張單、涉售價 $3,940）會令毛利略偏高，
   叫同事入齊成本價即可解決；Q3b `g_missing_cost_cnt` 會繼續監察。
→ R3(c) 保留一半：舊制 48% 假設同新實數**唔具可比性**，唔好攞嚟做趨勢比較。
   另外 $1M 集團 GP 目標當初按「車房 BC $365k × 48% ≈ $175k」校出，車房月營收目標已由 $400k 降到 $250k，
   **$1M 可能要相應下調** —— 呢項仍待老闆定。

### 仍然未決
- 集團 GP 月目標 $1M 要唔要跟住下調？
- BC pipeline（Retail Dashboard `bc_sales_invoices`，8/31 起壞）要唔要修？零售報表仲要用。
- Routine prompt（唔喺 repo 內）仲寫死住「車房(garage-system + BC GARAGE)」，要用 `update_trigger` 改。
- `main` branch 落後，routine 若 clone default branch 則 repo 改動唔會生效。
