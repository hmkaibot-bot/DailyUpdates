# 每日排程：怎麼讓 Dashboard 真正每天 09:30 自動跑

> TL;DR：你在 **Claude Code 雲端 web 環境**裡。容器閒置後會被回收，所以「session 內的 cron」只在 session 活著時有效。要真正無人值守，請用下面的**方案 A（最推薦）**或方案 B。

目標：每天 **09:30（Asia/Hong_Kong）** 自動執行 `daily-dashboard` skill，把五大區塊發到 Slack（目前設定為 DM 給本人）。

---

## ✅ 方案 A（最推薦）：Claude Code web 的「排程觸發器 / Schedule」

這是最省事、且能保留 Slack / Supabase / GitHub 等 MCP 連線授權的做法 —— 因為觸發器會用**這個環境**重新開一個 session，MCP 已經配置好。

設定步驟（在 Claude Code 網頁端操作）：
1. 進入這個專案 / 環境的設定，找到 **Triggers / Schedule（排程）**。
2. 新增一個排程觸發器：
   - 頻率：每天 **09:30**，時區 **Asia/Hong_Kong**
   - 指向分支：`claude/daily-dashboard-slack-yzffu9`（或合併後的主分支）
   - 啟動提示（prompt）：`/daily-dashboard`
3. 儲存。之後每天 09:30 會自動開 session 跑 skill 並發送。

> 文件：https://code.claude.com/docs/en/claude-code-on-the-web （triggers / schedules 章節）

---

## 🔧 方案 B：GitHub Actions 排程（容器之外、完全無人值守）

範本見 `.github/workflows/daily-dashboard.yml`。GitHub 的排程不依賴本機，最穩定。
**但要注意**：headless CI 沒有 claude.ai 的互動式 MCP 授權，所以 Slack / Supabase 必須改用
**存在 GitHub Secrets 的 token / 金鑰**（非 MCP）。需要你先備好：

| Secret | 用途 |
|--------|------|
| `ANTHROPIC_API_KEY` | 跑 Claude Code headless |
| `SLACK_BOT_TOKEN` | 發 Slack 訊息（或改用 `SLACK_WEBHOOK_URL`） |
| `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` | 讀頭盔王資料 |
| `GH_TOKEN`（選用） | 讀更多 repo 的 issue/PR |

備好 secrets 後，workflow 才能真正跑通。未設定前它會直接略過（不會誤報成功）。

---

## ⏱️ 方案 C：本 session 的 cron（即時、但短命）

我已用 `CronCreate`（durable）在這個 session 排了 `09:27`（避開整點 :30 的尖峰）。
限制：
- 只在**這個 session / 容器活著**時才會觸發；容器被回收後失效。
- 即使 durable，Claude Code 的 recurring cron **7 天後自動過期**。

所以方案 C 適合「今天先看到它動起來」，長期請用方案 A 或 B。

---

## 我的建議

1. 先用**方案 A**（web 排程觸發器，prompt 設 `/daily-dashboard`）—— 5 分鐘設定、保留所有 MCP 授權。
2. 若要完全脫離 Claude 平台、純後端跑，再投資**方案 B**（備 secrets）。
