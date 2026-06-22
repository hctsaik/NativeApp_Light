# 驗收標準與測試對映(/pm)

> 測試先行(先紅)。PM 擁有測試檔與測試基礎設施,**PG 唯讀**;PG 因基礎設施跑不動測試走 reverse gate 回報。
> 跨工具鏈定位(務實偏離,記於 ROADMAP):Rust 單元測試放 `5_PG_Develop/src-tauri/tests/`(cargo 慣例),JS shim 測試放 `5_PG_Develop/tests/`(vitest);本檔為權威 AC↔測試對映。

## 機器化綠燈三條件(judge 不由 PG 自述)
1. `cargo test` exit==0 且至少蒐集到 1 個測試、0 failed/0 error。
2. `npm run test`(vitest)exit==0 且 0 failed。
3. **真實 E2E 閘門通過**(見下)——Tier B 的 done 硬標準,不在 PG 自動修綠迴圈內,獨立步驟人觸發。

## 單元測試對映(防實作 bug)
| AC | 測試 | 性質 |
|---|---|---|
| AC2.1 find_free_port | `test_find_free_port_is_bindable` | 不變量:回傳埠可再次 bind |
| AC2.2 spawn 參數/env | `test_spawn_args_and_env` | 轉抄+反向稽查(必含 PYTHONUTF8=1) |
| AC2.3 健康輪詢就緒/逾時 | `test_poll_health_ready_ok` / `test_poll_health_timeout` | **metamorphic**:mock ok→Ok;mock 503→Err 且耗時≈timeout |
| AC3.1 介面完整性 | `cimhost-shim · exposes all 18 methods + 4 events` | 反向稽查:逐一列舉 key |
| AC3.2 方法→command 對映 | `cimhost-shim · maps method to invoke(cmd,args)` | 防實作 bug:mock invoke 斷言 cmd 名與參數 |
| AC3.3 getAppConfig 形狀 | (E2E,需真實 port) | — |

## 真實 E2E 閘門(golden path,全綠才可標 done)
人觸發 `npm run tauri:dev`(對 `engine.exe`)後,**人/MCP 逐項確認**:
1. **AC1.1** 視窗開啟,標題 `CIM Hybrid Edge Platform`,可見且 maximize。
2. **AC1.2** portal 畫面載入(看得到 portal UI,非白屏/錯誤頁)。
3. **AC2.3** engine 自動起(host log 出現 `Sidecar ready on port <P>`)。
4. **AC3.4** DevTools console 跑:
   `await window.cimHost.listTools()` 的結果,與 `fetch('http://127.0.0.1:<P>/tools').then(r=>r.json())` **內容相等**。
5. **R2 真實操作**:在 portal 啟動一個工具,**實際看到該工具 Streamlit 介面算繪並可操作**(不只看 iframe 出現)——這是 WebView2 平價性的真憑據。
6. **AC1.4** 關窗後 `tasklist | findstr engine` **無殘留**。

任一項不過 = 紅,記入 ROADMAP 決策日誌並回報。
