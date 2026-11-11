# 嶼行 AI 智慧旅遊入口網站 — Express / Render 整合版

本包保留既有 V3 平板 PWA、每日地圖、廈門 POI 與四條航線快照，改為真正的 Node.js Express 同源服務；同時補上 AI 從已取得的來源清單選擇景點與住宿的能力。不需要編譯前端。

## 1. 本機啟動

需要 Node.js 22 或更新版本。

```sh
cd ai-travel-portal
npm install
npm start
```

瀏覽 http://localhost:3000 。不填任何金鑰也能用廈門快照產生行程；其他城市沒有資料時顯示缺口，不編造景點。若要測試金鑰，僅在自己電腦複製 `.env.example` 成 `.env` 填值，不可上傳這份 `.env`。

```sh
npm test
```

14 組測試包括 HTTP 整合、API 並行與快取、AI JSON mode 與來源檢查、OAuth 快取及到期前續期、航班空結果與故障降級、路線座標轉換、導購 URL、參數與環境變數隔離。

## 2. Render 部署

1. 將解壓後的 ai-travel-portal 原始碼上傳自己的 GitHub repository。不要上傳 .env 或 node_modules。
2. 在 Render 建立 Node Web Service，連接該 repository。
3. 若 GitHub 根目錄包含 ai-travel-portal 子目錄，Root Directory 填 `ai-travel-portal`；若直接上傳其內容，Root Directory 留空。
4. Build Command：`npm ci --omit=dev`；Start Command：`npm start`；Health Check：`/api/health`。
5. 在 Environment 填 API 金鑰與聯盟 ID。Render 自動注入 PORT，程式監聽 0.0.0.0。
6. 開啟 Render 提供的 HTTPS 網址，先測 `/api/health`，再用正式需求生成行程。

可選：使用根目錄 `render.yaml` 建立 Blueprint，再於 Environment 加入金鑰。`npm install` / `npm start` 同樣可直接使用。

## 3. 環境變數

`.env.example` 全部留空，包括 PORT 與 AMADEUS_BASE_URL；實際金鑰從 `process.env` 讀取。

| 變數 | 預設／用途 |
|---|---|
| PORT | 3000，Render 注入時優先採用 |
| OPENAI_API_KEY | 未填則使用規則規劃，不呼叫模型 |
| OPENAI_MODEL | gpt-4o-mini，可改為帳戶可用的 Chat Completions JSON mode 模型 |
| AMAP_KEY | 高德 Web 服務 Key；POI 與道路查詢，請確認對應服務權限 |
| AMADEUS_CLIENT_ID / AMADEUS_CLIENT_SECRET | OAuth2 client credentials；未填使用歷史班表 |
| AMADEUS_BASE_URL | 預設 https://test.api.amadeus.com；正式環境填 https://api.amadeus.com |
| SERPAPI_API_KEY | Google 交通／預約搜尋結果摘要 |
| AFFILIATE_TRIP_ID | Trip.com Allianceid；未填則普通查詢入口 |
| AFFILIATE_KLOOK_ID | Klook aid；未填則普通查詢入口 |
| DAILY_REQUEST_LIMIT | 每 IP 每 UTC 日生成／追問上限，預設 30；0 表示拒絕生成 |
| DAILY_ROUTE_LIMIT | 每 IP 每 UTC 日路線查詢上限，預設 30 |
| TRUST_PROXY_HOPS | 預設 1，適用 Render 單層受信代理；直接架站可填 0，變更代理架構時需重設 |

使用者原指令中的 `https://://amadeus.com`、`https://://trip.com` 及 Klook URL 格式不合法，已改為有效網址。Amadeus base 僅接受上述 test/production 兩個官方 HTTPS origin，避免任意伺服器請求。

注意：test 報價是測試資料，不可對客人承諾可購買。production 需帳戶資格、供應商權限及實際查詢驗證。

## 4. 程式與資料流程

