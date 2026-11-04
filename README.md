# 控制點選點（survey-cp）專案說明

地籍測量組用的多人即時「控制點選點」網頁。手機／平板操作，主程式是單一檔案 `index.html`（另有 PWA 用的 `manifest.json`、`sw.js` 與圖示），放 GitHub Pages；資料存 Firebase Firestore（專案 `survey-cp`，免費額度）。目前版本 = v25。另有瀏覽用的 `viewer.html`（地籍瀏覽器 v1.4，見下方專節）。

## 檔案
- `index.html`：整個應用（Leaflet 1.9.4 from cdnjs、Firebase JS SDK 10.12.2 modular from gstatic）。
- PWA 檔案（和 `index.html` 放同一層）：`manifest.json`（名稱、圖示、全螢幕）、`sw.js`（離線快取）、`icon-192.png`、`icon-512.png`、`apple-touch-icon.png`（iOS 主畫面）。圖示原稿 `mockups/pwa-icons/icon-B.svg`。
- `viewer.html`：**地籍瀏覽器**（另一個網頁，以瀏覽為主；看已知點與自己匯入的地籍圖不用登入，全組分享地籍圖才用 Firebase，見 v1.2）。網址＝網站網址/viewer.html。自己的 PWA 檔：`viewer-manifest.json`、`viewer-icon-192.png`、`viewer-icon-512.png`、`viewer-apple-touch-icon.png`（v1.3 起為藍底地籍格線＋放大鏡，和選點頁的綠底區分）。
- `viewer/`：瀏覽頁的已知點。`list.txt`（一行一個檔名，`#` 開頭為註解）＋平差成果 CTL 原檔（每行「點名 N E [H]」，空白分隔）。檔名用「年度-地區.CTL」（例 `115-中埔.CTL`），年度、地區從檔名取。
- `viewer/format-docs/`：重測系統 Dxx／EXAM.MDB 格式說明（不上傳 GitHub 也可以，僅供開發參考）。
- `firestore.rules`：Firestore 規則（只允許白名單內的專案路徑，路徑 = 正規化後的密碼）。貼到 Firebase Console → Firestore → 規則。
- KML 檔（地籍圖、加密點、圖根、已知點通視）：現有兩個專案放 GitHub 根目錄；新專案放 `<dir>/`（見「新增專案步驟」）。
- `docs/`：使用說明（`控制點選點_使用說明.pdf`、來源 `guide.html` 與截圖）。重產 PDF：用 Playwright `pg.pdf(format='A4', prefer_css_page_size=True, print_background=True)` 開 `guide.html`。
- `tests/`：自動測試（Playwright + Leaflet／Firebase 假模組）。
  - `python3 ui_test3.py`（93 項）、`python3 pure_test.py`（純函式）。**請在 `tests/` 目錄內執行**，頁面路徑為 `../index.html`。
  - `guide_shots.py`：產生說明用示意截圖。

## 專案與密碼
- 兩個專案：「測試用」、「116Chiayi」（`dir:''`，KML 在根目錄）。登入用共用密碼（16 字，大小寫與橫線不拘，正規化＝小寫英數）。
- 頁面內只存密碼的 SHA-256（`PROJECTS=[{name,h,dir}]`，`resolveProject()` 用 `crypto.subtle`，回傳 `dir` 存在 `curDir`）；Firestore 路徑 `projects/<正規化密碼>/…`，規則白名單同一組密碼。
- 分享連結 `#<密碼>`（載入後會從網址移除）。`cp_pw`、`cp_name` 存 localStorage。
- 密碼明文只在 `firestore.rules` 裡，請勿公開。

## 資料
- Firestore 每專案集合：`points`（新設點）、`lines`（通視線）、`names`（點號唯一性登記，用 runTransaction）、`rootStat`（歷年圖根現況，doc id＝圖根 id，`{st:'lost'|'unused',note,by,at}`；正常＝刪除該筆）。
- 固定圖層不進 Firestore，由 GitHub 的 KML 載入（`fetch(curDir+檔名, {cache:'no-cache'})`）：位置＝根目錄（現有兩專案）或 `<dir>/`（新專案）。進入專案後才載入；換專案時先清掉舊圖層再載入新的，缺檔時狀態顯示「尚未上傳 <dir>檔名」。
  `cadastre.kml`（地籍圖，橘線，canvas）、`knownpoints.kml`（加密點）、`roots.kml`（歷年圖根）、`knownlines.kml`（已知點通視，洋紅線）。座標 WGS84、KML（非 KMZ）。
