# 每日排程：怎麼讓 Dashboard 真正每天 09:30 自動跑

> TL;DR：你在 **Claude Code 雲端 web 環境**裡。容器閒置後會被回收，所以「session 內的 cron」只在 session 活著時有效。要真正無人值守，請用下面的**方案 A（最推薦）**或方案 B。

目標：每天 **09:30（Asia/Hong_Kong）** 自動執行 `daily-dashboard` skill，把五大區塊發到 Slack（目前設定為 DM 給本人）。

---

## ✅ 方案 A（已採用）：Claude Code web 的「排程觸發器 / Schedule」

最省事、且保留 Slack / Supabase / GitHub 等 MCP 授權 —— 觸發器用**這個環境**重新開 session，MCP 與 `.claude/settings.json` 的 always-allow 都已就緒，會自動跑完並發送、不需人手批准。

**設定步驟（網頁端，約 3 分鐘）：**
1. 在 Claude Code 網頁端開啟這個專案 / 環境，找 **Schedule / Triggers（排程觸發器）**。
2. 新增排程：
   - 頻率：**每天 09:30**，時區 **Asia/Hong_Kong**
   - 分支：`claude/daily-dashboard-slack-yzffu9`（或先 merge 到 main 再指 main）
   - 啟動 prompt（建議用明確版）：
     ```
     執行 /daily-dashboard：讀 config/dashboard.config.json，產生六區塊（GitHub熱門 / 生產力 / Skill工作流 / 停滯工作 / 集團生意(零售·車房·賣車·租車·保險·旅行團 + 月目標追數 + 集團GP) / 社群熱門），營收報最後完整日，發送到 Slack #每日匯報 (C0BDLSKM2FQ)。任一區塊失敗降級不中斷。
     ```
3. 儲存。之後每天 09:30 自動送到 #每日匯報。

**前置確認（都已完成）：**
- skill `.claude/skills/daily-dashboard/` 已在分支上 ✅
- `.claude/settings.json` always-allow 已設（自動執行不卡權限）✅
- 發送目標 = #每日匯報 `C0BDLSKM2FQ`（config delivery）✅

> 文件：https://code.claude.com/docs/en/claude-code-on-the-web （triggers / schedules 章節）
> 注意：觸發器跑哪條分支就讀哪條的 skill/config。若想用 main，先把本分支 merge 入 main。

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
