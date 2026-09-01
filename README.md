# PANDORA CITY 線上叫號看板 — 前端

BrownDust2「2027 台北電玩展」現場叫號看板。純前端、無框架、單一 HTML 檔，
放到任何靜態主機或塞進現有網站都可以。

線上預覽：https://offerme2.github.io/pandora-city-queue/

---

## 檔案

| 檔案 | 說明 |
|---|---|
| **`app.html`** | ★ **正式版**。狀態驅動：展前 / 叫號中 / 今日結束 / 活動結束 / 單區暫停 / 斷線，全部同一頁靠後台資料切換。 |
| **`data-contract.md`** | ★ **後台串接規格**。API 回傳格式、欄位 enum、優先序、斷線行為、完整範例。**請先看這份。** |
| `index.html` | 舊版，只有「叫號中」一種畫面，無狀態切換。留著參考，正式上線用 `app.html`。 |
| `image/` | 背景與 logo 素材（BrownDust2 官方）。 |

---

## 後台工程師要做的事

只有一個串接點，在 `app.html` 的 `<script>` 開頭：

```js
const CONFIG = {
  endpoint : '/api/queue-state',   // ← 改成你的 API 路徑
  pollMs   : 4000,                 // 輪詢間隔
  staleMs  : 30000,
};

async function fetchState(){
  // const r = await fetch(CONFIG.endpoint, { cache:'no-store' });
  // if (!r.ok) throw new Error('HTTP ' + r.status);
  // return r.json();              // ← 回傳 data-contract.md 定義的 <State>
  return currentMock();            // ← 刪掉這行
}
```

1. 打開 `fetch` 那三行、刪掉 `return currentMock();`
2. 讓 `/api/queue-state` 回傳 `data-contract.md` 裡的 JSON
3. 開發期用的假資料（`MOCKS` / `currentMock()` / `SCHEDULE` / `LIVE_ZONES`）可以整段刪
4. 跨網域記得開 CORS

前端每 `pollMs` 抓一次、自動換畫面。斷線降級（stale / offline）前端自己處理，後台不用管。

---

## 本地開發

任何靜態伺服器都行，例如：

```bash
cd web
python3 -m http.server 8080
# 開 http://localhost:8080/app.html
```

### 預覽各狀態（開發用網址參數）

| 參數 | 畫面 |
|---|---|
| `?mock=live` | 正常 4 格叫號 |
| `?mock=live-paused` | B 區暫停 |
| `?mock=pre` | 活動尚未開始 |
| `?mock=today_ended` | 今日活動已結束 |
| `?mock=ended` | 本次活動已圓滿結束 |
| `?mock=stale` | 連線降級 |
| `?mock=offline` | 完全斷線 |
| `?layout=phone` / `?layout=wide` | 強制版型（正式上線靠裝置螢幕最短邊自動判：≤500px → 手機版） |

例：`app.html?mock=today_ended&layout=phone`

---

## 部署

`app.html` + `image/` 一起丟到靜態主機即可，不需要 build、不需要 Node。
字型走 Google Fonts CDN；現場網路不穩的話可改成 self-host（把 `.woff2` 放進 `fonts/`、改 `<link>`）。

---

## 視覺規格（前端已定，改動請走設計）

- 四區主色：A `#22e3c9` / B `#3b7bff` / C `#9b6cff` / D `#f0429e`
- 異常狀態（暫停 / 結束）統一琥珀色 `#f5a623`，外框仍保留該區主色
- 非 RWD：手機版與寬版是兩套固定版型，一個硬斷點切換
