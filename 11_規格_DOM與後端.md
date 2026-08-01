## 026　DOM 契約 — 50 個 id 一個都不能少

**標籤**：`#規格` `#DOM` `#契約` `#CSS` `#響應式`

### 026.1　`index.html` 的 50 個 id（**完整**）

`main.js` 以 `const $ = (id) => document.getElementById(id)` 取用它們。
**少一個就有一個元件靜默失效**——而且不會拋錯，只會「點了沒反應」。

| 區域 | id |
|------|-----|
| 頂列 | `conn-badge` `theme-select` `font-range` `who-label` `logout-btn` `disconnect-btn` |
| 連線面板 | `connect-panel` `host-input` `port-input` `autoconnect-check` `connect-btn` `connect-error` `connect-env` |
| 房間 | `room-header` `room-desc` |
| 左欄 | `entity-list` `exit-pad-host` |
| 中欄 | `msg-main` `combat-pane` `msg-combat` `cmd-input` `cmd-send` `quick-main` `quick-bottom` |
| 右欄 | `stat-bars` `msg-chat` `msg-sys` |
| 疊層 | `overlay-interact` `overlay-map` `overlay-paged` `overlay-dialog` `overlay-popmenu` `float-host` `toast-host` |
| 擴充面板 | `ext-panels` `ext-tabs` `ext-body` `ext-close` |
| 登入 | `login-modal` `login-id` `login-pw` `login-email` `login-remember` `login-error` `login-submit` `login-manual` |
| 建角 | `char-modal` `char-name` `char-submit` `char-error` |

另有 `data-chat-tab="chat"` 與 `data-chat-tab="sys"` 兩個屬性選擇器。

> **驗收**：一支測試比對 `index.html` 的 id 集合與 `main.js` 的 `getElementById` 集合，
> 兩者差集必須雙向為空。本專案實測 **50/50 完全吻合**：沒有多餘的 id，也沒有取用不存在的 id。

### 026.2　結構骨架

```html
<body>
  <header class="topbar">…</header>

  <div id="connect-panel" class="connect-panel">
    <div class="connect-card">…</div>
  </div>

  <main class="layout">
    <aside class="col col-left">   現場 / 出口 </aside>
    <section class="col col-center">
      <div id="room-header" class="room-header"></div>
      <div id="room-desc" class="room-desc"></div>
      <div class="center-body">        ← position:relative，疊層的定位容器
        <div class="scroll-pane"><div id="msg-main" class="msg-list"></div></div>
        <section id="combat-pane" class="panel combat-pane" hidden>…</section>
        <div id="overlay-interact" class="overlay sheet" hidden></div>
        <div id="overlay-map"      class="overlay overlay-plain" hidden></div>
        <div id="overlay-paged"    class="overlay overlay-plain" hidden></div>
        <div id="float-host" class="float-host"></div>
      </div>
      <div class="composer">…</div>
      <div id="quick-main"   class="quick-row"></div>
      <div id="quick-bottom" class="quick-row quick-row-bottom"></div>
    </section>
    <aside class="col col-right">  狀態 / 聊天 </aside>
  </main>

  <div id="ext-panels" class="ext-panels" hidden>…</div>
  <div id="login-modal" class="modal-backdrop" hidden>…</div>
  <div id="char-modal"  class="modal-backdrop" hidden>…</div>
  <div id="overlay-dialog"   class="modal-backdrop" hidden></div>
  <div id="overlay-popmenu"  class="modal-backdrop" hidden></div>
  <div id="toast-host" class="toast-host"></div>

  <script type="module" src="js/main.js"></script>
</body>
```

### 026.3　★ `[hidden]` 規則（**最前面，帶 `!important`**）

```css
[hidden] { display: none !important; }
```

**這一行必須放在樣式表最前面。**

`[hidden]` 的 `display:none` 來自瀏覽器 UA 樣式表，優先級最低。本專案有大量元件設了 `display`：

```css
.connect-panel  { display: grid; }
.modal-backdrop { display: grid; }
.overlay        { display: flex; }
.entity-list    { display: flex; }
.exit-cell      { display: flex; }
.panel          { display: flex; }
```

少了那一行，`element.hidden = true` **完全沒有視覺效果**——面板永遠不消失，症狀是「按鈕好像沒反應」，但實際上每次都成功了。

### 026.4　設計 token（`tokens.css`）

```
節奏  --r-sm/md/lg  --sp-1..5
字體  --font-ui  --font-mono  --font-scale  --fs-sm/base/lg  --leading
動效  --ease  --dur-fast  --dur-mid
色彩  --bg-0..3  --line  --line-soft  --fg-0..2
      --accent  --accent-soft  --accent-ink
      --hp  --mp  --good  --warn  --bad
其他  --shadow  --radius
```

三套主題以 `:root[data-theme="night|day|mud"]` 切換，**只換 token，元件樣式一行不動**。

