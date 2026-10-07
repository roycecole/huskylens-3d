# HuskyLens 2 × Uno 互動教學（3D 網頁）

用 3D 互動說明「HuskyLens 2 + Arduino Uno 自走車如何同時使用顏色辨識與標籤辨識」：

- **零件頁**：3D 爆炸圖，引線標籤、點選聚焦、接線視圖、自動導覽、截圖
- **模擬頁**：俯視地圖上的差速驅動自走車
  - 5 個預設場景，含**環形賽道**與**十字路口（道路紅綠燈）**；也可以**自己擺設場景**（牆、障礙物、標籤、紅綠燈、色塊、起點）
  - **紅綠燈可手動控制**（自動 / 強制紅 / 強制綠），也能直接點地圖裡的燈
  - **方向鍵手動駕駛**（螢幕方向鍵、鍵盤方向鍵 / WASD），HuskyLens 仍在看，可以邊開邊觀察
  - **規則表**可編輯：車子「看到什麼 → 做什麼」
  - **車載視窗**（HuskyLens 螢幕）可拖曳、縮放、全螢幕、截圖
  - 零件開關會改變模擬行為；Uno 的「顏色 ↔ 標籤」輪流切換逐段對照 [Arduino 程式](../HuskyLens2_Uno_ColorTag/HuskyLens2_Uno_ColorTag.ino)；視覺時間軸顯示盲區
- **學習頁**：為什麼 Uno 不能用 `setMultiAlgorithm`；**四份 Arduino 程式碼（可展開、複製、下載）**、API 速查、接線、常見錯誤、網站參數 ↔ 程式常數對照、實機檢查清單、尚未驗證的地方
- **實測值匯入**：用量測程式量切換耗時，貼回網站，把「假設」換成「實測」
- **設定**：淺色 / 深色 / 跟隨系統、3D 畫質、設定檔匯出匯入

純前端靜態頁面：沒有後端、沒有 API。需求與規格見 [docs/需求規格.md](docs/需求規格.md)。

> 這是**教學模型**。標示「假設」的數值（切換耗時、視角、影格率…）沒有官方資料，請以實機量測為準。

## 開發

需要 Node.js（開發時使用 v24）。

```bash
npm install
npm run dev        # 開發伺服器 http://localhost:5173
npm test           # 單元測試（模擬核心、編輯器、設定匯出匯入）
npm run typecheck  # TypeScript 型別檢查
npm run build      # 建置到 dist/（靜態檔案，可放在任何子路徑）
npm run tokens     # 從色表重新產生 src/styles/tokens.css（淺色 + 深色 + 程式碼上色）
npm run sync-code  # 把根目錄的 Arduino 草稿複製到 src/code/（學習頁顯示的程式碼；測試會檢查兩邊一致）
```

路由使用 hash（`#/parts`），靜態主機不需要設定網址改寫；資源使用相對路徑。

## 結構

```
src/
├─ data/parts.ts      零件資料（爆炸圖、車模、模擬器共用的唯一來源）
├─ sim/               模擬核心：純 TypeScript，不依賴 React 或 three
│  ├─ params.ts       參數（來自 .ino 的常數 + 標示「假設」的數值）
│  ├─ rules.ts        規則表（看到什麼 → 做什麼）
│  ├─ vision.ts       針孔相機投影、近平面裁切、遮蔽、辨識條件
│  ├─ device.ts       虛擬 HuskyLens（持續產生影格）
│  ├─ firmware.ts     主控板程式：UnoFirmware 逐段對照 .ino、Esp32Firmware
│  ├─ car.ts          差速驅動與碰撞
│  ├─ scenarios.ts    預設場景（含環形賽道、十字路口）
│  ├─ decor.ts        裝飾展開（道路、斑馬線、虛線…）
│  ├─ measured.ts     量測程式輸出的解析與換算（實測值）
│  ├─ custom.ts       自訂場景的資料模型、驗證、JSON 匯入匯出
│  ├─ editing.ts      編輯用幾何（吸附格線、貼牆、自動朝向）
│  └─ simulation.ts   把以上串起來，固定 10 ms 步長，結果可重現
├─ three/             3D 場景（react-three-fiber）：爆炸圖、模擬地圖、編輯層、截圖、色票
├─ ui/                面板、標籤、工具列、設定對話框；ui/sim 是模擬頁元件
├─ state/             zustand：零件安裝狀態、模擬設定、場景編輯器（含復原）、設定匯出匯入
└─ pages/             首頁、零件、模擬、學習
scripts/gen-tokens.mjs  色表 → tokens.css
scripts/sync-code.mjs   Arduino 草稿 → src/code/
src/code/               學習頁顯示的程式碼（副本；原始檔在專案根目錄的 HuskyLens2_Uno_* 資料夾）
tests/                  單元測試
```

重點約束：

- `sim/` **不可** import React 或 three。
- 零件是否安裝只有一份（`state/store.ts` 的 `installed`），爆炸圖、車模、模擬器都從這裡讀。
- 模擬以固定步長推進；畫面繪製只讀取結果，不影響模擬。
- 所有顏色來自 `tokens.css`（CSS）或 `three/palette.ts`（3D），不要在元件裡寫死色碼。

## 操作小抄

| 想做的事 | 做法 |
|---|---|
| 手動開車 | 模擬頁按方向鍵 / WASD，或按地圖右下角的方向鍵（按一下就自動切到手動） |
| 強制紅燈 | 面板的「紅綠燈」選「紅」，或直接點地圖裡的燈 |
| 擺設自己的場景 | 面板按「編輯場景」；選工具後點地圖。Ctrl+Z 復原、Delete 刪除、R 旋轉、Esc 取消 |
| 改規則 | 模擬頁面板的「規則表」 |
| 存設定 | 右上角「設定」→ 匯出設定 |
| 填入實機量到的切換耗時 | 燒錄 `HuskyLens2_Uno_Timing.ino` → Serial Monitor 複製 `HLTIMING` 那一行 → 模擬頁「參數與實測值」貼上 |
| 手機上看大一點的地圖 | 面板右上的 ⌄ 收合鈕，或把把手往下拉；點底部的把手條再展開 |
| 截圖 | 3D 畫面右上角相機按鈕；車載視窗標題列也有 |

## 開發小工具

開發模式下，模擬實例會掛在 `window.__sim`，可以在瀏覽器主控台直接推進時間檢查狀態：

```js
__sim.reset(); __sim.advance(10000); __sim.snapshot()
```

## 設定與儲存

使用者設定（零件安裝、畫質、主題、模擬參數、規則表、場景、自訂場景）存在 `localStorage`。讀寫都包了 try/catch，被封鎖時照常運作；讀回來的資料會重新驗證。畫面上的「還原預設」「全部還原為預設」「還原所有設定」可以清掉自己改過的設定。