```text
ai-travel-portal/
├── .env.example / .gitignore
├── package.json / package-lock.json
├── amadeusService.js
├── server.js
├── render.yaml
├── README.md / INTEGRATION_REPORT.md
├── lib/
│   ├── aiPlanning.js       AI 來源 ID 選擇與驗證
│   ├── planning.js         偏好、同行者、預算分配與基礎行程
│   ├── routes.js           路線查詢、地圖示意與失敗降級
│   └── coordinates.js      GCJ-02 → WGS84 近似轉換
├── data/
│   ├── flights/            四條歷史班表
│   └── poi/廈門.json       景點、美食、酒店各 12 筆
├── test/                   離線與 mock 整合測試
└── public/
    ├── index.html
    ├── manifest.webmanifest / sw.js
    ├── icon-192.png / icon-512.png
    ├── vendor/             本站提供的 Leaflet JS/CSS
    └── licenses/           Leaflet / eviltransform 原始授權
```

後端解析出發地、目的地、日期、天數、人數、預算、偏好與排除景點。缺少出發地／目的地／日期時先追問；不會為了省略追問自行猜測出發地。未指定年份的月日採台北當年，若已過則推下一年；3 天為起日加 2 天。

高德 POI（三類）、Amadeus 去程／回程、SerpApi 攻略並行查詢。POI／攻略快取 10 分鐘、航班 2 分鐘、道路 5 分鐘，Map 最多 300 筆。

有 OpenAI key 時使用 `response_format: {type: "json_object"}`，要求模型輸出 replyText、tripSummary、transportAndHotel、dailyItinerary，從 POI catalog 的 ID 選擇景點、餐飲與住宿。後端檢查 ID、重複景點、每天站數、鼓浪嶼分區、日期與來源，驗證全部通過後才更新卡片；失敗保留基礎行程。AI 無法覆寫供應商票價、日期、座標或注入預訂網址。

每日行程依座標近鄰排序，並不是實際道路最佳化。後端統一建立 Trip.com / Klook 查詢連結，含已設定的追蹤 ID；每個交通、飯店與景點／餐飲項目均有入口。

## 5. 平板功能

使用 Render HTTPS 網址在三星平板 Chrome 開啟，點瀏覽器選單的「加入主畫面／安裝應用程式」。有些瀏覽器只提供建立捷徑，安裝行為以裝置為準。

- 快速搜尋、親子／美食／海景／文化偏好、輕鬆節奏與連續對話。
- 每日分頁、編號位置示意、可選 OpenStreetMap 街道底圖、高德單點及站間查詢連結。
- 有 AMAP_KEY 才開啟道路／時間查詢；鼓浪嶼強制步行。無 key 或任一段失敗時保留虛線與未知時間。所有路段成功才合計道路里程與移動時間；不是旅遊日期的路況預測。
- 預算為使用者輸入的全團／每人金額分配試算，不是供應商價格。每房暫按 2 人、成人與兒童同額分配；實際總價維持 null。
- 只保存同一瀏覽器上次行程，不是雲端帳號同步。離線可讀已快取頁面與上次行程，新生成及街道底圖需網路。Service Worker 不預抓 OpenStreetMap 圖磚。
- 地圖座標是高德 GCJ-02 近似轉為 WGS84；POI 中心不是入口，不可當精準導航。

## 6. API

`GET /api/health` 回服務與金鑰「是否已配置」，不回金鑰，也不代表外部服務已連線成功。

`POST /api/plan`：

```json
{"message":"從台北到廈門，11月28號，3天兩夜，2大1小，親子，總預算30000","previous":{},"history":[]}
```

後續對話將回應 intent 放入 previous；history 是最多 12 則 user/assistant role 與 content。單則需求最多 2000 字。

成功回覆含 replyText、tripSummary、transportAndHotel、dailyItinerary、intent、sources、aiStatus、routeAvailable、generatedAt、planId。無 key 時 aiStatus=unconfigured；模型輸出來源驗證失敗為 fallback。航班 status 為 live / test / snapshot / unavailable。

`POST /api/route`：

```json
{"planId":"上次生成回應的planId","day":1,"mode":"walking","titles":["該日依序各景點名稱"]}
```

伺服器用 planId 保存的當日原始資料查路線，不相信客戶端任意座標。名稱與順序不同回 409。planId 綁定來源 IP，最多保留 200 份、1 小時；重啟、超過容量、過期或 IP 變更後需重新生成。planId 不是使用者帳號或可分享連結，部署多實例需改為共享儲存。

## 7. 快照來源與時效

