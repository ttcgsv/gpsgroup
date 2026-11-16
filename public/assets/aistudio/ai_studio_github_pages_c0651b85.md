# Google AI Studio 開發啟動指令與 GitHub Pages 部署指南

本文件提供一段專為 **Google AI Studio** 調校的完整開發指令（Prompt）。它已內建所有前端 UI/UX 設計規範、GIS 座標轉換、國土測繪中心 WMTS 圖層、Google Apps Script API 串接協議以及防 CORS 錯誤機制。

---

## 第一步：準備您的專屬開發指令

請完整複製下方框線內的指令文字，並將開頭的 **`【https://script.google.com/macros/s/AKfycbziKeYXvhvmM_LhzRXjKoqOei2ABQ66aJcD9kRc3MGEqwkCUbS6ywOudqzVrrnyens/exec】`** 替換為您剛剛複製的網址（例如 `https://script.google.com/macros/s/AKfycbx.../exec`）。

~~~~text
你是一位資深全端工程師與 UI/UX 設計專家。請幫我開發一個「人員工作分配與 Web GIS 導航系統」的完整前端網頁應用程式。

### 一、 技術架構與部署環境要求（極重要）
1. **單一檔案架構**：請輸出一個完整、獨立、可直接執行的單一 `index.html` 檔案（內含 HTML、CSS 與 JavaScript），以便我直接上傳至 **GitHub Pages** 即可運行，不需額外執行 Node.js 打包。
2. **CDN 套件引入**：請透過穩定的 CDN 引入以下套件：
   - **Tailwind CSS**（負責現代化響應式排版與美學設計）
   - **Leaflet.js**（負責互動式 Web GIS 地圖）
   - **Proj4js**（負責台灣 TWD97 TM2 121分帶 `EPSG:3826` 精準轉換為 WGS84 經緯度 `EPSG:4326`）
   - **Lucide Icons**（負責清爽專業的系統圖示）
3. **後端 API 連線設定**：
   - API 網址常數：`const GAS_API_URL = "【請在此貼上您的 GAS WEB APP URL】";`
   - **關鍵 CORS 規範**：呼叫 Google Apps Script `doPost` 時，所有 `fetch()` 請求必須使用 `method: "POST"`，且 `headers` 必須設為 `{ "Content-Type": "text/plain;charset=utf-8" }`，`body` 為 `JSON.stringify(payload)`（切勿使用 `application/json`，否則會觸發瀏覽器 CORS OPTIONS 預檢失敗）。

---

### 二、 安全驗證與權限架構
1. **密碼不寫死在前端**：前端程式碼絕對不可出現任何明文密碼。
2. **雙密碼登入畫面**：
   - 未登入前顯示高質感全螢幕登入卡片，只需輸入單一欄位「系統通行密碼」。
   - 送出後呼叫 `{ action: "login", password }`。
   - 驗證成功後，後端會回傳 `{ success: true, role: "viewer" | "admin", token: "..." }`。請將 `token` 與 `role` 存入 `sessionStorage`，並立即呼叫 `{ action: "getAllData", token }` 載入完整資料。
   - 若後續任何 API 回傳 `code === "UNAUTHORIZED"`，自動清除憑證並導回登入畫面。
3. **角色權限區分**：
   - **`viewer`（一般使用者）**：純檢視與查詢互動模式，隱藏所有新增/編輯/刪除按鈕，停用拖曳排班功能。所有已登入的團隊成員皆可完整檢視人員的單位、姓名、電話、車輛車號與備註，方便外業直接聯繫。
   - **`admin`（管理者）**：頂部導覽列顯示「管理者模式」徽章，開啟完整 CRUD 管理功能、拖曳派工與視窗派工功能，並在右上角顯示即時雲端同步狀態提示（`雲端同步中...` / `已儲存 ✓`）。

---

### 三、 資料結構與後端 GAS API 規格
系統包含三張資料表（由 `getAllData` 一次回傳 `{ personnel, locations, schedules }`）：
1. **`Personnel`（人員）**：`{ id, unit, name, phone, plate, note }`
2. **`Locations`（工作地點）**：`{ id, unit, name, twd97_x, twd97_y }`（TWD97 TM2 121分帶座標，X 約 140000~350000，Y 約 2400000~2800000）
3. **`Schedules`（派工紀錄）**：`{ id, date, personnel_id, location_id, updated_at }`（`date` 格式固定為 `YYYY-MM-DD`）

**核心業務規則**：
- **一人一天一個地點**：每位人員在同一天最多只會被指派到 1 個工作地點（整日制）。若管理者將已有排程的人員在同日重新指派新地點，請提示確認並更新/覆蓋該員當日的地點。
- **一地點一天多人出席**：同一個工作地點在同一天可以有多位人員共同出席。

