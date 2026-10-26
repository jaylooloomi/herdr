# Herdr Windows Agent Bundle：安裝與實作規格

此文件供實作 agent 與使用者交接。這是整合方案，不是已完成的一鍵安裝產品。

## 目前狀態

| 項目 | 狀態 |
|---|---|
| Herdr Windows Alt+V 圖片修正 | 已實作；commit `22069c46`；真實剪貼簿讀取與暫存測試通過 |
| 修正版用戶端連回既有 0.9.5 server | 已驗證；沒有停止執行中的 agent |
| 實體按鍵到 Claude 最終附件呈現 | 尚待使用者確認，不能等同上述程式測試 |
| Claude、Codex、Agy 三個 CLI | 本機已安裝；新機自動安裝器尚待實作 |
| OpenCode CLI | 已納入預設安裝規格；本次未執行本機安裝或連線驗證 |
| QuotaDeck Windows 用量面板 | 本機已驗證三個 provider 可讀取數據；不是 Agent Usage 的 Windows 移植 |
| Agent Usage | 已 fork 並以 submodule 納入本分支；Windows 移植尚未完成 |
| 一鍵安裝、doctor、更新與回退腳本 | 本文件定義規格，尚未實作或發佈 |

## Repository 與分支

- Herdr fork：`jaylooloomi/herdr`，整合分支 `integration/windows-agent-bundle`。
- 圖片修正分支：`fix/windows-local-clipboard-image-paste`，保留獨立維護。
- Agent Usage fork：`jaylooloomi/herdr-agent-usage`，分支 `integration/windows-support`。
- `bundled/agent-usage` 為 git submodule，鎖定一個具體 commit；branch 設定僅供維護者更新，安裝不得追蹤浮動 HEAD。
- 上游 Herdr PR #5575 因貢獻者名單限制被關閉；本套件由 fork 發行，不宣稱官方版本。

## 整合方式

採單一發行包與 PowerShell 安裝器，自動註冊並設定 Agent Usage 外掛。Herdr 核心維持 Rust，Agent Usage 維持 Go/provider registry，不把兩個程式硬改成同一個核心。

第一版正式目標是在 Windows 原生使用 Agent Usage。移植完成前，Windows 安裝必須明確報告尚不支援；若另提供 `-UsageBackend QuotaDeck`，應明確標示為替代模式，不能假稱是 Agent Usage，也不能同時讓兩個外掛覆寫同一組側欄與 statusLine。

保留上游 Apache-2.0 與 MIT 授權、來源 commit、第三方聲明。四個 agent CLI 從各自官方來源安裝，不直接重發登入資料或混包商用二進位檔。

## 預期的使用方式（尚未可執行）

```powershell
.\Install-HerdrBundle.ps1 -InstallRoot D:\Apps\HerdrBundle -Project D:\git\my-project
.\Install-HerdrBundle.ps1 -InstallRoot D:\Apps\HerdrBundle -Repair
.\Test-HerdrBundle.ps1 -InstallRoot D:\Apps\HerdrBundle -Json
```

預設安裝 Herdr、Agent Usage、Claude Code、Codex CLI、Antigravity CLI (`agy`)、OpenCode (`opencode`) 與 Windows Terminal。預設使用一般帳號權限，不為使用者全域開啟 bypass permissions。需要時提供明確的 FullAccess 啟動選項，不修改現有對話權限。

帳號登入仍由使用者完成：Claude/ChatGPT/Google OAuth 不能由安裝腳本假冒同意或自動填答。安裝器應列出 `needs-login` 和下一步，CLI 可用與 quota 可讀分開驗證。

OpenCode 預設提供 OpenRouter 連線指引：在 OpenCode 內執行 `/connect`，選 OpenRouter 並輸入 key，再用 `/models` 選模型。key 留在 OpenCode 的憑證儲存區，不寫進 repo、bundle.lock 或安裝記錄；安裝器不可預設產生計費請求。OpenRouter 只是可選 provider，未設定時回報 `needs-login`/`needs-provider`，不阻止其他 agent 安裝。OpenCode 模型不固定成某個付費模型。

OpenCode 1/2 共用 `opencode` 命令，不能無條件覆寫。release lock 必須選定且驗證一個版本、官方來源與 SHA-256，先查既有版本與 auth/config 路徑，再選擇沿用或顯示明確修復計畫。官方 V1 npm 套件為 `opencode-ai`；不能把 V1 指令與 V2 設定格式混用。參考 https://opencode.ai/docs/ 與 https://opencode.ai/docs/providers/#openrouter 。

## 目錄與交付檔案

