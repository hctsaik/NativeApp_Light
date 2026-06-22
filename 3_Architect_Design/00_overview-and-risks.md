# 架構總覽 + 風險清單(/architect)

> 本文回答 `/user` 的核心問題:**「直接換 Tauri 會不會有問題?」**
> 結論先講:**高度可行,Tauri 對這個架構甚至比 Electron 更合身**;真正的工作是重寫
> ~526 行的殼,真正的風險集中在「WebView2 能否原樣跑 embedded Streamlit」——只能靠真實 E2E 攔。

## 為什麼這個 app 特別適合 Tauri
Electron 在本架構裡**不是 UI 引擎,而是「視窗殼 + 程序管理器」**:
```
Electron host(殼,~526 行)
  ├─ 開窗、載入 React portal(純 web,URL 載入)
  ├─ spawn Python engine(FastAPI,HTTP on 127.0.0.1:control_port)── 真正的後端
  │     └─ spawn Streamlit 子程序 / 外部 GUI(xanylabeling…)
  └─ 把 portal 的呼叫轉發成 HTTP 打 engine
```
重活(engine、Streamlit、外部工具)全是**框架無關的子程序 + HTTP**。Tauri 同樣會 spawn 程序、同樣載 web、且殼本身小得多——這正是 Tauri 的主場。

## 責任對映:Electron → Tauri(逐項)
| Electron 機制(main.js/preload.js) | Tauri 對應 | 難度 | 備註 |
|---|---|---|---|
| `BrowserWindow` 視窗設定 | `tauri.conf.json` window + `WebviewWindow` | 低 | 直接對映 |
| `loadURL(portalUrl)` dev/packaged 分流 | `devUrl` / `frontendDist` + `app.path().resource_dir()` | 低 | dev 指既有 vite server |
| `spawn(engine, ...)` 子程序 | `tauri-plugin-shell` Command 或 Rust `tokio::process` | 中 | 核心;engine.exe 可走 `externalBin` sidecar |
| `findFreePort()`(net listen 0) | Rust `TcpListener::bind("127.0.0.1:0")` | 低 | 直接對映 |
| `requestJson()` HTTP 轉發 | Rust `reqwest` in `#[tauri::command]` | 低 | 或讓 portal 直接 fetch(見風險 R4) |
| `waitForSidecarReady` 健康輪詢 | Rust async 輪詢 `/health` | 中 | 狀態機需忠實移植 |
| 崩潰自動重啟 + `sidecar-*` 事件 | Rust 監看 exit + `app.emit(...)` | 中 | 非同步正確性 |
| `before-quit` graceful `/shutdown`→kill | Tauri `on_window_event(CloseRequested)` + RunEvent::Exit | 中 | 退出 hook 機制不同(見 R5) |
| `dialog.showOpenDialog`(chooseFile) | `tauri-plugin-dialog` | 低 | 原生對話框 |
| `contextBridge.exposeInMainWorld("cimHost")` | `withGlobalTauri` + 注入 JS shim,內部走 `invoke`/`listen` | 中 | portal 零改動的關鍵(見 R1) |
| env 注入(`labelMeDinoEnv`/`CIM_PYTHON`/`PYTHONUTF8`) | `Command.envs(...)` | 低 | 直接對映 |
| `appendLog`/`purgeStaleLogs` | Rust 檔案寫入 或 `tauri-plugin-log` | 低 | |
| dev log server(:19222) | 小型 Rust HTTP 或本輪略過 | 低 | dev-only,可 Could |
| `extraResources`(engine.exe/python/portal/labelme) | `bundle.resources` + `externalBin` + `resource_dir()` | **中高** | 打包機制不同(見 R6),本輪僅設計 |

## 風險清單(severity:🔴高 / 🟡中 / 🟢低)

### 🔴 R1 — `window.cimHost` 介面平價性
- **是什麼**:portal(`main.jsx`)整支依賴 `window.cimHost` 的 18 方法 + 4 事件。任一方法語義/回傳形狀對不上,portal 就壞。
- **為什麼**:這是 portal 與殼的唯一契約面;Electron 用 preload `contextBridge`,Tauri 沒有 preload,要改用「啟動時注入一段 shim JS」把同名方法接到 `invoke`/`listen`。
- **緩解**:把 `cimHost` 當**硬契約**逐一同名同簽章移植(module 03);`/pm` 驗收用「`window.cimHost.listTools()` 回傳 == 直打 engine `/tools`」這種跨來源交叉斷言,不只查「函式存在」。
- **如何驗**:真實 E2E 在 WebView2 裡呼叫每個方法,比對 engine 實際回應。

