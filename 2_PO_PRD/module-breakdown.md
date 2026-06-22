# 模組拆解明細(/po)

> 對外相依顯式列出且不成環:`03 → 02`(要 control_port + 轉發)、`01 → 02`(要就緒事件)、`04 → 01,02,03`。

## 01 `app-shell`
- **職責**:建立 Tauri 視窗(1280×820,min 960×640,啟動 maximize),載入 portal URL。
- **dev/packaged 分流**:dev → `http://127.0.0.1:5173`(既有 vite dev server);packaged → 打包進 bundle 的 portal `index.html`(本輪僅設計,不實作 packaged)。
- **相依**:訂閱 `02` 的「sidecar-ready / exited」事件以轉發給 portal。

## 02 `sidecar-manager`(最重、最像 Electron main 的核心)
- **職責**:Python engine 子程序的完整生命週期。
- **狀態機**:`找空閒埠` → `spawn(engine.py --control-port P --log-dir L)`(注入 env:`PYTHONUTF8=1`、`LABELME_*`、`CIM_PYTHON`)→ `輪詢 GET /health 直到 status==ok 或逾時` → `就緒`。
- **韌性**:stdout/stderr 收進 log;非預期 exit → 3 秒後自動重啟並發事件;關閉 → 先 `POST /shutdown`(5s 逾時)再 kill。
- **相依**:無上游;對下游(01/03)提供 `control_port` 與生命週期事件。

## 03 `cimhost-bridge`(portal 零改動的關鍵)
- **職責**:在 WebView 注入 `window.cimHost`,介面與 Electron preload **逐一同名同簽章**。
- **18 方法**:多數是把呼叫**轉發成 HTTP** 打 `127.0.0.1:control_port`(`listTools`/`startTool`/`stopTool`/`getToolStatus`/`getRuntimeStatus`/`getDiagnostics`/`startSheetTab`/`external*`…)。少數是**真原生**:`getAppConfig`(回 control url + logDir + devMode)、`chooseFile`(原生對話框→POST `/selected-paths`)、`log`(寫 host log)、`restartSidecar`。
- **4 事件**:`onSidecarExited/Restarting/Ready/RestartFailed` → 對映 Tauri `listen`。
- **相依**:`02` 的 control_port + HTTP 轉發。

## 04 `packaging-spike`(Should/Could,本輪可能只到設計)
- **職責**:驗證 Electron `extraResources`(engine.exe / python-runtime / portal dist / labelme-dino / sidecar-source)→ Tauri `bundle.resources` + `externalBin` + `resource_dir()` 的對映可行,不實際出可攜檔。

## Tier 判定
全部 **Tier B**:每個 module 都涉及 GUI 或外部程序或跨程序整合。**不允許降級為 Tier A**——這正是整合性 false-green 的核心價值區。done 標準一律「單元綠 AND 真實 E2E 綠」。