- 免費額度：Firestore 讀 5 萬／寫 2 萬（每日）；GitHub Pages 流量軟限 100 GB／月。

## 功能摘要
- 底圖：電子／正射（NLSC WMTS）／無底圖；段籍圖（NLSC LANDSECT2）與地籍圖（KML）各自開關。
- 縮放（v23）：最大 zoom 21（約 0.07 m／畫面點），方便現場長按拖曳微調點位。NLSC 底圖原生只到 19，20、21 是放大顯示（21 放大 4 倍會較糊），不多抓圖磚；點位、通視線、地籍圖 KML 為向量不受影響。自動縮放上限不變：搜尋最多 20，定位與進專案最多 19。
- 點號標籤（v24）：點位圈圈縮小為 10px（含白邊約 14px；手指可點範圍仍 44px）。每次縮放／拖動地圖結束、資料或選取改變時，點號依序試 右、右上、右下、左、左上、左下、上、下，找不會蓋到其他點或點號的位置；八個都放不下就先隱藏（放大後有空間會出現）。擺放優先序：選中的點、拖曳中的點、與選中點有通視線的點 > 新設點 > 加密點 > 圖根 > 遺失／不使用的圖根。zoom 16 以下（含）只顯示點、不顯示點號，但選中的點和它的通視點仍顯示（放不下時固定在右邊）。程式：`layoutLabels()`／`schedLabels()`，常數 `LBL_MINZ`。
- 地圖點位顏色：新設點 BI＝深綠、HI＝亮綠、自訂＝深灰；樁標未設定的新設點為空心圈（白底、外圈為該類型顏色）。加密點紫色菱形、圖根棕色方塊、遺失／不使用的圖根灰色。圖例在「圖層與資料」。
- 新增點位：BI／HI（3 位數、不重複、自動排號）或「自訂」自由輸入；橘色十字在畫面中央偏上 15%；GPS 或移動地圖。定位追蹤開著且已取得位置時（v26），按「新增點位」十字直接移到最新位置（zoom ≥ 19，記為 GPS 來源）；沒開追蹤則和以前一樣停在目前畫面，可按「使用我的位置」。移動地圖時，十字到 300 m 內最近 4 個點（新設點與顯示中的加密點、圖根）畫橘色虛線並標距離；300 m 內沒有點就不顯示。
- 地籍圖／加密點／圖根／已知點通視依專案資料夾載入。
- 點位詳細頁（新設點）：標題列＝點號、改號、收合、×；第一列＝樁標下拉選單＋通視（n），第二列＝導航＋刪除；樁標選「其他」時出現備註。新增點位預設鋼標。建立／修改紀錄與來源不在詳細頁，改在點位清單的「資訊」展開區。
- 加密點／圖根詳細頁：第一列＝通視（n）＋導航；通視清單收在「通視清單 ▸」。位置與點號為固定資料。
- 面板收合：點「收合」、點把手或標題、或往下滑，收成一條（點號・樁標或類別・通視數＋展開＋×）；點一下或往上滑展開。收合狀態存在本機（`cp_min`），點其他點位也維持。
- 歷年圖根現況：詳細頁「現況」可選正常／遺失（現場找不到）／不使用，非正常時可填現況說明，全組即時同步，可改回正常。遺失／不使用的圖根在地圖上變灰（不加叉），新增點位與定位的距離虛線不計入；點位清單顯示現況；工作進度列出遺失與不使用的數量與點號。
- 通視線：藍色（可換 10 色，僅本機）；在線上長按 0.6 秒開底部面板，顯示距離並可刪除（二次確認）。
- 選點：輕點用「距離判定」（36px 內最近的點，不依賴 Leaflet click，為了 iPad）；按住 0.8 秒內放開、位移 20px 內都算輕點；長按新設點 0.8 秒可拖曳修正位置，放開後跳出確認面板（顯示移動距離與新座標，原位置以白色圓圈＋橘色虛線標示），按「儲存新位置」才寫入，按「取消」或 × 回到原位。
- 右側按鈕（由上而下）：搜尋點位、全範圍、圖層與資料、點位清單、工作進度、定位。
  - 搜尋：點號可部分輸入、半徑（預設 200 m）決定縮放範圍；搜新設點、加密點與歷年圖根。
  - 全範圍：縮放到含約 90% 地籍圖頂點的範圍（各邊去掉最外 5%）。
  - 定位：按一下開始持續追蹤（按鈕變藍，watchPosition 高精度），再按一次停止，滿 5 分鐘自動停止。取得第一筆位置前按鈕轉圈並停用。距離虛線和新增點位一樣：連到 300 m 內最近 4 個點（新設點與顯示中的加密點、圖根，不含遺失／不使用的圖根）；300 m 內沒有點時只連最近的 1 點。第一筆位置時縮放到含自己與這些點（300 m 內沒點則以自己為中心，zoom ≥ 18）；之後地圖不跟隨，只更新藍點、精度圈與距離虛線，拖動／縮放地圖不清除。藍點有方向扇形（羅盤：iOS `webkitCompassHeading`，需在按鈕內 `DeviceOrientationEvent.requestPermission`；Android `deviceorientationabsolute`；依螢幕旋轉角修正），拒絕方向權限則只是不顯示扇形。
