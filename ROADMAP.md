# ROADMAP — nativeApp_Light(單一真相來源)

> 人寫區=里程碑/凍結決策/狀態判斷;機器生成區=模組與測試數(待 unet-status 稽核器)。

## 里程碑敘事
- **M0 可行性 spike(本輪,appetite ≤4 module / 僅 dev)**:Tauri 殼跑通既有 portal + engine,golden path E2E 綠,並產出風險清單。**正式遷移不在本輪。**

## 凍結決策(決策日誌:日期 / 決策 / 一句為什麼)
- 2026-06-22 / 定位為 feasibility spike 而非正式遷移 / 需求屬探索期,先 thin-slice 拿真實回饋(U-Net 對探索期的指引)。
- 2026-06-22 / portal 與 Python engine **零改動**,只重寫 ~526 行 Electron 殼 / 重活在框架無關的子程序+HTTP 層,移植面其實很小。
- 2026-06-22 / 四 module 全列 Tier B、done=單元綠 AND 真實 E2E 綠 / WebView2≠Chromium,embedded Streamlit 平價性是最大 false-green 風險,只能靠真實 E2E 攔。
- 2026-06-22 / **閘門:/pm /pg 卡在「使用者是否同意裝 Rust 工具鏈」** / Tauri build 硬前提是 Rust+cargo,本機未裝;裝與否是使用者的決定。
- 2026-06-22 / 機器為 **ARM64 Windows 11 Home**;採 x86_64 Rust 以 x64 模擬建置(Hostx64\x64\link.exe + lib\x64 齊備,零額外安裝);原生 aarch64 留作 production。
- 2026-06-22 / engine.exe 獨立驗證通過(/health ~24s、/tools=23、/shutdown);就緒 timeout 30→60s。
- 2026-06-22 / **【決定性阻擋 R9】真實 E2E 在本機被 WDAC 企業政策 `{0283ac0f}` enforce + Smart App Control On 擋住**:cargo build 執行未簽章 build-script 即被封鎖(CI 事件 3077)。Tauri 連 build 都不行、產出 exe 也無法執行;Electron 靠已簽章 runtime 繞過。本機要跑需關 SAC(不可逆)/放寬 WDAC(admin、可能 MDM 還原)——屬使用者安全決策,流程不自動執行。Tier B done=E2E 綠 因此 **blocked-by-environment**(非 false-green)。
- 2026-06-22 / 使用者關閉 SAC(個人電腦)→ 解封確認(未簽章 build-script 可執行)→ **cargo build 綠(1m44s)、cargo test 4/4、vitest 3/3**。
- 2026-06-22 / **真實 E2E 跑通(殼層)**:`cim-light.exe` 啟動 → Rust spawn `engine.exe`(21s ready)→ WebView2 算繪 React portal(品牌/DEV/角色/分頁全正常)→ shim `listTools` 回 23 工具(與 `/tools` 一致,AC3.4)、`getAppConfig/whoami/runtime` 全通 → **AC1.4 關窗 graceful 無殘留(10 engine 歸零)**。
- 2026-06-22 / **工具內容「Not Found」根因 = frozen `engine.exe` 打包(Streamlit static 未包全),非 Tauri/shell/env**:curl 直打 `/`=404 但 `/_stcore/health`=200;注入 Electron 全套 env 後仍 404。同 engine.exe 在 Electron 亦同。要展示工具真算繪需指向 `engine.py`(Python env)。
- 2026-06-23 / **改指向 `engine.py` 後工具真算繪確認**:加 Rust 支援(sidecar 偵測 `.py` 副檔名 → `CIM_ENGINE_PYTHON` 跑);py-3.11 已含全部 engine 依賴零安裝;sheet-edge-analysis curl `/`=200+HTML。
- 2026-06-23 / **後續三軌**:① code signing 管線示範(signtool 簽 cim-light.exe 成功 + tauri.conf `bundle.windows` 設定 + `SIGNING.md` production 指南,自簽 dev 憑證 thumbprint `9A91…`);② CI `.github/workflows/ci.yml`(vitest + cargo test,含 frontendDist stub);③ console error 根因=portal 直連 engine 被 CORS 擋(R4),已在 `nativeApp/engine.py` 加 CORSMiddleware 修掉(2→0)。
- 2026-06-23 / **Multi-agent Playwright E2E:5/5 menu 功能全 PASS**:Playwright 經 CDP 連 Tauri WebView2,5 工具各跑「獨立 WebView2 user-data + CDP port + log dir + engine」隔離實例,全部 `RENDERED`(sheet-edge-analysis/management-center/sheet-annotation/app-ai4bi/app-lv,health/root 皆 200)。harness 在 `5_PG_Develop/e2e/`(probe.mjs、run-tool.mjs)。teardown 小瑕疵:engine auto-restart 換 port 時 /shutdown 會漏掉新 port(已手動掃孤兒)。

## 模組狀態
| # | module | tier | 狀態 | 相依 |
|---|---|---|---|---|
| 01 | app-shell | B | **完成**(窗開/載 portal/就緒序/AC1.4 關窗無殘留 皆真實驗證) | 02(就緒事件) |
| 02 | sidecar-manager | B | **完成**(cargo test 4/4 + E2E:spawn engine、21s ready、graceful 關閉) | — |
| 03 | cimhost-bridge | B | **完成**(vitest 3/3 + E2E:listTools=23 與 /tools 一致、config/whoami/runtime 全通) | 02 |
| 04 | packaging-spike | B | 候選(未排程,本輪僅設計) | 01,02,03 |

> 註:embedded Streamlit 工具的「內容算繪」未在本輪展示,根因為 frozen `engine.exe` 打包(非 Tauri,見決策日誌)。殼/橋接/生命週期三模組皆已真實 E2E 驗證。

## 待辦優先序(appetite ≤4)
1. 風險清單 + 責任對映(架構,**回答「會不會有問題」**)— 進行中
2. 〔閘門〕使用者確認 spike 範圍 + Rust 安裝
3. /pm:02、03、01 的驗收測試(test-first)
4. /pg:實作到單元綠 → 真實 E2E 閘門
5. 候選(未排程):04 packaging、簽章、CI、退役 Electron

## 機器生成區(unet-status 稽核)
（尚未產生:本輪設計階段,5_PG_Develop 仍空。）
