# AGENTS.md — FFT Velocity Studio

本文件寫給協助開發此專案的 AI 代理（Claude、Codex、Cursor 等）。
修改任何檔案前請先讀完本文件。規則與程式衝突時，先詢問作者，不要自行決定。

---

## 1. 專案目的

FFT Velocity Studio 是一個在瀏覽器執行的振動量測分析工具：

- 讀取 LabVIEW 輸出的 TDMS 檔（FFT 頻譜與 RAW 時域訊號）
- 套用感測器校正曲線，把電壓換算成速度、位移或加速度
- 以 1/3 倍頻程對照 VC 準則線，評估精密設備環境的振動等級
- 依「論文單欄」或「簡報 16:9」樣板輸出 PNG / SVG 圖

三個核心原則（依優先順序）：

1. **數值正確**優先於畫面好看
2. **資料不離開瀏覽器**：不上傳、不追蹤
3. **單一檔案、打開就能用**：不需要安裝或建置

---

## 2. 技術規則

- `index.html` 是單一檔案，HTML、CSS、JavaScript 全部寫在裡面
- **不使用框架、套件、npm 或建置工具**；不拆成多個 JS 檔（除非作者要求）
- 唯一允許的外部資源是 Google Fonts 的 Inter 字型（載入失敗會退回系統字型），不得新增其他 CDN
- 程式使用 `"use strict"` 與原生 DOM API；`$()` 是 `document.querySelector` 的簡寫
- 必須能以 GitHub Pages 靜態網頁運作，不得依賴後端
- 瀏覽器儲存：
  - localStorage `fvs.settings.v1`：`st.an`、`st.po`、`TPL`
  - localStorage `fvs-lower-h`：下方表格高度
  - IndexedDB `fvs`：使用者載入的檔案與上次的工作階段
  - 若修改儲存格式，key 要升版（例如 `v2`），並保留讀取舊格式的能力

---

## 3. 檔案結構

| 路徑 | 用途 |
|---|---|
| `index.html` | 分析工具本體 |
| `docs/docs.css` | 說明頁共用樣式 |
| `docs/_template.html` | 說明頁範本，不可刪除、不可直接填內容 |
| `docs/guide.html` | 使用說明首頁，也是所有說明頁的目錄 |
| `docs/{slug}.html` | 其他說明頁 |

---

## 4. index.html 的程式分區

程式依下列順序分區，每區以 `/* ================= 名稱 ================= */` 標示。
**新程式一律放進對應分區，不要另開新區塊。**

| 順序 | 分區 | 內容 |
|---|---|---|
| 1 | `<style>` | 全部樣式，顏色只用 `:root` 變數 |
| 2 | HTML 版面 | `aside` 左欄（資料／分析／圖表三個 `.pane`）＋ `main` 右側（FFT／RAW／輸出預覽） |
| 3 | `window.CAL_PRESETS` | 內建校正檔 |
| 4 | constants | `VOLT` `SENS` `QUANT` `PREF` `QEN` `VC` `PALETTE` `TPL` |
| 5 | TDMS reader | `readTDMS` `parseFFT` `parseRaw` `fftC` `fftFromRaw` `parseCal` |
| 6 | math | `sensAt` `avgCols` `bandRms` `convQ` `vcRate` |
| 7 | state | 唯一狀態物件 `st`、`compute` `computeCached` `finalizeUnits` |
| 8 | plotting | `drawSVG`（頻譜）、`drawRaw`（時域） |
| 9 | UI | `renderAll` `renderDetail` `refresh` `draw`、游標、讀值 |
| 10 | file handling | `addFiles` `addRaws`、拖放 |
| 11 | controls | 所有控制項的事件綁定、縮放平移、匯出 |
| 12 | palette | Office 風格色盤 |
| 13 | start | `init` `syncControls` 數字步進器 |

---

## 5. 資料流與更新規則