### 🟡 R2 — WebView2(Edge)vs Chromium 算繪/行為差異(**真實 E2E 已部分證偽,降級**)
- **是什麼**:portal 內嵌 Streamlit 工具(iframe/webview)。Electron 夾帶 Chromium;Tauri 用系統 WebView2(Edge 核心)。
- **真實 E2E 實測結果(2026-06-22)**:
  - ✅ **React portal 在 WebView2 完整算繪**:標題列/品牌/DEV badge/角色下拉/工具下拉/Input-Output 分頁全部正常,中文正常,CSS 正常。
  - ✅ shim 全通:`getAppConfig`/`listTools`(回 23 工具,與 `/tools` 一致)/`whoami`/`runtime`/`startTool`(回 `category=sheet sheet_tabs=4`)。
  - ⚠️ **啟動標註工具後內容區顯示「Not Found」**——但**經 curl 直打(完全不經 WebView)同樣 404**,故**非 WebView2 算繪問題**。
- **「Not Found」根因(已查):frozen `engine.exe` 的打包問題,與 Tauri/shell/env 無關**。
  - 證據:① curl 直打工具埠 `/` 得 `404 uvicorn`,但 `/_stcore/health`=200(Streamlit 核心在跑、只是 `/` 沒 index);② 注入 Electron 全套 env(`CIM_REPO_ROOT` 等)後 `/` 仍 404 → 排除 env 注入缺漏;③ 典型 PyInstaller 凍結 Streamlit 沒包進前端 static 資產的徵狀。
  - **同一個 `engine.exe` 在 Electron 下也會一樣 404**;平常 dev 用 `engine.py`(原始碼含 static)才正常。**確認/解法**:Tauri 殼指向 `engine.py`(需 Python env)即可驗證工具真正算繪。
- **結論(經 multi-agent Playwright E2E 最終驗證)**:改用 `engine.py` 後,**5 個 menu 功能全部在真實 WebView2 中 RENDERED**(Playwright 經 CDP 驅動、各跑獨立隔離實例):sheet-edge-analysis / management-center / sheet-annotation / app-ai4bi / app-lv,health 與 root 皆 200,embedded Streamlit 真實算繪。**R2 完全證偽 —— WebView2 對這個 app(含 embedded Streamlit)無算繪問題**。先前的「Not Found」純為 frozen engine.exe 打包,非 Tauri。

### 🟡 R3 — sidecar 生命週期狀態機的非同步正確性
- **是什麼**:健康輪詢逾時、崩潰重啟(3s)、graceful shutdown(5s 逾時再 kill)、重啟期事件競態。
- **緩解**:`/pm` 對 `sidecar-manager` 寫 metamorphic/不變量測試(kill→自動重啟後 `/health` 再次 ok;雙重 stop 冪等;就緒前 exit → reject)。
- **如何驗**:`cargo test` 對狀態機 + E2E 殺掉 engine 觀察自動恢復。

### 🟡 R4 — portal 直連 engine 的 CORS(真實 E2E 抓到,已修)
- **是什麼**:多數呼叫走 cimHost shim(Rust 轉發,無 CORS 問題)。但 portal 有 **4 處直接 `fetch()` engine**(繞過 shim):`/whoami`、`/set-role`、`/tools/runs/log`、`/tools/preview/stop`(`main.jsx` 581/962/789/887)。
- **E2E 實測**:Tauri 的 portal origin 是 `tauri.localhost`,直連 `http://127.0.0.1:<engine>` 跨來源,engine **原本完全沒設 CORS** → 被 WebView2 擋(console:`blocked by CORS policy`)。全包在 `.catch()` 故 golden path 不受影響(靜默失敗:RBAC 角色/執行歷史 DEV 功能),但噴 console error。
- **修正(已套用)**:`nativeApp/sidecar/python-engine/engine.py` 的 `create_app()` 加 `CORSMiddleware(allow_origins=["*"])`。引擎僅綁 127.0.0.1(非網路暴露),放行 CORS 安全且標準,Electron 也一併受益。驗證:sheet-annotation console error 2→0,仍 RENDERED。
- **注意**:此修改在 `nativeApp`(共用 engine 原始碼,非本 spike repo);只影響 `engine.py`,frozen `engine.exe` 需重建才含。

### 🟡 R5 — 退出時 graceful sidecar 關閉
- **是什麼**:Electron `before-quit` 可 `preventDefault` 等 async shutdown;Tauri 的 `CloseRequested`/`RunEvent::Exit` 模型不同,要正確攔截關窗、先 `/shutdown` 再放行,否則殘留孤兒 Python 程序。
- **緩解**:用 `on_window_event` 攔 `CloseRequested` + `api.prevent_close()`,完成 shutdown 再 `window.close()`。
- **如何驗**:E2E 關窗後確認無殘留 engine 程序(工作管理員/`tasklist`)。