```text
Install-HerdrBundle.ps1        # 可重複執行；支援 Repair/WhatIf/無互動模式
Test-HerdrBundle.ps1           # doctor；文字與 JSON 報告
Update-HerdrBundle.ps1         # 明確更新至鎖定 bundle release
Uninstall-HerdrBundle.ps1      # 只移除本套件擁有的檔案與設定
bundle.lock.json              # schema/version、來源、commit、URL、SHA-256
bundled/agent-usage/           # 固定版本外掛原始碼
releases/<bundle-version>/    # 預編譯 Herdr + ConPTY runtime + usagebar.exe
bin/                         # 穩定入口 herdr/claude/codex/agy/opencode
config/                      # Herdr 專用設定，與個人 agent 設定分離
state/                       # installer 狀態、檢查結果與 update journal
backups/<timestamp>/         # 原設定、原入口、ACL、來源版本
logs/                        # 安裝與 doctor 記錄，不記錄憑證
```

一般使用者不需要 Rust、Go、Zig、C++ Build Tools。這些只在 release build/開發流程安裝。Herdr Windows package 必須包含匹配的 app-local ConPTY 檔案，不能只複製 herdr.exe。

## 安裝器執行順序

1. **Discover**：確認 OS、x64/ARM64、PowerShell、磁碟空間、winget、PATH、既有版本、執行中的 server；不用 `Get-Command` 單次失敗就斷言未安裝。
2. **Plan**：顯示將新增/沿用/更新的項目；既有可用 CLI 預設沿用，版本不足才修復。先解析 bundle.lock，無 Windows 相容產物就停止，不能退回不相容 Bash 外掛。
3. **Backup**：保存 config.toml、Claude settings/statusLine、Codex config/hooks、Agy settings/hooks、OpenCode config/plugin 註冊、使用者 PATH 與既有 launcher；保留檔案權限。憑證不複製到公開報告，既有 OpenCode auth 不移除或覆寫。
4. **Terminal**：缺少時以 winget 安裝 `Microsoft.WindowsTerminal`；驗證套件、wt.exe、實際啟動與捷徑。只設定本套件以 Terminal 啟動，不擅改 Windows 全系統預設終端機。
5. **Download**：以 lock 中的官方 URL/固定 commit 下載到 staging，檢查 hash，再啟用；下載失敗不可留下半套 current。
6. **Agent CLI**：安裝/確認 Claude、Codex、Agy、OpenCode；一律測試實際命令解析位置與版本。Codex 預設模型不寫死成某代模型，從登入帳號可用 catalog 選取或沿用。OpenCode 缺少 provider 時提供 `/connect` 指引，保留現有 provider 與預設模型。
7. **Herdr**：啟用帶圖片修正的 bundle build，設定 HERDR_HOME/HERDR_CONFIG_PATH。不得把這兩個變數誤稱為所有 runtime 資料都已搬到 D 槽。
8. **Integrations**：執行 `herdr integration install claude`、`codex`、`antigravity-cli`、`opencode`，核對 status；用 patch/adapter 保留原 hook，而非覆寫整個設定檔。OpenCode adapter 必須匹配其 major version，必要的首次 migration 完成前不強行註冊。
9. **Usage**：註冊鎖定的 Windows Agent Usage，設定四個 harness 的收集器、sidebar rows、更新機制與所需 statusLine adapter；檢查 setup/action 終止狀態，而非只檢查回應 receipt。區分 harness 與計費 provider：OpenCode 使用 OpenRouter 時顯示實際後端的 token/cost/餘額證據，不捏造 5h/7d 訂閱百分比。
10. **Launchers**：建立桌面/開始選單 Herdr Bundle 捷徑，透過 wt 的絕對路徑啟動；用 argv 處理空白/中文路徑。wrapper 只管理 bundle 自己的 bin，不修改 Volta 管理的 shim。
11. **Verify**：CMD、64-bit/32-bit PowerShell、Windows Terminal 均驗證命令解析與版本；檢查 named test session 與現有 session 的相容性。
12. **Finish**：產生安裝報告、登入待辦、回退入口；依 Project 參數打開 Terminal + Herdr。不得自行停止現有 server 或終止 agent 以完成更新。

## 本次踩坑與必須做的處理

### PATH 與多份 CLI

- agent 執行環境曾包含字面 `$PATH` 且缺少 Windows 變數；先讀 Machine/User PATH，必要時只修復目前程序。
- 登錄檔更新不等於既有 Explorer/Terminal 已刷新。安裝程序自己組合可用環境，啟動捷徑使用絕對路徑，並廣播 WM_SETTINGCHANGE；報告既有 shell 需要重開。
- Codex npm 更新到 0.164.0，但 PATH 優先找到 Volta 固定的 0.144.6。必須使用 `Get-Command -All`/`where.exe` 與 `--version` 驗證實際執行版本。
- 本機曾以改寫 Volta shim 緊急解決，正式安裝器不可沿用；應建立 bundle 專屬入口並提供 PATH 衝突診斷。
- 避免 PATH 重複、截斷、刪除其他軟體項目；解除安裝只撤銷本套件新增內容。

