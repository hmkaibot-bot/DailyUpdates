---
name: daily-dashboard
description: 產生頭盔王每日 dashboard 並發送到 Slack daily 頻道。當使用者要求「跑每日 dashboard」「daily update」「每日報告」，或由 cron 每天 09:30 自動觸發時使用。彙整 GitHub 熱門項目、生產力建議、skill 建立工作流程、GitHub 停滯工作提醒、頭盔王生意報表五大區塊。
---

# Daily Dashboard（頭盔王每日儀表板）

每天**把資料查一次**，組裝成**多份報告**發到各自 Slack 頻道（管理層綜合版 + 零售/賣車/車房 部組版）。設定來源：`config/dashboard.config.json`。

## 執行流程（總覽）—— compute-once → render-many

1. 讀 `config/dashboard.config.json`。
2. **把所有資料查一次**：**逐條照跑 `daily-dashboard/queries.md` 內鎖定 SQL（Q1/Q1b/Q2/Q3…），不得即興改寫欄位或口徑**。數字只算一次，供各報告共用 → 各頻道一致。任一資料源失敗 → 該 block 標「⚠️ 今日無法取得」，不中斷。查詢報錯 → 先修 queries.md，唔好 run 時亂改。
3. **逐份組裝並發送**：對 `delivery.reports[]` 每一項，用其 `blocks` 清單，依 `daily-dashboard/report-layouts.md` 對應版面組成該份訊息（繁體中文），發到該 `channel_id`。
   - 以本人身分發送（Alex，需為該頻道 member）。`send_mode: direct`=直接發；`draft`=先發草稿驗收。
   - 某頻道發送失敗 → fallback DM 給 `delivery.fallback_dm_user_id` 並註明，**其餘頻道照發**。
   - **部組頻道只出自己嘅數**（唔夾雜其他線/集團 GP）；公司級 block 只出 management。
   - **管理版每日只出精簡前頁**（`blocks`）；`weekly_blocks`（GitHub/社群/生產力/Skill/停滯）只在 `weekly_day`（預設 Mon）當日附加。判斷今日星期幾以決定是否加週報段。
4. 把當日各線數字寫回快照表 `daily_dashboard_log`（PM 專案 `saxtopyysysqzvcltash`）：`INSERT ... ON CONFLICT(report_date) DO UPDATE`，`data` 存 jsonb，`send_status` 記各頻道發送結果。—— 鎖數/可審計/隔日比對。

## 🛡️ 穩定性規則（防「空白數字」— 必守）

過往 autonomous run 出現賣車/車房空白，根因 = Supabase MCP 連線間歇性中斷，後面查嘅專案回空。以下**必守**：
1. **開跑前**先做一次輕量 Supabase 連線測試（例：`SELECT 1`）。連唔到 → 等 10 秒重試，最多 3 次。
2. **每條業務 query**（尤其賣車 26king、車房 garage-system）若回 null 或報錯 → **等 5 秒重試，最多 3 次**（連線通常幾秒重連）。賣車/車房與零售**同等優先**，不得因排後而放棄。
3. **永不出空白報告**：某線業務數字（金額/台數）重試後仍取不到 → **唔好發嗰份空白**；改發一句「⚠️ {線} 數據暫時取不到，已排程稍後補發」，並喺 `daily_dashboard_log.notes` 記低，同時再排一次重試（如 5 分鐘後）。寧可遲，唔好白。
4. **一致性**：所有頻道用**同一次計算結果**（先全部算好、寫快照，再逐頻道發）；發送前確認關鍵數字（GP、四線 MTD、賣車淨利、車房 MTD）皆非 null。
5. 任一頻道發送失敗 → fallback DM，其餘照發，並喺 `send_status` 記低。

## 🎯 準確性規則（防「數字對唔上 app」— 必守）