### 🟡 R6 — 打包機制(electron-builder → Tauri bundler)
- **是什麼**:`extraResources`(engine.exe 430MB、python-runtime、portal dist、labelme-dino、sidecar-source)→ Tauri `bundle.resources`/`externalBin`,執行期路徑從 `process.resourcesPath` 改 `resource_dir()`。
- **決策**:本輪 **Won't 出可攜檔**,只做設計與 dev 跑通;打包是後續輪次(風險已知、非阻斷)。

### 🔴🔴 R9 — WDAC 企業政策 + Smart App Control 封鎖未簽章原生碼(本機實測兌現,最決定性)
- **是什麼**:本機 **WDAC 使用者模式碼完整性 = 強制(enforce)** 且 **Smart App Control = 開啟**。實測 `cargo build` 在執行 `tauri-plugin-fs` 的 build-script 時被 WDAC 政策 `{0283ac0f-…}` 封鎖(CodeIntegrity 事件 3077):「未達企業簽章層級要求」。
- **為什麼這對 Tauri 致命、對 Electron 無感**:
  - **Electron**:你執行的是 **Electron 專案已簽章** 的 `electron.exe`(WDAC/SAC 信任),你的程式是被它載入的 JS——**全程沒有未簽章原生 exe 被執行**。所以 `npm run dev` 在本機正常。
  - **Tauri**:① build 過程 cargo 必須「編譯並執行」一堆**未簽章的 build-script .exe** → 被擋,連 build 都不能完成;② 即使 build 成功,產出的 `cim-light.exe` 也是**你自己的未簽章原生檔**,執行時同樣被 SAC/WDAC 擋。
- **代價(這就是「換 Tauri 的隱藏成本」)**:在 SAC/WDAC 鎖定的環境,Tauri 需要**用政策接受的憑證做 code signing**(WDAC「企業簽章層級」可能要求把你的簽章憑證加入政策允許清單)——這是組織級的簽章/IT 流程成本,Electron 靠廠商已簽章 runtime 完全避開。
- **本機要跑起來的唯一路徑**:關閉 Smart App Control(**不可逆**,需重灌才能再開)+ 視情況移除/放寬 WDAC 企業政策(需 admin、重開機,若 MDM 管理會自動還原)。屬使用者/IT 的安全決策,本流程不自動執行。
- **狀態**:**真實 E2E 閘門在本機被此政策阻擋,Tier B 模組無法達到「done=E2E 綠」**。誠實標註:非 false-green,而是 blocked-by-environment。

### 🟢 R7 — 需安裝 Rust 工具鏈(dev 環境變更)
- **是什麼**:Tauri build 需 Rust+cargo,本機未裝;首次 build 會編譯大量 crate(數分鐘)。WebView2 runtime 已裝。
- **緩解**:`rustup` 安裝,可逆(`rustup self uninstall`)。**這是 /pm /pg 的前置閘門,需使用者同意**。

### 🟢 R8 — dev 工作流變更
- **是什麼**:現用 `concurrently + wait-on + vite`;Tauri 用 `beforeDevCommand`/`devUrl` 取代,反而更乾淨。
- **緩解**:設定 `beforeDevCommand` 指向 portal 的 vite dev。

## 驗收切入點(交給 /pm)
1. **02 先行**(無上游):engine 起得來、`/health` ok、kill 後自動恢復、雙重 stop 冪等。
2. **03**:`window.cimHost` 每方法回傳 == 直打 engine;跨來源交叉斷言。
3. **01**:窗開、尺寸對、載入 portal、收到 02 就緒事件。
4. **真實 E2E 閘門(全綠才可標 done)**:`tauri dev` → 窗開 → portal 載 → engine 起 → 工具清單出現 → **操作一個工具看到輸出** → 關窗無殘留程序。

## 一句話結論(經完整真實 E2E 後最終版)
換 Tauri **技術上完全可行,且已實證跑通**:`cim-light.exe` 開窗 → Rust 重寫的 sidecar manager 正確拉起既有 `engine.exe`(21s ready)→ WebView2 算繪 React portal → `window.cimHost` shim 全通(listTools 回 23 工具與 `/tools` 一致)→ 關窗 graceful 無殘留(AC1.4)。單元測試 cargo 4/4、vitest 3/3。
兩個與「Tauri 技術可行性無關」的外部因素:
1. **R9 環境級代價(最重要)**:鎖定機器的 WDAC/SAC 會擋未簽章原生碼 → **Tauri 正式遷移的真正成本是 code signing + 政策放行**;Electron 靠廠商已簽章 runtime 繞過。
2. **工具內容算繪未展示**:根因為 frozen `engine.exe` 打包(Streamlit static 未包全),**非 Tauri**;指向 `engine.py` 即可驗。
→ 殼/橋接/引擎/生命週期全部驗證通過。Tauri 對這個「薄殼 + 程序管理器」架構是合身且可行的替代。