- 匯入新設點（圖層與資料）：
  - KML：讀 Placemark 名稱與座標，樁標預設鋼標；點名已存在則略過。
  - CSV：與匯出同格式（點名、N、E、樁標種類，TWD97；有緯度／經度欄時優先用經緯度）。標題列可有可無；統計列自動略過，「備註」列讀回該點備註。樁標空白＝鋼標，無法辨識＝未設定。**點名已存在會直接覆蓋**座標、樁標與備註（CSV 沒有備註列時保留原備註），匯入前顯示確認與覆蓋清單。座標不在台灣範圍的列略過。
- 匯入通視線（圖層與資料）：吃匯出的通視線 KML（LineString）。兩端先用座標比對 0.5 m 內最近的新設點／加密點／圖根，對不到再用線名「A-B」拆點名比對；仍對不到的線不匯入，在訊息列出線名。已存在的線略過。
- 點位清單：新設點／加密點／歷年圖根分頁；新設點的點號旁顯示樁標小標籤（未設定為紅色「未設」）。每點的「資訊」展開區＝建立／修改紀錄與來源（加密點、圖根為資料說明與現況標註者）＋座標、複製 N,E、修改座標（新設點）。
- 工作進度：BI／HI 使用號碼與跳號（含自訂點號）、樁標統計（道釘、其他、樁標未設定可點開看點號按鈕，「其他」附備註前 8 字；展開狀態在資料更新時維持）、通視線為 0／「其他」缺備註、圖根遺失與不使用。
- 匯出：新設點 CSV（點名、N、E、樁標種類；統計列含鋼標／道釘／其他／未設定／BI／HI／自訂／總計；備註列）、通視線 KML。
- PWA（v22）：可加到主畫面（Android Chrome 選單→安裝應用程式；iPad／iPhone Chrome 分享→加入主畫面，會以 iOS 網頁 App 開啟，儲存與 Chrome 分開，第一次要重新登入、重新允許 GPS／羅盤）。`sw.js` 快取：`index.html` 與 KML 先抓網路（4 秒沒回應或斷網才用快取，上傳新版重新整理即更新）；Leaflet、Firebase SDK、字型、圖示先用快取；NLSC 底圖看過才存（對方支援 CORS 時上限 2000 張，否則 300 張，超過刪最舊）；點位與通視線沿用 Firestore 離線快取。只在 https 註冊，本機檔案與 tests 不受影響。
- 即時同步：onSnapshot；回到前景（v25 起背景超過 1 分鐘，原 15 秒）、online、pageshow、點標題會重新訂閱；短暫切走交給 Firebase 自己重連以省讀取，iOS 若監聽卡住可點標題重新同步。Firestore 開 `experimentalAutoDetectLongPolling`。
- 今日讀取（v25）：「工作進度」最下面列出每人今天的讀取筆數（開網頁／回前景／有人修改／合計）與全組合計占每日上限 5 萬的百分比，並註明只記筆數、不記位置或操作。換日用美國太平洋時間，與 Firebase 免費額度同時重算（台灣冬令下午 4 點、夏令下午 3 點，自動判斷）。數字為網頁估算：監聽的第一份伺服器結果，若這台裝置 30 分鐘內監聽過只算有變動的筆數，否則算整份（至少 1 筆）；之後每筆變動算 1（含自己改的）。每台裝置一筆 `projects/<code>/usage/<太平洋日期>_<裝置id>`（day、name、dev、open、resume、edit、at），切到背景或每 5 分鐘（有變動才）上傳；同名多裝置合併顯示。只有打開工作進度時才讀（讀取數＝當天使用的裝置數）。裝置 id 存 localStorage `cp_dev`，本機計數 `cp_use`。規則不用改。