**寫入型 API 指令（皆需帶入 `token` 且僅 `admin` 可用）**：
- 新增/修改人員：`{ action: "savePersonnel", token, data: { id?, unit, name, phone, plate, note } }`
- 刪除人員：`{ action: "deletePersonnel", token, id }`
- 新增/修改地點：`{ action: "saveLocation", token, data: { id?, unit, name, twd97_x, twd97_y } }`
- 刪除地點：`{ action: "deleteLocation", token, id }`
- 單筆新增派工：`{ action: "addSchedule", token, data: { date, personnel_id, location_id } }`
- 單筆刪除派工：`{ action: "deleteSchedule", token, id }`
- 批次同步派工：`{ action: "batchSyncSchedules", token, operations: [{ type: "add"|"delete", id?, date?, personnel_id?, location_id? }] }`
- 請採用「樂觀更新（Optimistic UI Update）」：管理者操作後畫面立即反應，同時在背景呼叫 API 寫入 Google Sheets 並顯示同步狀態。

---

### 四、 Web GIS 座標轉換與圖層規格
1. **Proj4js 座標轉換定義**：
   - `proj4.defs("EPSG:3826", "+proj=tmerc +lat_0=0 +lon_0=121 +k=0.9999 +x_0=250000 +y_0=0 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=m +no_defs");`
   - 將每筆地點的 `[Number(twd97_x), Number(twd97_y)]` 轉換為 `[lng, lat]` 供 Leaflet 標記與導航使用。
2. **4 種可切換地圖底圖（右上角圖層切換器）**：
   - **臺灣通用電子地圖（預設）**：`https://wmts.nlsc.gov.tw/wmts/EMAP/default/GoogleMapsCompatible/{z}/{y}/{x}`
   - **臺灣正射影像（航照圖）**：`https://wmts.nlsc.gov.tw/wmts/PHOTO2/default/GoogleMapsCompatible/{z}/{y}/{x}`
   - **臺灣混和地圖（航照＋路網地名疊加）**：使用 `L.layerGroup` 將 `PHOTO2` 作為底層，並疊加國土測繪中心透明路網層 `https://wmts.nlsc.gov.tw/wmts/EMAP2/default/GoogleMapsCompatible/{z}/{y}/{x}`
   - **OpenStreetMap**：`https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`
3. **Google Maps 直接導航**：
   - 點擊工作地點展開後的「Google Maps 導航」按鈕，直接開啟新分頁導向：`https://www.google.com/maps/dir/?api=1&destination=${lat},${lng}&travelmode=driving`（不需要複製座標或檢視地標按鈕）。

---

### 五、 三方連動邏輯（人員 $\leftrightarrow$ 日曆 $\leftrightarrow$ 地圖）與聚焦模式
1. **視覺風格與聚焦模式（Focus Mode）**：
   - 整體採用低飽和度「現代工程灰白（Cool Slate `#F8FAFC`）」搭配「深海藍（`#0F4C81`）」主色，各「單位」自動分配柔和識別色標籤。
   - **預設狀態**：地圖顯示**全部工作地點**（標準藍色圖釘）；日曆顯示當月所有排班摘要。
   - **聚焦高亮**：當使用者選取特定人員、特定地點或特定日期時，關聯項目以高對比強調色（琥珀橘 `#F59E0B` 或翡翠綠 `#10B981` + 脈衝光環）凸顯，無關項目則適度降低透明度（`opacity-40`），讓視覺焦點一眼清晰。
   - 提供明顯的 **「清除選取 / 重置檢視」** 按鈕，一鍵恢復所有項目為乾淨的預設狀態。
2. **三向連動規則**：
   - **點擊「人員卡片」**：該人員卡片高亮；日曆上標示出此人被排定工作的日期與地點名稱；地圖上將此人前往的工作地點圖釘變換為強調色。
   - **點擊左下角「工作地點項目」**：該項目展開並顯示「Google Maps 導航」按鈕；地圖平滑移動（`flyTo`）至該地點並放大強調該圖釘；日曆與人員清單同步標示相關排程。
   - **點擊地圖上的「工作地點圖釘」**：左下角地點清單自動展開對應的主責單位抽屜、捲動至該地點並換上明顯高亮底色；同時**所有當天（或目前所選日期／當期排程）要出席該地點的人員卡片**，在左上角人員清單中全部變換為明顯高亮顏色並自動展開其單位抽屜，方便一秒辨識出席團隊。
   - **點擊日曆上的「日期格子」**：選取該日期，地圖同步變色標示該日有出勤的工作地點，清單同步顯示該日出勤人員。

---