資料源自使用者原上傳「嶼行-部署包-20261111(1).zip」，保留原始來源與 fetched_at：

- TPE-XMN、TSA-XMN、XMN-TPE、XMN-TSA：原檔記載 2026-11-11，FlightsFrom 週班表彙整。
- 廈門 POI：原檔記載 2026-11-11，高德 POI 三類各 12 筆。

本次複製不代表重新向官方查證。班表依星期過濾，不包含票價、可售座位或未來有效期保證，抵達時間也不是即時確認。快照仍會作歷史參考並明確標示；正式營運需定期更新，或為過舊資料設定拒用策略。

高德目前只查已支援的廈門、上海、北京、香港；其他城市不以高德空結果冒充資料。只有廈門有本地快照；東京等境外城市沒有完整 POI 來源，會顯示資料不足。SerpApi 摘要只供參考，不代表官方預約規則。廈門／上海／北京會強制加入台胞證、支付及熱門景區實名預約核對提醒。

## 8. 營運驗收界線

這是可部署的完整原始碼整合包，不等同已完成正式商業上線驗收：

- 本次無真實供應商金鑰，真實 Amadeus / 高德 / SerpApi / OpenAI 呼叫與帳戶權限未驗證；測試採 fixture/mock，不能當即時報價成功證據。
- 聯盟 ID 與 deep link 格式須以平台核准帳戶的要求驗證；未測訂單歸因、佣金，不保證連結帶 ID 就能獲利。
- 未部署到使用者的 Render 帳號。此前平板網站與此 ZIP 為不同部署；此包新增的 AI 選擇邏輯不會自動更新先前網站。
- 未串接即時飯店房價、付款下單、船票庫存、公交換乘、營業時間排程或真正道路最佳化，亦無使用者帳號、雲端多份旅程與後台。
- IP 額度、快取與 planId 都只在單一 Node 程序記憶體；重啟會清空。公開流量須補共享限流、供應商費用上限、監控、備份、隱私告知與客服政策。不得將本機配額當作永久費用熔斷。
- 前端使用者文字採 textContent，供應商金鑰只在伺服器。前後端同源，不啟用任意跨域 CORS。公用快取不儲存 API 回應。
- Tailwind CDN 按要求保留，需要客戶端連外。OpenStreetMap 瓦片為 best-effort，正式規模化應評估有 SLA 的圖磚服務，保留來源署名並遵守使用政策。

## 官方與授權參考

- https://github.com/amadeus4dev/developer-guides/blob/master/docs/API-Keys/authorization.md
- https://github.com/amadeus4dev/developer-guides/blob/master/docs/resources/flights.md
- https://developer.amap.com/api/webservice/guide/api/search/
- https://developer.amap.com/api/webservice/guide/api/direction
- https://leafletjs.com/examples-v1/quick-start/
- https://operations.osmfoundation.org/policies/tiles/
- https://github.com/googollee/eviltransform

## 9. 雙輸出展示入口

本包另外包含 `public/demo.html`：真正獨立的單頁 HTML，CSS、JavaScript、SVG 插畫及「廈門 3 天 2 夜」資料都在一個檔案內，不需 API、外部字體或 CDN 就能展示。可直接以瀏覽器開啟檔案，也可啟動 Express 後瀏覽 `/demo.html`。

展示可切換 Day 1–3、日期、成人數、全團預算、親子／美食／海景主題，並測試「不去鼓浪嶼」、恢復展示、列印目前頁面、複製三日行程及第三方導購連結。展示對話是預設規則，沒有呼叫 AI；金額、交通時間與住宿均為固定模擬值，不是供應商資料。路線插畫不是地理或道路資訊，連結不帶真實聯盟 ID、不保證佣金。

正式入口仍為 `/` 與 `public/index.html`，保留 Express API、資料來源與金鑰隔離。獨立展示頁不會取代正式頁的業務邏輯。介面最終樣式可在使用者檢視展示後再同步調整。

## 10. 已確認版面同步

2026-11-11 使用者確認展示版後，正式首頁同步海島插畫、白底深藍／橘色樣式、深藍摘要卡與每日分頁樣式；API、資料來源與地圖流程保留。導覽列可開啟 /demo.html，正式頁不使用展示頁的模擬價格。