## 地籍瀏覽器（viewer.html）功能摘要
- v1.4（2026-11-03）：地籍圖清單改成資料夾式，每段一列「☑ ▸ 段名」：勾選框是整段開關，點段名展開該段的圖層（宗地界線、地號、新地號、界址點、參考點、補點）與「整段範圍／改段名／刪除」（管理者另有「分享」）。每段的開關存本機 `cv_sv`（JSON，以段 id 為 key；新段預設開宗地界線＋地號），展開狀態不存。上方「圖層」只留已知點、註記、段籍圖。拿掉段界（不再讀 D23）。已知點改成洋紅三角形（canvas 自畫 `TriMark`）。新增參考點。搜尋跳到關著的段或圖層時會自動打開。
- 參考點（v1.4）：D14 裡 COT_REF≠0 的點，點號寫成「主號.兩位」（COT_REF 3 → 100.03、30 → 101.30），存在段的 `rpts`（n、N、E、src、ll）。深青色叉叉（canvas 自畫 `XMark`，白邊），zoom 19 以上顯示、20 以上顯示點號；點一下看坐標與坐標來源。約一成參考點和界址點同坐標，詳細頁有「同位置的點」可互相切換。搜尋要輸入小數點（例 100.03、8309.）才會找參考點。v1.3 以前匯入或分享的段要重新匯入／分享才有參考點；範例 QD1753 有 7572 個，雲端分享的壓縮大小從 484 KB 變 636 KB。
- v1.3（2026-11-03）：改名「地籍瀏覽器」、新圖示；拿掉上方「已知點／地籍圖」捷徑（只留底圖切換，圖層一律在「圖層與資料」）；D20 一律稱「補點」；「地號」只顯示舊地號，另加「新地號」圖層（預設關，在舊地號下一行顯示 N101、N101-2，沒有新地號的不顯示）；新增註記；匯入按鈕下列出會讀的 6 個檔。
- 底圖：電子／正射／無底圖；全域圖層：已知點、註記、段籍圖（NLSC）；地籍相關圖層改在每段底下（見 v1.4）。開關存本機。「圖層與資料」裡已知點清單標題為「已知點(CTL)」。
- 已知點：啟動時讀 `viewer/list.txt` 列出的 CTL（不用登入，GitHub 上公開）。全部同一種樣式（洋紅圓點），zoom 16 以上顯示點號。同一點號出現在多個檔時以年度新的為準，詳細頁列出其他成果與坐標差。「圖層與資料」可逐檔勾選（存本機）。
- 地籍圖：匯入重測系統 Dxx（工作段 zip，或一次多選 D11、D13、D14；可加 D10、D20），**只存在這台裝置（IndexedDB `cv_local`），不會上傳**。可匯入多段同時顯示，清單可縮放、改名、刪除；同工作段再匯入會詢問是否取代。zip 用瀏覽器內建 DecompressionStream 解壓。只支援 TWD97 二度分帶 121°E，坐標不在台灣本島範圍的不匯入。EXAM.MDB（含姓名）不讀。
- 宗地由 D13 界址點串列＋D14 坐標組成（圓弧中點 MIDARC 會畫成圓弧），D13 地號先對 D11 舊地號、再對新地號。點宗地顯示：段、新地號、舊地號、面積（新／舊，公頃換 m²）、地目、等則、權屬、所有權人數、調查情形、周長（圖形計算）、界址點數、注記點坐標；可複製地號、導航。地中地點內含宗地會選到內含的那筆（取面積最小）。
- 界址點（v1.1）：D14 的界址點（COT_REF=0，只留 D13 用到的）匯入時一起存（點號、N、E、樁類 COT_MATTER、坐標來源 COT_SOURCE、所屬地號）。圖層「界址點」預設關，zoom 19 以上顯示小點、20 以上顯示點號；點一下看坐標、樁類、坐標來源與所屬地號（可跳到該宗地）。選到宗地時，它的界址點亮藍色（zoom 19 以上加點號），詳細頁有「界址點坐標」展開表，可「複製全部」（點號 Tab N Tab E）。v1 匯入的段沒有界址點，要重新匯入。參考點（COT_REF≠0）v1.4 起另成圖層，見上方。
- 雲端分享（v1.2）：右上角「輸入密碼」輸入**瀏覽密碼**（和選點頁的專案密碼不同，存本機 `cv_pw`），就會讀 Firestore `viewer/<正規化瀏覽密碼>/meta/index`（目錄），本機沒有或版本不同的段才下載 `chunks/<段id>_<版本>_<序號>`（整段 JSON gzip 後切 900 KB，欄位 `d` 為 Bytes；範例 QD1753 約 484 KB、1 塊）。管理者分享的段標「全組」，同事不能改名、刪除或用匯入取代；管理者刪除後，同事下次開啟自動從本機移除。自己匯入、沒分享的段標「只在本機」。
- 管理者：同一視窗「管理者登入」用 Firebase Authentication 電子郵件／密碼登入（右上角變「管理者」），可「分享」、改名（全組同步）、刪除（全組）；匯入時會問要不要分享。寫入權限由規則限制 UID。Firebase SDK 用動態 import（`firebase-app/firestore/auth` 10.12.2），沒有密碼時不載入。
- 雲端設定步驟：① Firebase Console → Authentication → 登入方式 → 啟用「電子郵件/密碼」；② 使用者 → 新增使用者（管理者自己的信箱與登入密碼），複製 UID；③ 自訂瀏覽密碼（至少 8 個英數，建議 16 字），正規化＝轉小寫、只留英數；④ Firestore → 規則，在 `projects` 區塊下面加上 `firestore.rules` 裡的 `viewer` 區塊，填入正規化瀏覽密碼與 UID，發布；⑤ 上傳 `viewer.html`；⑥ 開瀏覽頁輸入瀏覽密碼、管理者登入，到「圖層與資料」分享；⑦ 把瀏覽密碼給同事。密碼不要寫進 GitHub 或這個 README。
- 搜尋（放大鏡）：地號「101」「101-2」「1753 101」「1753段101」，新舊地號都找（舊地號標「舊」）；點號可部分輸入（已知點、顯示中的補點）；也搜尋註記文字。結果點下去會縮放並避開面板（iPad 寬螢幕面板在右側，手機在下方）。
- 註記（v1.3）：底部「＋ 新增註記」→ 移動地圖讓橘色十字對準位置（或按「GPS 定位」移到目前位置）→ 輸入文字 → 儲存。**只存在這台裝置**（IndexedDB `cv_local` 第 2 版的 `notes` store：id、t 文字、lat、lng、at 建立、up 修改），不分享、不同步。地圖上為橘色旗子，zoom 17 以上顯示前 10 字；點旗子可修改、複製文字、導航、刪除。「圖層與資料」的註記清單可「匯出 KML」（notes_年月日.kml，含建立時間）與「匯入 KML」（同文字且位置相同的略過），用來換手機或傳給同事。
- 點選：輕點用距離判定（24px 內最近的已知點／補點／界址點／參考點／註記，否則判斷點在哪個宗地內），不依賴 Leaflet click。
- 定位：按一下持續追蹤（藍點＋精度圈），再按停止，5 分鐘自動停止。沒有方向扇形與距離虛線（和選點頁不同）。
- 部署：上傳 `viewer.html`、`viewer-manifest.json`、3 個 `viewer-*.png`、`viewer/list.txt` 與 CTL，以及新版 `sw.js`（v3）。新增成果：上傳 CTL 到 `viewer/`，在 `list.txt` 加一行。

