# 交接文｜PANDORA CITY 線上叫號看板

> BrownDust2「2027 台北電玩展」現場叫號看板前端。
> 本文件是前端 → 後端工程的交接說明。技術規格細節見 [`data-contract.md`](./data-contract.md)。

- **Repo**：https://github.com/offerme2/pandora-city-queue
- **線上預覽**：https://offerme2.github.io/pandora-city-queue/app.html
- **前端負責**：Offerme2 設計部（Karen）
- **狀態**：前端完成，等後端串接

---

## 1. 這是什麼

一支純前端、無框架、單一 HTML 檔的看板頁。**同一個網址**同時給：

- 現場**電視 / 大螢幕**（橫向 1920×1080 基準）
- 觀眾**手機**（直向）

版型靠裝置螢幕最短邊自動判定（≤ 500px → 手機版），不是 RWD、不會中途跳版。

畫面內容（正常叫號 / 展前 / 結束 / 暫停 / 斷線）**不寫死**，前端每 4 秒向後端輪詢一次，後端回什麼就顯示什麼。

---

## 2. 交付內容

| 檔案 | 說明 |
|---|---|
| **`app.html`** | ★ 正式版。完整前端，含所有狀態畫面。 |
| **`data-contract.md`** | ★ 後端 API 回傳格式規格。**串接前必讀。** |
| `README.md` | 快速上手（本地開發、預覽參數、部署）。 |
| `image/` | 背景 / logo 素材（BrownDust2 官方）。 |
| `index.html` | 舊版，只有「叫號中」單一畫面、無狀態切換。**不使用**，留作參考。 |

> 專案上層的 `frontend-v2-tv/`、`frontend-v2-mobile/`、`OPEN-ME.html` 是更早的**靜態設計預覽稿**（一個狀態一個檔、舊視覺）。**不屬於本次交接**，工程請以 `web/app.html` 為準。

---

## 3. 後端要做的事（唯一的串接點）

`app.html` 的 `<script>` 開頭：

```js
const CONFIG = {
  endpoint : '/api/queue-state',   // ← 改成實際 API 路徑
  pollMs   : 4000,                 // 輪詢間隔（ms），可調
  staleMs  : 30000,                // 連續失敗超過這麼久 → 前端顯示斷線畫面
};

async function fetchState(){
  // const r = await fetch(CONFIG.endpoint, { cache:'no-store' });
  // if (!r.ok) throw new Error('HTTP ' + r.status);
  // return r.json();               // 回傳 data-contract.md 的 <State>
  return currentMock();             // ← 刪掉這行
}
```

**步驟：**

1. 打開 `fetch` 三行、刪掉 `return currentMock();`
2. `endpoint` 改成你的 API
3. API 回傳 JSON，格式見 `data-contract.md` §2
4. 假資料（`MOCKS` / `currentMock` / `SCHEDULE` / `LIVE_ZONES`）可整段刪除
5. 前端跨網域時，API 要開 CORS（`Access-Control-Allow-Origin`）

**前端已處理、後端不用管：** 輪詢排程、版型判定、縮放、斷線降級（stale / offline）、跳號 UI。

---

## 4. 狀態對照（畫面 ← 資料）

### 全域：`phase` 決定整個中央區

| `phase` | 畫面 | 需要的欄位 |
|---|---|---|
| `pre` | 「本活動尚未開始」+ 場次表 | `title` `message` `schedule` |
| `live` | 4 格叫號看板 | `zones[]` |
| `today_ended` | 「今日活動已結束」+ 場次表 | `title` `message` `schedule` |
| `ended` | 「本次活動已圓滿結束」 | `title` `message` |

### 單格：`zone.state` 決定每一格（只在 `phase=live`）

| `zone.state` | 該格顯示 |
|---|---|
| `serving` / `calling` | 號碼（`value`） |
| `paused` | 「暫停服務」（琥珀字）+ `note` |
| `closed` | 「本區已結束」（琥珀字） |
| `full` | `value` +「已額滿・候補中」 |

各格獨立：某區暫停不影響其他區。異常狀態外框仍保留該區顏色，只有中央文字轉琥珀色（`#f5a623`）。

### 連線（前端自動判定）

| 情況 | 畫面 |
|---|---|
| 抓取失敗、距上次成功 ≤ 30s | 畫面壓暗去彩、時間戳轉琥珀，保留最後畫面 |
| 連續失敗 > 30s | 全屏「暫時無法取得叫號資訊，請洽現場工作人員」 |

---

## 5. 部署

- `app.html` + `image/` 一起放到任何靜態主機（或塞進現有網站的一個路徑）。
- **不需要** build、Node、後端框架。
- 字型走 Google Fonts CDN。現場網路不穩可改 self-host（`.woff2` 放進 `fonts/`、改 `<link>`）。
- 現場電視：瀏覽器全螢幕開網址即可，會自動撐滿 16:9。

---

## 6. 驗收前自我檢查（工程）

- [ ] `?mock=live`（拿掉後改抓真 API）4 格號碼正確更新
- [ ] 改後端 `phase` → 前端 4 秒內換到對應畫面
- [ ] 某區 `state:"paused"` → 只有那格變「暫停服務」
- [ ] API 關掉 → 30s 內出現斷線畫面；API 恢復 → 自動回正常
- [ ] 電視 1920×1080 全螢幕：無捲軸、無黑邊
- [ ] 手機（實機）直向：4 格一個畫面看完、不用捲

---

## 7. 視覺規格（未經設計不要改）

- 四區主色：A `#22e3c9`　B `#3b7bff`　C `#9b6cff`　D `#f0429e`
- 異常狀態文字：琥珀 `#f5a623`（外框仍用該區主色）
- 卡片切角、霓虹外框、旋轉掃光：前端已定版
- 非 RWD：手機版與寬版是兩套獨立版型

---

## 8. 已知事項 / 備註

- 舊 `DELIVERY-NOTES.md` 用 `?paused=A` 這種網址參數控制暫停；**本版改為資料驅動**（`zone.state`），由後端決定。
- 舊版「暫停中的卡停止流光」目前未做；如需要可補（`is-abnormal` 時關掉 `::before` 動畫）。
- `notice`（全域跑馬燈公告）欄位已在契約中預留，前端尚未渲染，之後要再加。
- 即時性：目前是輪詢，足夠現場使用。日後要更即時可改 SSE，前端封裝好換傳輸不動畫面。