| 主題 | 特徵 |
|------|------|
| `night`（預設） | 暖褐深底 `#12100e`，強調色琥珀 `#d9a441` |
| `day` | 淺底 `#efe9dd`；伺服器色碼另有「綠黃青白→藍」改寫（見 §019） |
| `mud` | 純黑 + `#aaaaaa`，`--radius: 0`，`--font-ui` 改為等寬 |

### 026.5　版面

| 斷點 | 版面 |
|------|------|
| ≥ 900px | `grid-template-columns: 280px minmax(0,1fr) 320px` |
| < 900px | 單欄堆疊；左欄改橫向捲動；右欄 `order:3`；動作列改直向 |
| < 560px | 頂列換行；字級縮小 |

**無障礙**：所有動效包在 `@media (prefers-reduced-motion: reduce)` 中停用；`:focus-visible` 有 2px 強調色外框。

---

## 027　`net.js` — Transport 抽象

**標籤**：`#規格` `#傳輸` `#雙後端` `#重連` `#狀態機`

### 匯出

```javascript
export function isTauri() → boolean          // globalThis.__TAURI__ != null
export function environment() → 'tauri' | 'browser'
export function createTransport({ onLine, onState }) → TransportAPI
```

### `TransportAPI`

```javascript
{ connect(host, port), send(line), close(), retryNow(), stopReconnect(),
  get state(), get backend() }
```

### 後端介面（兩個實作必須一致）

```javascript
{ name, available() → boolean, unavailableReason: string,
  open(host, port) → Promise, send(line) → Promise, close() → Promise }
```

| 後端 | 選用條件 | 通道 |
|------|---------|------|
| `tauriBackend` | `isTauri()` | `__TAURI__.core.invoke` + `event.listen` |
| `websocketBackend` | 否則 | `new WebSocket(`${scheme}//${location.host}/mud`)` |

`scheme` 依 `location.protocol` 為 `https:` 決定 `wss:` 或 `ws:`。

### 連線狀態機

```
IDLE ──connect()──▶ CONNECTING ──成功──▶ OPEN
                          │失敗              │斷線
                          ▼                  ▼
                    RECONNECTING ◀───────────┘
                       │  │
                 成功──┘  └──重試上限──▶ FAILED
```

**指數退避**：`[1000, 2000, 4000, 8000, 16000, 30000]` 毫秒，最多 **8** 次。

`stopReconnect()` 把 `retries` 設為上限但**不斷線**——供登入逾時止血用。

### `onState` 回呼參數

```javascript
{ state, host, port, retries, nextRetryMs?, lastError? }
```

### Tauri 後端細節

```javascript
await t.event.listen('mud://line',  e => onLine(String(e.payload ?? '')));
await t.event.listen('mud://state', e => { /* closed|error → onClosed */ });
await t.core.invoke('mud_connect', { host, port });
```

**每次 `open()` 必須先解除舊監聽**，否則重連後同一行會被處理多次。

### WebSocket 後端細節

第一則訊息是連線請求：

```javascript
ws.send(JSON.stringify({ host, port }));
```

之後每一則訊息 = 一行指令。

**伺服器→客戶端的訊息有兩類**：

| 形態 | 判斷 | 處理 |
|------|------|------|
| 狀態 | 以 `{"__state"` 開頭 | `JSON.parse(...).__state` |
| 遊戲資料 | 其他 | 直接 `onLine(text)` |

狀態物件：`{ state: 'open'|'closed'|'error', host, port, reason?, message? }`

---

## 028　Rust 端與橋接

**標籤**：`#規格` `#Rust` `#Tauri` `#telnet` `#橋接` `#安全`

### 028.1　Tauri 指令（4 個）

```rust
#[tauri::command] async fn mud_connect(app, state, host: String, port: u16) -> Result<(), String>
#[tauri::command] async fn mud_send(state, line: String)                    -> Result<(), String>
#[tauri::command] async fn mud_disconnect(state)                            -> Result<(), String>
#[tauri::command] async fn mud_is_connected(state)                          -> Result<bool, String>
```

### 028.2　事件

```rust
pub const EVENT_LINE:  &str = "mud://line";   // payload: &str（一行，已去 \r\n）
pub const EVENT_STATE: &str = "mud://state";  // payload: ConnState
```

```rust
#[serde(tag = "state", rename_all = "lowercase")]
enum ConnState { Open{host,port}, Closed{reason}, Error{message} }
```

### 028.3　★ `capabilities/default.json`（**缺了 `event.listen` 會被 ACL 擋**）

```json
{
  "identifier": "default",
  "windows": ["main"],
  "permissions": ["core:default", "core:event:allow-listen",
                  "core:event:allow-unlisten", "core:window:allow-close"]
}
```