## 程式結構備忘
- `/* PURE:START */ … /* PURE:END */` 之間是純函式（TM2、parseName、nameCmp、parseKml、parseKmlLines、buildCsv、parseCsvRows、parseCsvPoints、calcProgress、buildLinesKml、matchLines、kmlColor、csvEsc、distM），由 `pure_test.py` 抽出測試。
- 模式 `mode`：idle／detail／link／add／list／layers／progress／line／search／move（拖曳後等待確認，`pendMove={id,o,n}`；等待中 onSnapshot 仍把該點顯示在新位置）。
- 點位標記為 divIcon（`.pin` 內 `<i>` 44px 點擊區＋標籤），`interactive:false`；點選由 `handleTap`（pointerup 距離判定）處理；`window.__cpt` 為測試掛鉤。樣式改變時只改 `.pin` 的 class 與文字，不 `setIcon`（iOS 手指按住時元素被換掉會收不到放開事件）。
- 點位樣式（v21）：`pinCls` 對新設點加 `bi`／`hi`／`ot`（依 `parseName`），樁標不在 `CSV_MARKS` 時加 `nm`（空心）；顏色用 `--pc`，選取／通視候選／已連線狀態優先。
- iOS 觸控防殘留（v20）：`tapS`（按下中的手指）與 `lp`（長按拖曳）的 pointerup／pointercancel 改在 document 捕獲；新的主要觸控（`isPrimary`）或超過 2 秒的紀錄會被清掉；另有 touchend／touchcancel 備援，切到背景（visibilitychange、pagehide）時全部重設。症狀是 Safari 越用越難點、重開網頁就好。
- `sw.js` v3（v1.3 換圖示與 manifest 名稱，圖示是先用快取，所以要加版本）；v2：每個網頁各自一把快取 key（v1 會把任何網頁都存成 index.html 的 key，加了 viewer.html 後會互相覆蓋）；CTL 與 `viewer/list.txt` 先抓網路；預先快取 viewer 的頁面與圖示。
- `viewer.html` 的純函式也在 `/* PURE:START */ … /* PURE:END */` 之間（TM2、parseCtl、ctlMeta、mergeKnown、parseDbf、arcPts、ringArea、inRings、buildSet、dxxKey、parseQuery、searchAll、xmlEsc、noteHead、buildNotesKml、parseNotesKml、rpNo），`window.__cv` 為測試掛鉤。
- 部署：改完把 `index.html`（與 KML 檔）上傳到 GitHub 專案根目錄；PWA 檔案第一次要一起上傳，之後只改 `index.html` 不必動；改了 `sw.js` 要把裡面的 `VER` 加一；規則變更需到 Firebase Console 貼上。

