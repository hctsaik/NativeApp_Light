# PRD:Electron → Tauri 殼移植可行性 Spike(/po)

## 三行自問(放行前必答)
- **誰會用**:平台維護者(你本人)——要評估「正式遷移到 Tauri」值不值得、會不會踩雷。
- **不做會怎樣**:繼續扛 Electron 的打包體積與記憶體;或在沒有實證下盲目遷移,風險不可知。
- **成功的可觀察判準**:`tauri dev` 起殼 → 載入既有 portal → 自動拉起既有 Python engine → 工具清單出現 → 至少一個工具(golden path)能跑,且行為與 Electron 版一致。

## 定位:這是 **feasibility spike(可行性探針)**,不是正式遷移
U-Net 對「純探索期」的指引是 **先 thin-slice 打通端到端拿真實回饋**。
本輪只證明/否證「Tauri 能承接這個架構」,並產出**風險清單**;正式遷移(打包/簽章/CI/退役 Electron)是後續輪次。

## 範圍(MoSCoW)
| 優先 | 項目 |
|---|---|
| **Must** | (1) Tauri 殼開窗、載入既有 portal dev URL;(2) 用 Rust 拉起 `engine.py`(dev)並輪詢 `/health` 就緒;(3) `window.cimHost` shim,讓 portal **零改動**跑起來;(4) golden path 真實 E2E:工具清單出現 + 啟動一個工具 |
| **Should** | sidecar 崩潰自動重啟、關窗時 graceful `/shutdown`、檔案對話框(`chooseFile`) |
| **Could** | 對映 `engine.exe`(packaged 路徑)、dev log server、外部工具佇列 IPC |
| **Won't(本輪)** | electron-builder → Tauri bundler 完整打包、簽章、自動更新、Fleet 分發、退役 Electron |

## 模組拆解(粒度轉換:feature → module)
詳見 [`module-breakdown.md`](module-breakdown.md)。四個 module,全為 **Tier B**(有 GUI/整合/外部程序,正是 false-green 重災區,不可降級):

| # | module | 一句話職責(無「以及」) | 對應 Electron 來源 |
|---|---|---|---|
| 01 | `app-shell` | 開出視窗並載入 portal URL(dev/packaged 分流) | `createWindow` / `portalUrl` |
| 02 | `sidecar-manager` | 管理 Python engine 子程序的完整生命週期(找埠→spawn→健康輪詢→崩潰重啟→graceful 關閉) | `startSidecar`…`stopSidecar` |
| 03 | `cimhost-bridge` | 在 WebView 注入 `window.cimHost`,把 18 個方法+4 事件對映到 Tauri command/event | `preload.js` + 各 `ipcMain.handle` |
| 04 | `packaging-spike` | (Should/Could)dev 可跑後,驗證 resource 路徑對映可行性 | `extraResources` / `resourcesPath` |

### 拆解自檢
- **內聚**:每個 module 一句話講得完(見上表),無「以及」。✅
- **契約耦合**:`app-shell` 只需 `sidecar-manager` 給的「就緒事件」;`cimhost-bridge` 只需 `sidecar-manager` 給的 `control_port` + 一個 HTTP 轉發函式。介面顯式、不成環。✅
- **可獨立驗收**:01 可單獨驗(窗開、URL 對);02 可單獨驗(engine 起、`/health` 200、kill 後重啟);03 可單獨驗(`window.cimHost.listTools()` 回傳與直打 engine 一致)。✅

## Appetite(本輪上限)
- **≤4 module、僅 dev 模式跑通**。達標即停輪,packaged 打包與正式遷移進「候選/未排程」。
- 例外:若 E2E 發現 portal 在 WebView2 下**不可用**(false-green 風險兌現),修到真能用不受 appetite 約束(這是本法強項)。