`windows` 陣列要對得上 `tauri.conf.json` 裡視窗的 `label`（本專案明確設為 `"main"`）。

### 028.4　★ `tauri.conf.json` 的兩個必要設定

```json
"app": {
  "withGlobalTauri": true,          // 純 HTML 前端靠 window.__TAURI__，預設是 false
  "security": { "csp": "… connect-src 'self' ipc: http://ipc.localhost" }
}
```

沒有 `withGlobalTauri`，前端會判定「不在 Tauri 環境」而完全無法連線。

### 028.5　讀取管線（**不可用 `read_line`**）

```
socket bytes
   → telnet::filter()          剝 IAC；WILL→DONT、DO→WONT 全部拒絕
   → 以 b'\n' 切行             去掉行尾 \r
   → String::from_utf8_lossy   壞位元組降級成替換字元
   → emit("mud://line")
```

`MAX_LINE_BYTES = 64 * 1024`。兩處保護缺一不可：
**① 切出來的行超過上限 → `truncate`**；
**② 緩衝區沒有換行卻已超過上限 → `clear()`**（否則異常伺服器可以只送不換行就把記憶體吃光）。

`app.emit` 回傳 `Err` 代表前端已關閉，此時**直接 return 結束讀取任務**。

寫入走 `mpsc::Sender<Vec<u8>>`（**不是 `String`**，因為協商回覆是二進位），單一 writer 任務。

### 028.6　`telnet::filter(buf)` → `{ data, reply }`

```
buf[i] != IAC(255)     → data
IAC IAC                → data 推一個 0xFF
IAC SB … IAC SE        → 整段丟棄
IAC WILL(251) opt      → reply: IAC DONT(254) opt
IAC DO(253) opt        → reply: IAC WONT(252) opt
IAC WONT/DONT opt      → 吃掉，不回覆
其他兩位元組命令        → 吃掉
序列不完整（尾端截斷）  → 丟棄，不可外洩成資料
```

**驗收資料**（實機抓到的開場，兩份實作都用它當測試輸入）：

```
輸入：ff fd 18 ff fd 1f ff fd 27 ff fb 56 ff fb 46 ff fb 2a 0d 0a "ver1.0:" 0d 0a
data ：\r\nver1.0:\r\n
reply：IAC WONT 18, IAC WONT 1f, IAC WONT 27, IAC DONT 56, IAC DONT 46, IAC DONT 2a
```

> `0x56` = MCCP2。**必須拒絕**，答應了之後整條連線會變成 zlib 串流。

### 028.7　`bridge/server.mjs`

| 參數 | 預設 | 說明 |
|------|------|------|
| `--port` | 8080 | HTTP + WebSocket |
| `--mud-host` | 127.0.0.1 | 轉發目標 |
| `--mud-port` | 5001 | 同上 |
| `--bind` | 0.0.0.0 | 監聽位址 |
| `--allow-any` | 關 | 允許前端指定其他位址 |

**三個職責**：供應 `src/` 靜態檔、接 WebSocket、代開 TCP 並做 IAC 剝除與分行。
**它不解析 ZJMUD 協議**——opcode 解析全在前端，與 Tauri 版共用同一份 `protocol.js`。

**兩條安全規則**：

```javascript
// ① 防目錄穿越：解析後必須仍在 WEB_ROOT 底下
const file = path.resolve(WEB_ROOT, '.' + rel);
if (!file.startsWith(WEB_ROOT)) return res.writeHead(403).end('Forbidden');

// ② 不是開放代理：預設只允許連到啟動時設定的目標
if (!ALLOW_ANY && want.host && (want.host !== MUD_HOST || Number(want.port) !== MUD_PORT)) {
  sendState({ state: 'error',
    message: `此橋接只允許連到 ${MUD_HOST}:${MUD_PORT}（要放寬請以 --allow-any 啟動）` });
  ws.close();
}
```

兩條都有專門的測試守著。

> 💡 君之一席話
> **雙形態的代價不是寫兩份，而是要證明兩份行為一致——所以那 42 個實機位元組，必須同時是 Rust 測試和 JS 測試的輸入。**

> 🔍 老手視角──真正的門道
> 這一節最容易被低估的是 026.3 和 026.4 那兩個 JSON 設定：它們不在任何原始碼裡，卻能讓一個編譯零錯誤、測試全綠的程式**完全無法連線**。老手交接跨框架專案時，會刻意把「設定檔裡的關鍵開關」單獨列一節，而不是散落在架構說明中——因為設定檔的錯誤不會出現在編譯期、不會出現在單元測試，只會在執行期以一句語焉不詳的錯誤現身。本專案的 Tauri ACL 問題就是這樣：`event.listen not allowed`，而 `event.listen` 明明存在。可落地的做法：任何「不寫就會壞、但編譯器不管」的設定，都值得在規格裡加一個 ★ 標記與一句「沒有它會怎樣」。
