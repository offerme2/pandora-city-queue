# PANDORA CITY 線上叫號看板 — 前後台資料契約

前台檔案：`app.html`（單檔、無框架、self-contained）。
同一個網址，前台每 `4s` 輪詢後台一次，後台回什麼就顯示什麼。

---

## 1. 後台要做的事

前台 `app.html` 裡只有一個串接點：

```js
const CONFIG = {
  endpoint : '/api/queue-state',   // ← 改成你的 API 路徑
  pollMs   : 4000,                 // 輪詢間隔（ms）
  staleMs  : 30000,                // 超過這麼久沒成功抓到 → 前台自己判定為斷線
};

async function fetchState(){
  const r = await fetch(CONFIG.endpoint, { cache:'no-store' });
  if (!r.ok) throw new Error('HTTP ' + r.status);
  return r.json();               // ← 回傳下面的 <State> 物件
}
```

- 打開這三行、刪掉 `return currentMock();`。
- 開發期用的 `MOCKS` / `currentMock()` / `SCHEDULE` / `LIVE_ZONES` 整段可以刪。
- 只要是 `GET` 回 JSON 就好；不需要 WebSocket（之後要即時再說）。
- CORS：前台若跟 API 不同網域，記得開 `Access-Control-Allow-Origin`。

---

## 2. `<State>` 物件格式

```jsonc
{
  "phase": "live",                 // 必填。pre | live | today_ended | ended
  "event": "2027 台北電玩展",       // 顯示在標頭的活動名
  "updatedAt": "2027-01-23T14:27:05+08:00",  // ISO8601；只有 phase=live 會顯示成「最後更新 HH:MM」
  "notice": null,                  // 預留：全域公告字串，目前前台尚未渲染

  // ↓ phase=live 時必填
  "zones": [
    {
      "id": "A",                   // 必填。A | B | C | D（決定顏色與代碼牌）
      "name": "報到區",             // 區域名稱
      "label": "目前叫號",          // 號碼上方的小字（可自由改）
      "state": "serving",          // 見 §3
      "value": "A-001",            // 顯示的號碼／梯次字串（serving/calling/full 用）
      "sub": "10:00–10:25",        // 選填。號碼下方補充（例如梯次時段）
      "note": null                 // 選填。異常狀態（paused/closed/full）的說明字
    }
  ],

  // ↓ phase=pre / today_ended / ended 時填這些（zones 可省略）
  "title": "本活動<br>尚未開始",     // 大標，用 <br> 換行
  "message": "叫號服務將於活動正式開始後啟用",  // 副標，可含 <br>
  "schedule": {                    // 選填；pre / today_ended 會顯示，ended 不顯示
    "caption": "2027 TAIPEI GAME SHOW",
    "dates": "01/21 - 01/24",
    "hours": "09:00 - 17:00"
  }
}
```

> **文案 vs 機器碼**：`phase` / `zone.state` 是固定 enum，樣式對應在前台，**不要送別的字**。
> 其餘 `event` / `label` / `value` / `title` / `message` / `note` / `schedule.*` 都是顯示字串，後台想改字直接改、前台不用動。
> `title` / `message` 允許 `<br>`；其他欄位會被 HTML 轉義。

---

## 3. `phase`（全域）— 決定整個中央區

| phase | 中央區 | 標頭 | 頁尾 |
|---|---|---|---|
| `pre` | 全屏「尚未開始」狀態卡 + 場次表 | 只有活動名 | 洽詢版 |
| `live` | 4 格叫號看板（依 `zones`） | 活動名 + 最後更新時間 | 一般提醒版 |
| `today_ended` | 全屏「今日已結束」狀態卡 + 場次表 | 只有活動名 | 洽詢版 |
| `ended` | 全屏「已圓滿結束」狀態卡（無場次表） | 只有 logo | 洽詢版 |

未知 / 缺值 → 前台當作 `live`。

## 4. `zone.state`（單格）— 只有 `phase=live` 時看

| state | 該格顯示 | 需要的欄位 |
|---|---|---|
| `serving` | `value` 號碼（正常） | `value`（`sub` 選填） |
| `calling` | 同 `serving`（預留給「剛跳號」高亮，之後加動畫） | `value` |
| `paused` | 琥珀字「暫停服務」+ `note`（預設「請依照現場工作人員指示」） | `note` 選填 |
| `closed` | 琥珀字「本區已結束」 | `note` 選填 |
| `full` | `value`（或「額滿」）+ `note`（預設「已額滿・候補中」） | `value` / `note` 選填 |

- 異常狀態（`paused`/`closed`/`full`）：**外框顏色維持該區主色**（識別用），只有中央文字轉**琥珀色**，讓現場一眼看出這格不一樣。
- 未知 state → 前台當作 `serving`（顯示 `value`）。
- 各格互相獨立：B 暫停不影響 A/C/D。

## 5. 斷線處理（前台自動，後台不用管）

| 情況 | 前台行為 |
|---|---|
| 抓取失敗，但距上次成功 ≤ `staleMs`(30s) | 整體畫面壓暗去彩、時間戳轉琥珀色，**保留最後一次的畫面**（不清空號碼） |
| 抓取連續失敗 > `staleMs` | 全屏「暫時無法取得叫號資訊，請洽現場工作人員」 |

---

## 6. 預覽網址參數（開發用）

| 參數 | 說明 |
|---|---|
| `?layout=wide` / `?layout=phone` | 強制版型（正式上線靠裝置螢幕最短邊自動判：≤500px → 手機版） |
| `?mock=live` | 正常 4 格 |
| `?mock=live-paused` | B 區暫停 |
| `?mock=pre` | 活動尚未開始 |
| `?mock=today_ended` | 今日已結束 |
| `?mock=ended` | 本次活動已圓滿結束 |
| `?mock=stale` | 模擬連線降級 |
| `?mock=offline` | 模擬完全斷線 |

例：`app.html?layout=phone&mock=pre`

---

## 7. 範例 — 一個完整的 live 回傳

```json
{
  "phase": "live",
  "event": "2027 台北電玩展",
  "updatedAt": "2027-01-23T14:27:05+08:00",
  "notice": null,
  "zones": [
    { "id": "A", "name": "報到區",   "label": "目前叫號", "state": "serving", "value": "A-001" },
    { "id": "B", "name": "商售區",   "label": "目前叫號", "state": "paused",  "note": "請依照現場工作人員指示" },
    { "id": "C", "name": "預約取貨", "label": "目前叫號", "state": "serving", "value": "C-012" },
    { "id": "D", "name": "尊榮酒吧", "label": "目前梯次", "state": "serving", "value": "第1梯次", "sub": "10:00–10:25" }
  ]
}
```