- `st` 是唯一的狀態來源：`st.ds[]` 資料組、`st.an` 分析設定、`st.po` 圖表設定
- 依修改內容呼叫對應的函式，**不要一律呼叫最重的那個**：

| 改了什麼 | 呼叫 |
|---|---|
| 資料組清單、目前選取的資料組 | `renderAll()` |
| 會改變數值的設定（單位、平均、頻譜模式…） | `refresh()` |
| 只影響外觀（顏色、線型、樣板尺寸） | `draw()` |

- **快取規則（最容易出錯）**：`computeCached()` 的 key 列出所有影響計算的參數。
  新增任何會改變計算結果的 `st.an` 欄位，**必須**把它加進這個 key，否則畫面不會更新。
  `velRows()` 的 key 同理（校正檔、單位、增益）。
- 只改名稱或顏色時，不得觸發重新計算。

---

## 6. 數值慣例（不得任意更改）

- 內部一律使用 SI 單位（速度 m/s、頻率 Hz），只在 `finalizeUnits()` 轉成顯示單位
- 電壓 → 速度：`v(f) = A(f) × VOLT[FFT 單位] ÷ 增益 ÷ S(f)`
  - `S(f)` 由 `sensAt()` 在 log–log 座標內插
  - 低於校正範圍回傳 `NaN`；高於範圍沿用最後一點
- 位移、加速度由速度除以或乘以 `2πf` 得到（`convQ`）
- 由 RAW 計算 FFT：單邊 RMS 振幅頻譜、矩形窗（`|X|·√2/N`），與 LabVIEW 輸出一致
- VC 等級：取平均頻譜的 1/3 倍頻程 RMS，在 0.99–80.8 Hz 內的最大值，對照 `VC` 陣列
- 無效值一律用 `NaN`，繪圖時跳過，**不要用 0 代替**
- 修改任何計算後，要用同一組檔案比對修改前後的 VC 等級與最高點，並寫進 commit 訊息

---

## 7. 介面規則

- 介面文字使用**繁體中文、台灣用語**
- 圖上文字（座標軸、圖例）使用**英文**，因為圖會放進論文，例如 `Frequency (Hz)`、`Velocity (µm/s)`
- 左欄的新功能一律做成一張 `.card`，放進對應的 `.pane`（資料／分析／圖表）：

```html
<div class="card">
  <div class="ct">卡片標題</div>
  <div class="row"><label class="k" for="xxx">欄位</label><select id="xxx" class="grow">…</select></div>
  <div class="hint">補充說明</div>
</div>
```

- 使用現有元件，**不要重寫**：

| 需求 | 使用 |
|---|---|
| 多選一按鈕 | `.seg` ＋ `segInit(id, get, set)` |
| 開關 | `label.sw` ＋ `input.switch` |
| 顏色＋線型＋粗細 | `styleCtl(id, obj, onChange)` |
| 綁定 `st.po` 欄位 | `bindPo(id, key, event, fn)` |
| 確認對話框 | `await ask(訊息, 按鈕)`，不用 `alert` / `confirm` |
| 暫時提示 | `notice(訊息)`；常駐狀態用 `setStatus()` |
| 數字欄位 | 直接用 `<input type="number">`，步進器會自動加上 |
| 選色 | `.colorbtn` 內放 `<input type="color">`，會自動開啟色盤 |

- 顏色只用 `:root` 的 CSS 變數；資料線預設顏色取自 `PALETTE`
- 放在 `st.an` / `st.po` 的新設定會自動存進 localStorage，但要在 `syncControls()` 補一行，重新整理後控制項才會顯示正確的值
- 可能讓瀏覽器變慢的操作（大量序列、大檔案）要先用 `ask()` 確認，門檻參考 `LOAD_LIMIT`、`LINE_LIMIT`
- 長時間計算要分段執行（`await pause(0)`），並用 `setStatus()` 顯示進度

---

## 8. 常見擴充的標準做法

