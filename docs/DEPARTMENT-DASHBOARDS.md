# 部組獨立 Dashboard 設計（零售 / 賣車 / 車房）

目標：除咗 management 綜合版（#每日匯報），再為各部組出**專屬、聚焦、可行動**嘅每日報告，各自入自己頻道。

## 一、架構（建議）：一個 skill、多份報告（compute-once → render-many）

**唔好**為每個部組複製一條 skill / 一條 SQL。改成**設定驅動嘅 `reports[]`**：
1. skill 每次 run **先把所有資料查一次**（零售/車房/賣車/租車/保險/現金/庫存/例外）。
2. 然後**逐份報告**（management / 零售 / 賣車 / 車房）用各自嘅 block 清單，由同一批數字**組裝並發到各自頻道**。
3. 一條 routine、一次 run、數字算一次 → 各頻道**數字必然一致**（唔會 A 頻道 $196k、B 頻道 $213k）。

> 這正好同「鎖 SQL + 每日快照」相輔相成：多份報告更加需要單一真相來源。建議一齊做。

### config 結構（草案）
```jsonc
"reports": [
  { "key":"management", "channel_id":"C0BDLSKM2FQ", "audience":"管理層",
    "blocks":["group-business","group-gp","cashflow","inventory","exceptions",
              "stalled","github","community","productivity","skill"] },
  { "key":"retail", "channel_id":"C08JNU5NQ5N", "audience":"零售團隊",
    "blocks":["retail-daily","retail-target","retail-top","retail-lowstock","retail-actions"] },
  { "key":"sales", "channel_id":"C03LALA3CDR", "audience":"賣車團隊",
    "blocks":["sales-target","sales-pipeline","inventory-health","sales-cashflow",
              "insurance","sales-exceptions","sales-actions"] },
  { "key":"garage", "channel_id":"C02HY4J8VLN", "audience":"車房",
    "blocks":["garage-target","appointments-capacity","garage-exceptions","daily-close","garage-actions"] }
]
```
`delivery`（現時單一頻道）→ 由 `reports[]` 取代；每份有自己 `channel_id` + `blocks`。

## 二、各頻道內容（聚焦該部組 + 行動導向）

### #零售（Shopify）
- 昨日營業額（訂單/AOV）vs 前日、上週同日
- **MTD vs 140萬 target**：達成% / 應達% / 距標 / 餘下每日需追 / 預估埋月 / vs 上月同期
- Top 賣品、退款/取消
- **低庫存暢銷品**（補貨提示，接返現有 #補貨 文化）
- 行動：今日追數目標 + 要補嘅貨
- （Tier2）廣告 ROAS（Meta，待數據）、轉換率（Shopify analytics）

### #賣車（26King）
- **本月成交台數 + 淨利(毛利−10%佣) vs 25萬 target** + 追數
- Pipeline：預留台數、開單 pipeline $
- **在庫健康**：在庫台數、鎖資金、車齡>60/90、最舊壓貨點名
- 現金流：入/出 MTD、應付車行(AP)、應收(待確認)
- **保險(CostGo 續保)**：宗數/保費/佣金
- 例外：真蝕本、**成交售價未入**、續保到期未跟
- 行動：追 5 台預留成交、補售價、chase 續保

### #車房（BC GARAGE + garage-system）
- 昨日營業額(BC GARAGE 發票) vs 前日、上週同日
- **MTD vs 40萬 target** + 追數 + vs 上月同期
- **今日/明日預約 vs 產能(8 師傅)**、待交車積壓
- 例外：SLA 逾期工單、長期滯場車、零件到貨逾期
- 現金日結（靠 `garage-daily-close` 埋數）
- 行動：今日排程、chase 逾期、補日結

> **跨部門/公司級 block**（集團 GP、GitHub 熱門、社群、生產力、Skill、停滯工作）**只放 #每日匯報**，部組頻道保持聚焦。

## 三、資料分隔（管理考量）
- 各部組**只睇自己嘅數**（唔會喺 #零售 見到 #賣車 嘅毛利/GP）→ 天然分權。
- **營運頻道 vs 管理頻道**：#車房、#零售舖面 係前線營運台（好多同事），放**財務/target/margin** 未必適合。
  可考慮部組財務版入 **#車房管理 / #零售管理**（私人），營運提示（今日預約/補貨）先入營運台。**此點需老闆定。**

## 四、前置（要做）
1. **確認每個部組要邊條頻道**（見上「營運 vs 管理」）。
2. **把發送 bot 加入每個目標頻道**（尤其公開頻道 #賣車 #車房，bot 要係 member 先發到）。
3. 首次用**草稿模式**發畀老闆過目版面，OK 先轉正式。

## 五、排程
- 預設：**同一條 routine**、09:30 一次過出 4 份。
- 若部組想唔同時間（如車房想早啲睇今日預約），可拆多條 routine，或 config 加 per-report 時間（進階）。

## 六、Rollout 步驟
1. 確認頻道對應 + 加 bot 入頻道。
2. refactor config：`delivery` → `reports[]`；skill 改為 compute-once → 逐份 render/send。
3. 用草稿試發 3 個部組版 → 老闆驗收。
4. 轉正式；一齊上「鎖 SQL + 每日快照」保一致。
