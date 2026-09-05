---
name: garage-daily-close
description: 頭盔王車房每日現金結算（埋數）。當使用者說「埋數」「日結」「車房結算」「結帳」或在收市時要記錄當日現金/收款，使用此 skill。彙整當日各收款方式金額，由人手輸入實點現金後寫入 garage-system 的 daily_cash_reports。
---

# 車房每日結算（埋數）

把當日車房收支寫進 garage-system (`qxxegmvwtndoosqrhyar`) 的 `daily_cash_reports`。
**這是流程缺口的正解**：歷史日結無法回填（實點現金系統沒記錄），只能從今天起每天人手埋數。詳見 `/DATA-GAPS.md` 缺口 3。

## 為什麼要人手

`daily_cash_reports` 的核心是**現金櫃**的進出（cash / FPS / 卡 / 轉帳 各多少）與**實點現金**——這些只有店員收市時數得到，BC 與 payment_receipts 都沒有完整的「收款方式」明細。所以此 skill 是「協助 + 人手輸入」，不是全自動。

## 流程

1. 確認日期（預設今天 Asia/Hong_Kong）。
2. **給參考數**（只讀，幫店員對數，不當作日結值）：
   - 當日已交車工單總額（**2026-09-04 起改用 garage-system；BC 已不再是權威且已停更**）：
     ```sql
     -- 在 garage-system (qxxegmvwtndoosqrhyar) 執行
     -- gateway → daily_cash_reports 五個桶嘅 mapping（payment_method 係 text 唔係 enum）
     SELECT
       round(sum(b.grand_total) FILTER (WHERE pm='cash'),2)                                   AS ref_in_cash,
       round(sum(b.grand_total) FILTER (WHERE pm LIKE 'fps%'),2)                              AS ref_in_fps,
       round(sum(b.grand_total) FILTER (WHERE pm IN ('visa','master','unionpay','amex')),2)   AS ref_in_card,
       round(sum(b.grand_total) FILTER (WHERE pm IN ('payme','alipay','wechat','octopus')),2) AS ref_in_other,
       round(sum(b.grand_total),2)                                                            AS ref_total,
       count(*)                                                                               AS ref_jobs,
       count(*) FILTER (WHERE j.payment_method IS NULL)                                       AS ref_missing_pm,
       -- ⚠️ 唔准慳呢欄：payment_method 係 text，app 隨時可以寫入新 gateway 而唔報錯
       string_agg(DISTINCT pm, ',') FILTER (WHERE pm NOT IN
         ('cash','visa','master','unionpay','amex','payme','alipay','wechat','octopus')
         AND pm NOT LIKE 'fps%')                                                              AS ref_unknown_methods
     FROM v_job_order_billing b
     JOIN job_orders j ON j.id = b.job_order_id
     CROSS JOIN LATERAL (SELECT lower(coalesce(j.payment_method,'(未填)')) AS pm) m
     WHERE (j.delivered_at AT TIME ZONE 'Asia/Hong_Kong')::date = current_date;
     ```
     > 實測 2026-09-01~04：$6,140(4) / $9,658(6) / $12,939(7) / $1,150(4)，四日合計 **$29,887 / 21 張**，
     > 同 `queries.md` Q3b 嘅 MTD 分毫不差 —— 兩條完全唔同嘅 SQL 交叉驗證過。
     >
     > ⚠️ **呢個係參考數，唔等於現金櫃實際進出。** 差異來源：早前已收訂金、未交車先收錢／交車後賒數、
     > 所有支出（garage-system 完全冇來源）。實點現金同支出**依然要人手**。
     > ⚠️ 訂金唔准扮識拆：`deposit_received_at` 全表 0 個有值、`deposit_payment_method` 全 NULL，
     > DB 而家答唔到訂金幾時收、點收。
     > ⚠️ `ref_unknown_methods` 非空 → 有新 gateway，唔好靜靜掉落 in_other，要問清楚先入數。
       AND invoice_date = current_date;
     ```
   - 當日 payment_receipts（若有）：在 garage-system 查 `payment_receipts` WHERE payment_date = current_date。
3. **向使用者詢問**（必填，逐項）：
   - 入帳：`in_cash`, `in_fps`, `in_card`, `in_bank_transfer`, `in_other`
   - 支出：`out_cash`, `out_fps`, `out_card`, `out_bank_transfer`, `out_other`
   - `physical_cash_count`（實點現金）
   - 備註 `notes`（選填）
4. **寫入**（garage-system，upsert：同一天可重跑覆蓋）：
   ```sql
   INSERT INTO public.daily_cash_reports
     (id, report_date, closed_at,
      in_cash, in_fps, in_card, in_bank_transfer, in_other,
      out_cash, out_fps, out_card, out_bank_transfer, out_other,
      cash_in_total, cash_out_total,
      physical_cash_count, cash_variance, notes)
   VALUES
     (gen_random_uuid(), :report_date, now(),
      :in_cash, :in_fps, :in_card, :in_bank_transfer, :in_other,
      :out_cash, :out_fps, :out_card, :out_bank_transfer, :out_other,
      (:in_cash + :in_fps + :in_card + :in_bank_transfer + :in_other),
      (:out_cash + :out_fps + :out_card + :out_bank_transfer + :out_other),
      :physical_cash_count,
      (:physical_cash_count - :in_cash + :out_cash),  -- 差異 = 實點 −（現金應有 = 現金入 − 現金出）
      :notes)
   ON CONFLICT (report_date) DO UPDATE SET
      closed_at = excluded.closed_at,
      in_cash = excluded.in_cash, in_fps = excluded.in_fps, in_card = excluded.in_card,
      in_bank_transfer = excluded.in_bank_transfer, in_other = excluded.in_other,
      out_cash = excluded.out_cash, out_fps = excluded.out_fps, out_card = excluded.out_card,
      out_bank_transfer = excluded.out_bank_transfer, out_other = excluded.out_other,
      cash_in_total = excluded.cash_in_total, cash_out_total = excluded.cash_out_total,
      physical_cash_count = excluded.physical_cash_count,
      cash_variance = excluded.cash_variance, notes = excluded.notes,
      updated_at = now();
   ```
   - `net_cash` 為 generated 欄（cash_in_total − cash_out_total），不需手填。
5. **回報**：寫入後讀回該列，向使用者確認 net_cash、cash_variance；若 variance 不為 0 提醒對數。

## 原則

- **絕不**用 payment_receipts 自動填日結（會錯 2–3 個數量級，見 DATA-GAPS）。
- 沒有人手輸入的收款明細與實點現金，就不要寫入；寧可提醒「今日未埋數」。
- 每天一列（`report_date` 唯一），重跑同日為覆蓋更新。