### 8.1 新增內建校正檔
在 `window.CAL_PRESETS` 加一筆：
```js
{"name":"SN2001-Vertical.dat", "label":"SN2001 垂直（Vertical）", "unit":"V/(in/s)", "text":"Frequency\tSensitivity\n…"}
```
- `label` 格式：`SN編號 方向中文（English）`
- `unit` 只在檔案表頭沒寫單位時使用，值必須是 `SENS` 的 key
- `text` 是兩欄、以 tab 分隔的原始內容，包含表頭

### 8.2 新增平滑點數
在 `#aSm` 加 `<option value="ma15">15 點平均</option>` 即可，
`compute()` 會用 `/^ma(\d+)$/` 自動處理，不需改程式。

### 8.3 新增輸出樣板
1. 在 `TPL` 加一個 key，欄位與 `paper` 相同（`name w h dpi font lw lwI lwVc frame legend title tickIn`）
2. 在 `#tplSeg` 加 `<button data-v="新key" aria-pressed="false">名稱</button>`

「恢復預設」與設定儲存會自動支援。

### 8.4 新增準則等級
在 `VC` 陣列依限值由大到小插入 `["名稱", µm/s]`，勾選介面會自動產生。
若會影響結果表的好壞判斷，一併調整 `refresh()` 裡 `pill` 的條件。

### 8.5 新增單位
在 `VOLT`、`SENS` 或 `QUANT` 加 key（值為換算成 SI 的倍率），下拉選單會自動產生。
若要從檔案表頭自動判讀，在 `guessVolt()` / `guessSens()` 加規則。

### 8.6 新增平均方法
在 `#aAvg` 加 option，並在 `avgCols()` 依 method 字串加分支。

### 8.7 新增說明頁
見第 9 節。

---

## 9. 說明頁（docs/）

1. 複製 `docs/_template.html`，另存為 `docs/{slug}.html`（英文小寫＋連字號，例如 `calibration-file.html`）
2. 替換所有 `{大括號}` 佔位文字，不得殘留
3. 每個主題一個 `<section>`，`<h2>` 為區塊標題；最後一個區塊固定是「相關功能」，寫出這個主題在工具中的位置（例如：左欄「分析」→「頻譜」）
4. 樣式只用 `docs/docs.css`，不寫 inline style、不新增顏色
5. 到 `docs/guide.html` 的「其他說明」清單加上連結
6. **內容必須和程式實際行為一致**：寫公式、數值或操作步驟前，先在 `index.html` 找到對應程式碼確認

---

## 10. 禁止事項

- 不得把量測資料（TDMS）放進 repo 或寫進程式；`init()` 刻意不載入範例資料
- 不得把使用者的檔案傳到任何伺服器，不得加入分析追蹤
- 不得引入框架或套件
- 不得改變第 6 節的計算慣例，除非作者明確要求
- 不得刪除看似未使用的程式（`velMatrix`、比值圖相關、`isClip`），它們保留供日後恢復

---

## 11. 目前狀態

- 飽和（clip）偵測暫時關閉：`isClip()` 永遠回傳 `false`
- 比值圖已實作，但 `drawSVG()` 裡 `ratio=false` 停用中
- 匯出檔案時，若在 claude.ai 內開啟會使用 `window.claude` 下載介面，其他環境改用一般下載連結

---

## 12. 完成前檢查清單

- [ ] 直接開啟 `index.html`，瀏覽器 Console 沒有錯誤
- [ ] 載入一組 FFT 檔＋校正檔，單位列顯示 ✓
- [ ] 左欄三個分頁、右側三個檢視都能正常切換
- [ ] 重新整理後，設定仍然保留
- [ ] 匯出 PNG / SVG 正常
- [ ] 若改到計算：已比對修改前後的 VC 等級與最高點
- [ ] 若新增說明頁：`guide.html` 已加連結，頁面可回到工具
- [ ] 新增的介面文字為繁體中文