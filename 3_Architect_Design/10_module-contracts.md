# 逐模組 I/O 契約 + 驗收標準(/architect)

> 對映來源見 [`00_overview-and-risks.md`](00_overview-and-risks.md) 的責任對映表。
> AC 帶**具體期望值**,禁止「存在/非空」。同名對齊:本檔 ACn ↔ 測試 `# ACn` ↔ 實作。

## 02 sidecar-manager(Rust;對映 main.js startSidecar…stopSidecar)

**輸入**:`engine_exe`(路徑,spike 來源 = `nativeApp/sidecar/python-engine/dist/engine.exe`,可由 env `CIM_ENGINE_EXE` 覆寫)、`log_dir`。
**輸出**:`control_port: u16`、生命週期事件、轉發 HTTP 的 `request_json`。

- **AC2.1 找空閒埠**:`find_free_port()` 回傳 `1024..=65535` 的埠,且該埠當下可被 `TcpListener::bind("127.0.0.1:0")` 成功綁定;連續兩次呼叫不保證相異但都須可綁定。
- **AC2.2 spawn 參數**:spawn engine 時 argv 必含 `["--control-port", "<port>", "--log-dir", "<log_dir>"]`,env 必含 `PYTHONUTF8=1`,且 `CREATE_NO_WINDOW`(windowsHide 對等)。
- **AC2.3 健康輪詢就緒**:對一個回傳 `{"status":"ok"}` 的 mock server,`wait_for_ready(timeout=2s)` 在 timeout 內回 `Ok`;對一個恆回 503 的 server,逾時回 `Err("...timed out...")`,且耗時 ≥ timeout、< timeout+0.5s。(R3 不變量)
- **AC2.4 就緒前 exit 視為失敗**:engine 程序在 `/health` ok 前 exit,`start()` 回 `Err`,訊息含 exit 資訊,不得卡死等到逾時。
- **AC2.5 graceful 關閉**:`stop()` 先 `POST /shutdown`;5s 內未 exit 才 `kill`;重複呼叫 `stop()` 冪等(第二次為 no-op,不 panic)。
- **AC2.6 崩潰自動重啟**:非主動停止下 engine exit → 發 `sidecar-restarting`,3s 後重啟;重啟成功發 `sidecar-ready`,失敗發 `sidecar-restart-failed{error}`。

## 03 cimhost-bridge(Rust commands + 注入 shim JS;對映 preload.js)

**契約鐵則(R1)**:`window.cimHost` 的 **18 方法 + 4 事件**與 Electron preload **逐一同名同簽章**。

- **AC3.1 介面完整性**:注入後 `window.cimHost` 存在以下 key(型別 function),一個不缺:
  `getAppConfig, listTools, startTool, startSheetTab, stopTool, chooseFile, restartSidecar, getToolStatus, getRuntimeStatus, getDiagnostics, log, externalOpenXanylabeling, externalOpenLabelingTool, externalQueueImage, externalGetQueue, externalDequeue, onSidecarExited, onSidecarRestarting, onSidecarReady, onSidecarRestartFailed`。
- **AC3.2 方法→command 對映**:呼叫各方法時以正確 command 名與參數呼叫 Tauri `invoke`,例:
  `startTool('m7')` → `invoke('start_tool', {toolId:'m7'})`;`listTools()` → `invoke('list_tools')`;
  `externalDequeue('id1')` → `invoke('external_dequeue', {itemId:'id1'})`;`log('info','x')` → `invoke('renderer_log',{level:'info',message:'x'})`。
- **AC3.3 getAppConfig 形狀**:回傳物件含 `sidecarControlUrl`(= `http://127.0.0.1:<port>`)、`mockJwt`、`logDir`、`devMode`(boolean)、`allowedOrigins`(陣列)。
- **AC3.4 跨來源交叉斷言(真實,E2E)**:WebView 內 `window.cimHost.listTools()` 的回傳,與直接 `GET http://127.0.0.1:<port>/tools` 的回傳**內容相等**(R1/R2 防 false-green:不准只查「函式存在」)。
- **AC3.5 chooseFile**:取消時回 `{canceled:true, paths:[]}`;選檔時回 `{canceled:false, paths:[...]}` 並已 `POST /selected-paths {paths}`。

## 01 app-shell(tauri.conf.json + Rust setup;對映 createWindow/portalUrl)

- **AC1.1 視窗尺寸**:主視窗初始 1280×820、min 960×640、title `CIM Hybrid Edge Platform`,啟動後 maximize。
- **AC1.2 載入 portal**:dev spike 載入既有 portal 產物(`nativeApp/apps/portal-react/dist`),畫面出現 portal 根節點(非空白、非錯誤頁)。
- **AC1.3 啟動序**:engine 就緒後才顯示視窗(對等 Electron `show:false`→ready 後 show);engine 啟動失敗顯示錯誤而非空白卡死。
- **AC1.4 關閉清理(R5)**:關窗攔截 → 完成 sidecar graceful 關閉 → 視窗關閉 → **無殘留 engine 程序**(E2E 用 `tasklist` 確認)。

## 真實行為 E2E 閘門(全綠才可標 done)——對映 PRD 成功判準
`tauri dev` → 窗開(AC1.1)→ portal 載入(AC1.2)→ engine 自動起(AC2.3)→
`window.cimHost.listTools()` == 直打 `/tools`(AC3.4)→ **啟動一個工具看到輸出**(R2)→ 關窗無殘留(AC1.4)。
