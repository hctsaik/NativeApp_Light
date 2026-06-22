# nativeApp_Light — U-Net 開發協定(Tauri 輕量化殼移植)

> 本專案用 **U-Net Agent Workflow**(Spec-Driven + TDD + Phase Gating)開發。
> 目標:把 `C:\code\claude\nativeApp`(CIM Hybrid Edge Platform)的 **Electron host 殼**
> 換成 **Tauri**(輕量),**而不動 Python engine、portal、外部工具**。

## 技術假設(已從預設 Python+pytest 調整)

| 面向 | 本專案採用 |
|---|---|
| 殼框架 | **Tauri 2.x**(Rust + WebView2) |
| 殼後端語言 | **Rust**(取代 Electron main.js 的 Node) |
| 前端 | **沿用既有 React portal**(不重寫;只加 `window.cimHost` Tauri shim) |
| JS 單元測試 | vitest(shim 邏輯) |
| Rust 單元測試 | `cargo test`(port 探測、env 組裝、健康輪詢狀態機) |
| **真實行為驗收(done 的硬標準)** | `tauri dev` 起殼 → 載入 portal → 自動拉起 engine → golden path E2E 綠 |

## 五角色與資料夾隔離(鐵則)

| 指令 | 只能寫 | 產出 |
|---|---|---|
| `/user` | `1_user_needs/` | 需求(要什麼、為什麼痛),不碰技術 |
| `/po` | `2_PO_PRD/` + `ROADMAP.md` | PRD、範圍、module 拆解、appetite、tier |
| `/architect` | `3_Architect_Design/` | I/O 契約、責任對映、AC(帶具體期望值)、風險清單 |
| `/pm` | `4_PM_Feedback/` | 驗收測試(test-first,先紅);擁有測試基礎設施 |
| `/pg` | `5_PG_Develop/` | 實作(Rust + JS shim),自動修到單元全綠 |

同名對齊(skip connection):`3_.../02_sidecar-manager.md` ↔ `4_.../test_sidecar_manager.*` ↔ `5_.../sidecar_manager.rs`。

## 兩條核心紀律

1. **契約不可被下游偷改**:PG 發現設計/測試有錯 → 停手、回報、退回 `/architect` 或 `/pm`,
   不准就地改 `3_*` / `4_*` 或硬編碼騙綠。每次反向閘門在 `ROADMAP.md` 留一行。
2. **真實行為才是 done**:本專案是 **Tier B(GUI/整合/外部程序)**。
   單元綠 ≠ done。涉及「視窗能開、portal 能載、engine 能起、工具能跑」的 module,
   **done = 單元綠 AND `tauri dev` 真實 E2E 綠**。禁止用「檔案存在 / 程序有 spawn / 回傳非空」當綠燈。

## 移植面(這是全部要重寫的 Electron 專屬碼)

來源 `nativeApp/apps/host-electron/`,僅 ~526 行:
- `src/main.js`(493 行):視窗、sidecar 生命週期、IPC handler、dev log server、logging、env 注入、檔案對話框。
- `src/preload.js`(33 行):`contextBridge` 暴露 `window.cimHost`(18 方法 + 4 事件)。

**Python engine / portal / 外部 GUI 完全不動**——它們經 HTTP + 子程序驅動,與殼框架無關。

## 誠實邊界

- U-Net 的「獨立驗證」對 shim 純邏輯只防實作 bug,不防同源語義盲區 → 押在**真實 WebView2 E2E**。
- WebView2(Edge)≠ Electron(Chromium):**embedded Streamlit 工具的算繪/websocket 行為平價性,只能靠真實 E2E 確認**,這是本專案最大 false-green 風險區。
