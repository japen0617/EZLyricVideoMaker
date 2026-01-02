# Gemini Lyric Video Maker - 程式碼審查報告 (Code Review Report)

**審查日期**: 2025-05-24
**審查範圍**: 整個專案 (Frontend, Electron, Services)
**審查重點**: 安全性、效能、架構設計、程式碼品質

## 1. 總結 (Executive Summary)

此專案是一個結合 Gemini AI 與 FFmpeg.wasm 的創新應用，概念驗證 (PoC) 完成度高，能成功執行核心功能。程式碼結構清晰，使用了現代化的 React + Vite 堆疊。

然而，在**生產環境就緒度 (Production Readiness)** 方面存在幾個關鍵問題：
1. **安全性風險**：缺少 Content Security Policy (CSP)，且過度依賴外部 CDN，這在 Electron 應用中是安全隱患。
2. **效能瓶頸**：影片渲染邏輯在主執行緒進行密集的 Canvas 操作，可能導致介面凍結。
3. **離線功能矛盾**：README 宣稱 "Offline Processing"，但核心腳本 (FFmpeg, Tailwind) 均透過 CDN 載入，無網路環境下無法運作。

---

## 2. 安全性審查 (Security Review)

### 🔴 高風險 (Critical)
*   **缺少 Content Security Policy (CSP)**
    *   **位置**: `index.html`
    *   **問題**: Electron 應用必須定義 CSP 來防止 XSS 攻擊。目前完全未設定，允許執行任何來源的腳本。
    *   **建議**: 在 `<head>` 中加入 `<meta http-equiv="Content-Security-Policy" ...>`，嚴格限制腳本來源。

*   **外部腳本依賴 (Supply Chain Risk)**
    *   **位置**: `services/ffmpegService.ts`, `index.html`
    *   **問題**: 透過 `unpkg.com` 和 `esm.sh` 載入核心程式庫。如果這些 CDN 被入侵或服務中斷，應用程式將無法運作或執行惡意程式碼。
    *   **建議**: 應將 `ffmpeg-core`, `react`, `tailwindcss` 等依賴項納入本地建置 (Bundling)，而非執行時下載。

### 🟡 中風險 (Medium)
*   **API Key 儲存**
    *   **位置**: `services/geminiService.ts`
    *   **問題**: API Key 以明文儲存在 `localStorage`。雖然這是純客戶端應用的常見做法，但在 Electron 環境中，若遭惡意軟體讀取檔案系統或透過 XSS 攻擊，Key 容易外洩。
    *   **建議**: 考慮使用 Electron 的 `safeStorage` API 進行加密儲存。

*   **Electron 權限設定**
    *   **位置**: `electron/main.cjs`
    *   **現狀**: `nodeIntegration: false`, `contextIsolation: true` 設定正確 (Good Job!)。
    *   **建議**: 應明確啟用 `sandbox: true` 以進一步隔離渲染進程。

---

## 3. 架構與效能審查 (Architecture & Performance)

### 🔴 效能瓶頸 (Performance Bottleneck)
*   **影片渲染流程效率低**
    *   **位置**: `services/ffmpegService.ts` -> `createVideo` & `renderFrame`
    *   **問題**: 每一幀 (Frame) 都執行以下流程：
        1. 建立新的 Canvas
        2. 繪製圖片與文字
        3. `canvas.toBlob` (非同步)
        4. `Blob` -> `ArrayBuffer` -> `Uint8Array`
        5. `ffmpeg.writeFile` (寫入虛擬檔案系統)
        *假設 3分鐘影片 @ 4fps = 720幀，這代表大量的記憶體配置與 I/O 模擬，且 Canvas 操作佔用主執行緒。*
    *   **建議**:
        1. 重用同一個 Canvas 實例。
        2. 考慮使用 `OffscreenCanvas` 在 Web Worker 中繪製，避免卡頓 UI。
        3. 僅在字幕變更時重新生成 Frame，重複使用相同的圖片資料 (目前已有簡單的 `lastSubtitle` 緩存機制，這點很好，但實作上仍有優化空間)。

### 🟡 架構矛盾 (Architecture Issue)
*   **離線功能不實**
    *   **位置**: `index.html`, `services/ffmpegService.ts`
    *   **問題**: README 強調 "Offline Processing"，但 `ffmpeg-core.js` 與 `.wasm` 檔是寫死從 `unpkg.com` 下載。Tailwind CSS 也是透過 CDN 載入。
    *   **建議**: 必須下載這些資源並放在 `public/` 資料夾中，改為本地路徑引用，才能真正實現離線可用。

*   **Vite 建置 vs Import Map**
    *   **位置**: `index.html`
    *   **問題**: 使用了 Vite 進行建置，但 `index.html` 中卻保留了 `importmap` 指向 `esm.sh`。這通常是開發階段的殘留物，正式 Build 應該依賴 Vite 打包好的 chunk。這可能導致重複載入 React 或版本衝突。

---

## 4. 程式碼品質 (Code Quality)

### 🟢 優點 (Pros)
*   **專案結構**: 清晰的職責分離 (`services/` vs `components/`)。
*   **類型定義**: 定義了 `VideoData` 等介面，大多數地方都有型別標註。
*   **UI/UX**: `App.tsx` 中的狀態管理邏輯直觀，提供了不錯的載入回饋。

### 🟡 改進空間 (Improvements)
*   **死程式碼 (Dead Code)**
    *   **位置**: `services/ffmpegService.ts` 中的 `srtToAss` 方法。
    *   **說明**: 目前使用 Canvas 燒錄字幕，ASS 轉換邏輯未被使用。建議移除以保持程式碼整潔。

*   **整數解析風險**
    *   **位置**: `services/ffmpegService.ts`
    *   **問題**: `parseInt(parts[0])` 未指定基數 (Radix)。雖然現代瀏覽器預設為 10，但明確指定 `parseInt(str, 10)` 是最佳實踐，避免潛在的八進位解析錯誤。

*   **錯誤處理**
    *   **位置**: `App.tsx`
    *   **問題**: `catch (err: any)` 雖然方便，但建議自定義錯誤類型，或至少確保錯誤訊息的 fallback 機制更健壯。

*   **Regex 健壯性**
    *   **位置**: `services/geminiService.ts` -> `generateSrtFromAudio`
    *   **問題**: 依賴 LLM 輸出的格式。雖然 Prompt 寫得很好，但 LLM 偶爾會輸出不可預期的字元。
    *   **建議**: 增加更嚴格的 SRT 解析驗證層，如果格式錯誤，提供重試或手動修復的機會。

---

## 5. 行動建議 (Action Plan)

建議依照以下優先順序進行修正：

1.  **修復安全性**:
    *   移除 `index.html` 中的外部 CDN 連結 (Tailwind, Import Maps)。
    *   安裝 `tailwindcss` 與 `autoprefixer` 作為 DevDependencies 並透過 PostCSS 處理。
    *   下載 `ffmpeg-core` 檔案至本地。
    *   加入 CSP Meta Tag。
2.  **清理依賴**:
    *   移除 `index.html` 中的 `esm.sh` importmap，確保 React 是由 Vite 打包的。
3.  **優化效能**:
    *   重構 `ffmpegService.ts`，確保 Canvas 操作盡量少佔用主執行緒資源。
4.  **程式碼清理**:
    *   移除 `srtToAss` 及其他未使用的輔助函式。

這份報告旨在協助專案從 "原型" 邁向 "產品級" 應用，整體的基礎建設已經相當不錯。