## 新增專案步驟
1. 自訂 16 字密碼（例如 `abcd-efgh-jkmn-pqrs`）。正規化＝轉小寫、只留英數（同 `normPw`）。
2. 開網站頁面，在 Chrome 主控台貼上下面一行（把密碼換成你的），印出的 hex 就是 `h`：
   `crypto.subtle.digest('SHA-256',new TextEncoder().encode('你的密碼'.toLowerCase().replace(/[^a-z0-9]/g,''))).then(b=>console.log([...new Uint8Array(b)].map(x=>x.toString(16).padStart(2,'0')).join('')))`
3. `index.html` 的 `PROJECTS` 加一筆 `{name:'117xxx',h:'<上一步的 hex>',dir:'117xxx/'}`。資料夾用專案名命名，不可含密碼。
4. `firestore.rules` 白名單加入正規化後的密碼，貼到 Firebase Console → Firestore → 規則並發布。
5. GitHub 建 `<dir>` 資料夾（例如 `117xxx/`），上傳 4 個 KML：`cadastre.kml`、`knownpoints.kml`、`roots.kml`、`knownlines.kml`。
6. 上傳新的 `index.html`。

## 注意
- 新增、改號、匯入需要網路（確保點號不重複）。斷網時可開啟頁面、看已快取的底圖與點位；刪除、改樁標、拖曳等修改先存本機，連線後由 Firestore 自動上傳。
- 座標系統 TWD97，中央經線 121°E；地籍圖僅供參考。
