# 區塊 4：⏳ 停滯工作提醒

**目標**：偵測 GitHub 上停滯的工作（指派給你的 issue、開著沒動的 PR、等你 review 的東西），提醒進度並給下一步建議。

## 做法

1. 讀 `config.sections.stalled_work`：`repos`、`stale_days`（預設 3 天）、`watch` 類別。
2. 對每個 repo 用 GitHub MCP：
   - `list_issues`（state=open，assignee=本人）→ 找 `updated_at` 超過 `stale_days` 沒動的。
   - `list_pull_requests`（state=open）→ 找你開的、或 review 卡住的 PR。
   - `pull_request_read` 看 CI / review 狀態。
3. 每筆算出「停滯天數」= 今天 − 最後更新日。
4. 與前一日 dashboard log 比對（Supabase `PM.daily_dashboard_log`），標出「又多卡一天」的項目。

## 每項輸出

- `#編號 標題`（連結）— 停滯 N 天
- 卡點推測（等 review？CI 紅？等你回覆？）
- 👉 一句具體下一步建議

## 排序

依停滯天數由多到少，最多列 5 筆，其餘用「另有 N 項較新的待辦」帶過。

## ⚠️ 權限限制

目前 GitHub MCP 授權僅涵蓋 **hmkaibot-bot/dailyupdates** 一個 repo。
若要監測使用者其他 repo 的停滯工作，需先擴大 GitHub 授權範圍，再把 repo 加進
`config.sections.stalled_work.repos`。在那之前，此區塊只覆蓋已授權的 repo，並在
dashboard 中註明涵蓋範圍，避免讓人誤以為「全部都看過了」。
