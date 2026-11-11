# Render 整合驗收 — 2026-11-11

## 結論

Node.js Express 後端、同源 HTML 入口、V3 PWA／每日地圖、原始快照與四個供應商串接邏輯已整合。相較最早 ZIP，這次補上模型真的從來源 ID 選擇每日景點／飯店，再由伺服器驗證與渲染；模型不得覆寫價格、座標或網址。

## 實測

| 項目 | 結果與範圍 |
|---|---|
| npm install / npm start | 乾淨副本重新安裝與 HTTP 啟動驗證，無金鑰可使用廈門降級流程 |
| Node 測試 | 14 組通過，包含日期、連續條件、POI、路線與 API 整合 |
| 三源並行 | fixture 等待三類請求均啟動才回應，確認並行調度，並驗證重複查詢命中快取 |
| Amadeus | OAuth 併發只取一份 token、過期前續期；TWD 查詢、空報價、故障回快照 |
| AI 結構化 | OpenAI SDK JSON mode fixture 驗證；選 ID 真正更新卡片；虛構 ID／混排島區拒絕並保留 fallback |
| 高德 POI／SerpApi | fixture 驗證真實 adapter 路徑、狀態與欄位；無 key 回快照／空資料 |
| 地圖 | GCJ-02 近似轉換、示意無交通時間、部分路段失敗保留未知時間；planId 驗證來源 |
| 前端 | 同源 HTML／Leaflet／PWA 靜態檔 HTTP 200；V3 來源介面前已做瀏覽器生成、日切換與底圖檢查。本 Express 包另驗證前端語法與 HTTP 契約 |
| 聯盟注入 | fixture 驗證 Trip.com Allianceid、Klook aid；無真實佣金歸因測試 |
| 金鑰隔離 | .env.example 全空；.gitignore 排除 .env 與 node_modules；封裝時檢查內容 |
| 依賴稽核 | 打包當下 npm audit --omit=dev 無已知漏洞，非永久安全保證 |

## 未實證／仍缺功能

真實金鑰、正式帳戶 API 權限、production 機票可售性、聯盟佣金、Render 雲端部署尚未實證。資料快照未重新查證。尚無即時房價、船票庫存、下單付款、道路最佳化與營業時間排程、帳號與共享資料庫。

完整限制、金鑰設定及 Render 步驟見 README.md。不得將 mock、快照、本機驗收描述成正式即時服務成功。
