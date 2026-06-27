# 區塊 6：💬 社群熱門（AI + 電單車）

**目標**：每天撈出 AI 與電單車（電單車/摩托）社群的當週熱門話題，各 2-3 條，附「對頭盔王/26King 的關聯」。

## 做法

1. 用 `WebSearch`（**不要**用 WebFetch 抓 reddit.com / old.reddit.com / redlib 鏡像——本環境代理封鎖 reddit，鏡像常 503）。
2. 兩個主題各跑 1-2 個查詢，例：
   - AI：`reddit r/LocalLLaMA r/artificial r/singularity trending this week`、`AI 熱門 討論 本週`
   - 電單車：`reddit r/motorcycles trending this week new bikes gear helmet`、`motorcycle 2026 trend ADV electric helmet`
3. 從結果歸納「主題」（非逐帖榜，因 Reddit 直連被擋）。每條註明來源連結。

## 每項輸出

- 🤖 AI：2-3 條當週話題（一句話）＋ 來源連結
- 🏍️ 電單車：2-3 條當週話題（新車/技術/護具/市場）＋ 來源連結
- 💡 一句「對你生意的關聯」（例：ADV 當道→相關頭盔/護具需求；自動變速/電動化→車房技能與零件庫存）

## 限制（誠實標註）

- 來源為**網路搜尋歸納的主題**，非 Reddit 即時逐帖排行（reddit 直連在本環境被代理封鎖）。
- 若日後要真 Reddit 榜：需配 Reddit API 憑證（n8n / edge function），見 config `sections.community_trends.note`。

## 降級

搜尋無有效結果時，該主題標「⚠️ 今日無新話題」，不中斷整份 dashboard。