### CMD、PowerShell、Terminal

- Windows Terminal 不一定已安裝；執行 wt 失敗或搜尋只出現 terminal.ts 並不奇怪。先檢查，再安裝並建立可實際執行的捷徑。
- PowerShell 7 可能不能載入 Appx 模組；使用 Windows PowerShell 5.1 作 read-only fallback。
- CMD 跨磁碟切換用 `cd /d`；PowerShell 可直接 `cd D:\...`。
- CMD 不使用反斜線跳脫雙引號。必須輸出 `set "NAME=value"` 的實際內容，不能輸出反斜線加引號。
- `Start-Process` 的隱藏模式只用於背景 helper；使用者要操作的 Terminal 視窗正常顯示。

### 本機圖片修正

- 官方原版 local Windows 圖片 bridge 未接線，改 CMD/Terminal 不能保證解決。
- 本分支在 local active surface 攔截 Alt+V，讀 Windows PNG/DIB，重用 ClipboardImage 訊息，server 暫存檔案並貼路徑；不自動按 Enter。
- 沒圖片時按鍵繼續傳給 agent；Ctrl+V、文字、remote binding 不變。正式版可再做可設定的 local_image_paste，避免快捷鍵衝突。
- 測試涵蓋 Alt+V/無 active surface/按鍵 release/文字與 Ctrl+V/mixed batch，並有 opt-in 真實剪貼簿到 PNG 暫存的測試。
- 使用者實體按鍵與 Claude/Codex/Agy/OpenCode 接受圖片，列為 clean-machine 驗收，不以單元測試替代；模型本身不支援圖片時須分別回報，不誤判為 clipboard 故障。

### Usage 與相容性

- Agent Usage 原版可能回報 Installed，但在 Windows 跳過 build；setup 仍回傳 platform_unsupported。下載成功不算可用。
- 目前實際 Windows 使用的是 QuotaDeck，而不是 usagebar。不可在文件、doctor 或 UI 混淆。
- Agy 不能只裝 CLI：要安裝 statusLine collector，收到真實 payload，再刷新 provider snapshot；未回報前不得宣稱用量成功。
- 讀取帳號/額度需要保護憑證；診斷輸出不得含 token、完整 credential payload 或 API key。

### 編譯與發佈

- 已有 Visual Studio 不等於有 MSVC linker/SDK。缺元件時會出現 link.exe not found 或 dbghelp.lib missing。
- VS quiet installer 未提升權限會回傳 5007；如開發模式需要安裝，說明 UAC 並等待程序完整結束再編譯。
- Herdr debug build 預設使用 herdr-dev namespace，不能當成已連到既有正式 server；交付用 release build 並核對 endpoint。
- fork 的套件更新不得被 upstream herdr update 悄悄覆蓋。bundle 更新器控制來源與版本；啟動入口與上一版可回退。

## Doctor 與可重跑契約

每個檢查輸出 `ok / needs-login / unsupported / failed / skipped`，失敗附具體處理步驟。CLI installed、integration installed、authenticated、quota available 分開呈現。無互動模式不假裝完成 OAuth，使用結構化 exit status/JSON 回報。

安裝中斷後重跑應可續接，重跑不新增重複 hook/row/捷徑。更新失敗時 current 仍指向上一個完整版本；回退不刪 agent 歷史、登入、專案或執行中的工作。

## 分期與驗收

1. **先完成 Agent Usage Windows port**：見 bundled/agent-usage/WINDOWS_PORT.md。Go build tags、file locks、process detach、resize、Windows hooks、cmd/shim、tests、native manifest 都要處理，不能只改 platforms。
2. **建立 release pipeline**：Rust + Zig + ConPTY + Go 的 Windows 產物，固定來源/版本/hash、保留授權；在乾淨 VM 驗證。
3. **實作 Installer/Doctor/Repair/Rollback**：預設四個 agent（Claude/Codex/Agy/OpenCode）+ Terminal，沿用既有登入，無開發工具需求。
4. **Clean-machine 驗收**：Windows 10/11、全新/既有 Volta/npm/中文與空白安裝路徑、未裝 Terminal、離線/下載中斷、重跑、更新、回退。
5. **功能驗收**：四個 agent 可用、四個 harness 的 usage 與實際 provider 來源一致、圖片實際貼入支援圖片的四種對話、OpenCode/OpenRouter 連線與選模可用、既有工作不被終止；未提供 key 時正確顯示待設定。通過後才發佈一鍵安裝命令。

下一位 agent 從第 1 階段開始，不要將這份規劃或 submodule 當成 Windows 功能已實作完成。
