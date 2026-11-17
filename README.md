# OSSLab-agent：多人 AI Agent 與 SSO 協同架構

> **狀態：OSSLab 正在測試上線中。** 現階段先公開架構與可公開的 Chrome 容器；後續預定釋出可重放的 patch file 與安裝／部署 plan，讓人或 AI agent 都能依步驟完成安裝，而不必從零猜測環境差異。

OSSLab-agent 是一套自架的協同架構。我們將 **Lark Suite** 作為唯一的企業 IM 與工作入口，讓 AI agent 像虛擬同事一樣進入團隊原本的對話、資料、文件與審核脈絡。

## 關鍵技術組件

- **Lark Suite (工作入口)**：整合 IM、Mail、Base、Forms、審核與文件。bot 直接在團隊的協作脈絡下接任務、執行、回報，無需另開聊天網站。
- **Magpie (安全性與模型管理)**：在 Agent 與上游模型之間管理流量。
    - **去敏保護**：對接入且已啟用去敏的請求，依內建與自訂規則過濾 PII、帳號、密碼、API key 與 token。
    - **模型路由與替補**：辦公任務可依設定的路由與可用額度，在主模型異常時使用替補來源；需要固定模型身分的開發與審查則指定具名模型，例如 GPT-6 系列。
- **Multica (專案生命週期管理)**：將專案轉化為 issue/PR 協作流。
    - **主動式管理**：AI 主動協助追蹤任務細節、進度，並將變更或待確認事項主動推播回 Lark。
    - **完整紀錄**：保留需求、施工、審查、驗收與部署決策的歷史，確保專案能跨 agent、跨天持續推進。
- **Agent Runtime (執行環境)**：以 `cc-connect` 串接 Codex（GPT-6 系列）、Claude Code 等訂閱制 agent runtime，搭配隔離的 Chrome 容器與已授權工具，執行實際辦公業務。

## 實際應用情境

### 1. 行銷案專案管理 (Multica 驅動)

從活動策劃開始，Multica 記錄行銷目標、內容缺口分析、廣告成效整理、文案/落地頁製作及追蹤轉換數據。Agent 透過 issue 派工，並將階段成果、審核需求透過 Lark 自動通知負責人，確保行銷活動從企劃到追蹤的節奏不中斷。

### 2. 內部系統開發 (Multica + Magpie 驅動)

在需求與驗收條件確立後，Agent 實作功能 (如新增 Odoo 表單欄位、自訂內部工具)、提交 PR，經獨立審查流程後等待驗收。在涉及錯誤紀錄分析或測試開發時，經 Magpie 且已啟用去敏的請求會先按規則過濾機敏憑證，再將錯誤上下文交給模型分析。

## 工作流概覽

```text
Lark IM / Mail / Base (協作入口)
  ├─ 日常交辦 → cc-connect + lark-cli → Agent Server
  └─ 行銷案／內部開發 → Multica (需求／派工／審查／驗收)
                              ↓
                  指定 agent 與訂閱制 runtime
                    ├─ 模型請求 → Magpie (接入時去敏／具名模型或允許的替補) → LLM
                    └─ 工具執行 → 隔離 Chrome／Odoo／內網服務

進度、成果、待確認事項 → Lark／原工單
```

## 核心原則

- **每位同事一個專屬 agent**：每位同事以自己的 Lark 企業帳號作為 SSO 身分，不共用人員帳密；專屬 agent 的 session、browser profile、bot identity、project context 與能力設定獨立，可依同事需求不同。
- **只使用 Lark Suite**：不論人員或 AI agent，訊息、文件、部門公用信箱、email 讀取、Base、表單、審核與 bot 都以同一個 workspace 作為協作與工作入口。Docs、Sheet、Wiki 提供類似 Google Docs／Microsoft Office 的多人文件與試算表協作；不要求團隊改用第二套聊天 UI。
- **Base／Forms 先行**：Base 是可自動化的多維表格與資料流程，而非 XLS 檔。它讓人員、agent、團隊與外圍客戶透過表單、欄位、事件、通知與審核連動；即使只是庫存、訂單或簡單出貨紀錄，也能先快速形成輕量 ERP 流程。需要完整 ERP 控管時，Odoo 才是選配。
- **企業 Mail 是身分基礎**：Lark 現行免費方案提供可自訂的商務 Email；可用自有網域建立個人與部門／功能性信箱。這些企業帳號是 SSO 的身分依據，搭配 Authentik 才能把同一個登入脈絡延伸到自架服務；實際額度與功能依 Lark 方案為準。
- **SSO 優先**：Lark → Authentik → Cloudflare Access／NetBird／維護服務，減少帳密散落與新人上手成本。
- **訂閱制 agent runtime**：AI Agent Server 以 cc-connect 串接 Codex（GPT-6 系列）、Claude Code 等訂閱制 runtime。Magpie 另在模型連線層管理去敏與替補，可按用途配置訂閱、API 或本地模型；費用與可用額度依各來源分開管理。
- **人先解鎖，agent 才接手**：專屬 agent 的 Chrome 使用對應同事的 Web 身分；本人先在 [OSSLab-agent 修改版 KasmVNC Chrome](docker/chrome/README.md) 解鎖 Vaultwarden，agent 才能在已解鎖的工作階段內協助操作，且永遠不取得主密碼。
- **AI 是可控的虛擬助理**：依授權，agent 可直接操作 ERP、讀取與回覆 email、上架、查訪價格，或執行其他辦公室工作流；權限也能細緻限制為查詢、整理或草稿。寄信、付款、刪除、改權限與例外情況仍可要求人工確認或由真人接手。

## 文件

- 從實際營運出發的筆記：[docs/field-notes/](docs/field-notes/README.md)

| 主題 | 說明 |
| --- | --- |
| [01 · 多人 AI agent 應用](docs/field-notes/01-ai-agent-applications.md) | 同事只用 Lark，agent 如何在後端協助並讓真人接手。 |
| [02 · Lark Suite、Mail、Base 與 SSO](docs/field-notes/02-lark-suite-and-sso.md) | 為何 Lark 是唯一工作入口，以及多信箱、表單、Base 與 SSO 如何接在一起。 |
| [03 · 管理與治理](docs/field-notes/03-agent-governance.md) | 多人、多 agent 的分權、審核、留痕與資料隔離。 |
| [04 · 自架平台與 IT 維護](docs/field-notes/04-open-source-it-operations.md) | 身分、VPN、Git、密碼庫、Agent Server、Odoo 選配與維護服務如何分工。 |
| [05 · LarkSuite 作為 AI agent channel 的三方 review](docs/field-notes/05-larksuite-as-ai-agent-channel-review.md) | **定案**：LarkSuite 作 AI agent channel（Free／半年紀錄／Basic ~USD $6；SSO 主因；實作已完成）。 |

## 公開進度

OSSLab 正在測試上線；現階段的文件描述已驗證的方向與尚在整理中的部署基礎。預定把可公開的環境差異整理成 **patch file**，把前置條件、安裝順序、驗證與回復方式整理成 **plan**，讓後續的人或 AI agent 可以照同一條可追溯的路徑部署，而不是重做一次猜測與手動設定。

目前已先列出 [Kasm 使用者身分、CDP 邊界與 VNC secret 輪替 plan](plans/kasm-identity-cdp-and-secret-rotation.md)：先把可直接連到瀏覽器的風險收斂，再以 Authentik／Lark OIDC 平行驗證完整 Kasm Workspaces，讓「哪位同事、哪一個 browser session、哪一個 agent」可被可靠對應。

## License

[MIT License](LICENSE)
