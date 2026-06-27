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
   - 當日 BC GARAGE 開單總額：
     ```sql
     -- 在 Retail Dashboard (myrangmxyjamsupbxbba) 執行
     SELECT count(*) invoices, sum(total_amount_incl_tax) total
     FROM bc_sales_invoices
     WHERE dimension1_code='GARAGE' AND number LIKE 'SI-%' AND status<>'Canceled'
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