老闆會攞 Slack 個數同各系統嘅 app 對數，對唔上就等於報表冇用。以下**必守**：
1. **口徑以 app 為準**：賣車以 26KING app「總覽 / 本月至今」為權威。`queries.md` Q2 已對齊並實測重現 app 全部 headline，**逐條照跑，唔好即興改**。改動前必讀 Q2 下面「口徑依據」。
2. **NULL ≠ 0**：任何「售價/金額未入」嘅記錄，**唔准 `coalesce(...,0)` 之後攞去減成本**——咁會變成假蝕本。一律排除出利潤計算，另計入例外清單。（2026-08 就係一架未入價嘅車令毛利少報 $42,877。）
3. **累計 vs 流量分開**：MTD 累計（台數/收入/利潤/現金流）用 `me_mtd`（含今日，同 app 一致）；單日/流量（昨日營收、日比較）先用 `me`（LCD）。
4. **利潤兩個都報**：`gross_profit`（未計佣，= app headline）同 `net_profit`（×0.9，= 追目標用）。淨報一個必定有人覺得唔準。
5. **對唔上就講**：某個數同 app 口徑未對齊（例：賣車應收），**照出但標明「口徑同 app 未對齊」**，唔准夾數、唔准靜靜當啱。寧可講清楚，唔好扮準。
6. **車房營收唯一來源 = garage-system**（2026-09-04 老闆決定，見 `queries.md` Q3b）。
   **絕不准**用 BC（`bc_sales_invoices`）補車房數、絕不准同 garage-system 相加。
   歷史比較欄 `comparable=false` 就出「未夠比較期」，**唔准為咗個報表好睇而 fallback BC**。
7. **車房毛利直接叫「毛利」**（收入 − 零件成本）。師傅人工唔入 GP，係公司會計口徑，唔好加「未計人工」但書。但舊制 48% 假設同新實數**唔具可比性**，唔好攞嚟做趨勢比較。

## 區塊總覽

| # | 區塊 | 來源 | 說明檔 |
|---|------|------|--------|
| 1 | 🪖 集團生意報表 + 月目標追數 + 集團總GP | Supabase | `05-helmet-king.md`（零售/車房）＋ `07-vehicle-sales.md`（賣車）＋ `08-rental.md`（租車） |
| 2 | 💵 現金流 & 應收應付（管理） | Supabase | `09-cashflow-ar-ap.md` |
| 3 | 📦 賣車庫存健康（管理） | Supabase | `10-inventory-health.md` |
| 4 | 🚨 例外/風險清單（管理） | Supabase | `11-exceptions.md` |
| 5 | ⏳ 停滯工作提醒 | GitHub issues / PR | `04-stalled-work.md` |
| 6 | 🔥 GitHub 熱門項目 | github.com/trending | `01-github-trending.md` |
| 7 | 💬 社群熱門（AI + 電單車） | WebSearch | `06-community-trends.md` |
| 8 | 🚀 生產力建議 | AI 綜合產生 | `02-productivity.md` |
| 9 | 🛠️ Skill 工作流程 | 輪播指南 | `03-skill-workflow.md` |

> 對象 = management。順序由「要決策的數字」到「資訊性」：生意/目標 → 現金 → 庫存 → 例外 → 其餘。
> 區塊 1 是「集團生意」四線（零售/車房/賣車/租車）+ 保險/旅行團 + 月目標追數 + 集團總 GP。
> 區塊 2–4 為管理層新增（現金/庫存/例外），詳見各 section 的已驗證 SQL。

## 原則

- **簡潔可行動**：每個區塊控制在 3-6 行，重點是「我今天該做什麼」，不是資料傾倒。
- **失敗不中斷**：任一資料源掛掉就降級該區塊，整份 dashboard 照樣送出。
- **不外洩機密**：Slack 連結勿放敏感 query string；生意數字只發到指定頻道。
- **可調**：所有來源、門檻、開關都在 config，不要把這些寫死在邏輯裡。

## 附：（已移除）車房發票同步 BC → garage-system

2026-09-04 起車房營收唯一權威 = garage-system 交車工單，**呢步已刪**。三個理由：

1. **上游已死 + 靜靜講大話**：BC 全表停喺 2026-08-31，同步窗係 `invoice_date > last_date`，
   所以每日補 0 行但**永遠唔會報錯**，skill 照報「同步完成」。
2. **保留係主動有害**：BC pipeline 係可以修嘅。一旦修好補回 9 月 GARAGE 發票，呢步會**自動**
   把 9 月 BC 發票寫入 garage-system 個 `invoice_summary` —— 即係喺**權威 DB 內部**同時有
   「9 月 BC 鏡像」同「9 月交車工單」，正正就係雙計陷阱搬咗入屋。降級成「失敗唔理」消除唔到呢個風險。
3. **違反本 skill 唯讀原則**：呢步係成個 daily-dashboard 唯一寫入 DB 嘅地方。刪咗之後 skill 全程唯讀。

> ⚠️ **唔好 DROP `invoice_summary` 張表** —— 7,013 行係 2023-11 → 2026-08 嘅歷史鏡像，
> BC 死咗之後佢係 garage-system 內部唯一嘅歷史營收備份。保留表、唔准當營收來源、唔准同 job_orders 相加。

## 手動測試

直接呼叫此 skill 即可產生並（依設定）發送或建立草稿。第一次建議用草稿模式確認格式。