### 六、 響應式版面配置（電腦版 vs. 手機版）
1. **電腦版（寬度 $\ge 1024\text{px}$）三欄式配置**：
   - **左欄（寬度約 `340px`）**：
     - 上半部：**人員清單**（依「單位」分為可收合的手風琴抽屜，展開後顯示人員卡片：姓名、電話、車號、備註；管理者模式可拖曳卡片，並有新增/編輯/刪除按鈕）。
     - 下半部：**工作地點清單**（上方附即時關鍵字搜尋框，下方依「主責單位」分為手風琴抽屜，點擊地點可展開「Google Maps 導航」按鈕）。
   - **中欄（彈性寬度）**：
     - 類似 Google 日曆的排程區，支援 **「月曆視角」與「日曆（單日明細）視角」切換**、月份上下切換與「今天」按鈕。
   - **右欄（寬度約 `42%`）**：
     - 滿版高度的 Leaflet Web GIS 地圖。
2. **手機版（寬度 $< 1024\text{px}$）三頁橫向滑動切換**：
   - 分為三個可左右觸控滑動（或透過頂部頁籤切換）的全寬頁面：
     - **頁 1**：單位人員清單 ＋ 搜尋與工作地點清單（頂部設有明顯的「清除選取」重置按鈕）。
     - **頁 2**：月曆 / 日曆切換排程畫面。
     - **頁 3**：滿版 Web GIS 地圖。
   - **智慧浮動方向箭頭導引（重要 UX 細節）**：
     - 在 **頁 1** 點選某位人員或某個工作地點時，**不要強制自動跳轉頁面**，而是在畫面右側邊緣浮現一個醒目的 **向右浮動箭頭按鈕（例如 `查看日曆/地圖 →`）**，提示使用者可以往右滑動或點擊前往頁 2。
     - 進入 **頁 2** 後，畫面左右兩側邊緣同時顯示 **向左浮動箭頭（`← 清單`）** 與 **向右浮動箭頭（`地圖 →`）**，提示使用者可返回頁 1 或繼續前往頁 3。
     - 進入 **頁 3** 後，畫面左側邊緣顯示 **向左浮動箭頭（`← 日曆`）** 導引返回。

---

### 七、 管理者後台排班與資料維護操作（`admin` 角色專屬）
1. **人員與地點維護**：提供簡潔的彈出表單（Modal），讓管理者新增、修改、刪除人員資料與工作地點（含 TWD97 X/Y 座標輸入與即時轉換經緯度預覽檢查）。
2. **雙軌派工機制（兼顧電腦拖曳與手機觸控）**：
   - **電腦版拖曳派工**：直接將左上方「人員卡片」拖曳至中欄日曆的某個日期格子，放開後立即彈出輕量「選擇派工地點」視窗，選定地點即完成該日派工。
   - **點擊日期視窗派工（支援手機/平板觸控與批次指派）**：點擊日曆某日期的 `+ 派工管理` 按鈕，開啟該日排程視窗，管理者可一次勾選多位人員並指定前往的工作地點，或一鍵移除當日某位人員的派工。

請確保程式碼結構清晰、註解完整、無任何語法錯誤，並具備現代化 SaaS 軟體的精緻陰影、圓角與過場動畫細節。
~~~~

---

## 第二步：在 Google AI Studio 生成網頁

1. 前往 [Google AI Studio](https://aistudio.google.com/)。
2. 建立新對話（建議選擇最新的 **Gemini Pro / Flash** 模型，並開啟較高的輸出長度上限）。
3. 將上方替換好 `GAS_API_URL` 的整段指令貼入並送出。
4. AI Studio 生成完完整的 `index.html` 後，您可以：
   - 直接點擊右上角下載，或完整複製程式碼儲存為一個名為 **`index.html`** 的檔案（請注意副檔名必須是 `.html`，編碼為 `UTF-8`）。
   - 直接在電腦上用 Chrome / Edge 雙擊開啟這個 `index.html` 檔案，即可進行本機實測（測試登入、新增地點、拖曳派工與地圖定位）。

---

## 第三步：免費上架至 GitHub Pages

當您在本機測試滿意後，只需 3 分鐘即可讓整個團隊透過網址使用：

1. 前往 [GitHub](https://github.com/) 登入帳號，點擊右上角 **`+`** $\rightarrow$ **`New repository`**。
2. 輸入專案名稱（例如 `team-gis-scheduler`），權限可選 **Public**（免費版皆支援 GitHub Pages），點擊 **`Create repository`**。
3. 在新專案頁面點擊 **`uploading an existing file`**，將您的 **`index.html`** 拖曳上傳，然後點擊下方的 **`Commit changes`**。
4. 點擊專案上方的 **`Settings`**（設定） $\rightarrow$ 左側選單找到 **`Pages`**。
5. 在 **Build and deployment $\rightarrow$ Branch** 下拉選單選擇 **`main`**（資料夾保持 `/ (root)`），點擊 **`Save`**。
6. 等待約 1～2 分鐘重新整理頁面，上方就會出現您的專屬正式網址（格式為 `https://您的帳號.github.io/team-gis-scheduler/`）！

> **💡 團隊使用小技巧**：  
> 您可以將這個網址傳給團隊成員，並告知他們「一般檢視密碼」；而您自己或主管則輸入「管理者密碼」，即可隨時在電腦或手機上調整排班！