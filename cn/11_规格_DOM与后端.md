## 026　DOM 契约 — 50 个 id 一个都不能少

**标签**：`#规格` `#DOM` `#契约` `#CSS` `#响应式`

### 026.1　`index.html` 的 50 个 id（**完整**）

`main.js` 以 `const $ = (id) => document.getElementById(id)` 取用它们。
**少一个就有一个组件静默失效**——而且不会抛错，只会「点了没反应」。

| 区域 | id |
|------|-----|
| 顶列 | `conn-badge` `theme-select` `font-range` `who-label` `logout-btn` `disconnect-btn` |
| 连接面板 | `connect-panel` `host-input` `port-input` `autoconnect-check` `connect-btn` `connect-error` `connect-env` |
| 房间 | `room-header` `room-desc` |
| 左栏 | `entity-list` `exit-pad-host` |
| 中栏 | `msg-main` `combat-pane` `msg-combat` `cmd-input` `cmd-send` `quick-main` `quick-bottom` |
| 右栏 | `stat-bars` `msg-chat` `msg-sys` |
| 叠层 | `overlay-interact` `overlay-map` `overlay-paged` `overlay-dialog` `overlay-popmenu` `float-host` `toast-host` |
| 扩充面板 | `ext-panels` `ext-tabs` `ext-body` `ext-close` |
| 登录 | `login-modal` `login-id` `login-pw` `login-email` `login-remember` `login-error` `login-submit` `login-manual` |
| 建角 | `char-modal` `char-name` `char-submit` `char-error` |

另有 `data-chat-tab="chat"` 与 `data-chat-tab="sys"` 两个属性选择器。

> **验收**：一支测试比对 `index.html` 的 id 集合与 `main.js` 的 `getElementById` 集合，
> 两者差集必须双向为空。本项目实测 **50/50 完全吻合**：没有多余的 id，也没有取用不存在的 id。

### 026.2　结构骨架

```html
<body>
  <header class="topbar">…</header>

  <div id="connect-panel" class="connect-panel">
    <div class="connect-card">…</div>
  </div>

  <main class="layout">
    <aside class="col col-left">   现场 / 出口 </aside>
    <section class="col col-center">
      <div id="room-header" class="room-header"></div>
      <div id="room-desc" class="room-desc"></div>
      <div class="center-body">        ← position:relative，叠层的定位容器
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
    <aside class="col col-right">  状态 / 聊天 </aside>
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

### 026.3　★ `[hidden]` 规则（**最前面，带 `!important`**）

```css
[hidden] { display: none !important; }
```

**这一行必须放在样式表最前面。**

`[hidden]` 的 `display:none` 来自浏览器 UA 样式表，优先级最低。本项目有大量组件设了 `display`：

```css
.connect-panel  { display: grid; }
.modal-backdrop { display: grid; }
.overlay        { display: flex; }
.entity-list    { display: flex; }
.exit-cell      { display: flex; }
.panel          { display: flex; }
```

少了那一行，`element.hidden = true` **完全没有视觉效果**——面板永远不消失，症状是「按钮好像没反应」，但实际上每次都成功了。

### 026.4　设计 token（`tokens.css`）

```
节奏  --r-sm/md/lg  --sp-1..5
字体  --font-ui  --font-mono  --font-scale  --fs-sm/base/lg  --leading
动效  --ease  --dur-fast  --dur-mid
色彩  --bg-0..3  --line  --line-soft  --fg-0..2
      --accent  --accent-soft  --accent-ink
      --hp  --mp  --good  --warn  --bad
其他  --shadow  --radius
```

三套主题以 `:root[data-theme="night|day|mud"]` 切换，**只换 token，组件样式一行不动**。

| 主题 | 特征 |
|------|------|
| `night`（缺省） | 暖褐深底 `#12100e`，强调色琥珀 `#d9a441` |
| `day` | 浅底 `#efe9dd`；服务器色码另有「绿黄青白→蓝」改写（见 §019） |
| `mud` | 纯黑 + `#aaaaaa`，`--radius: 0`，`--font-ui` 改为等宽 |

### 026.5　版面

| 断点 | 版面 |
|------|------|
| ≥ 900px | `grid-template-columns: 280px minmax(0,1fr) 320px` |
| < 900px | 单栏堆栈；左栏改横向卷动；右栏 `order:3`；动作列改直向 |
| < 560px | 顶列换行；字级缩小 |

**无障碍**：所有动效包在 `@media (prefers-reduced-motion: reduce)` 中停用；`:focus-visible` 有 2px 强调色外框。

---

## 027　`net.js` — Transport 抽象

**标签**：`#规格` `#传输` `#双后端` `#重连` `#状态机`

### 导出

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

### 后端接口（两个实作必须一致）

```javascript
{ name, available() → boolean, unavailableReason: string,
  open(host, port) → Promise, send(line) → Promise, close() → Promise }
```

| 后端 | 选用条件 | 信道 |
|------|---------|------|
| `tauriBackend` | `isTauri()` | `__TAURI__.core.invoke` + `event.listen` |
| `websocketBackend` | 否则 | `new WebSocket(`${scheme}//${location.host}/mud`)` |

`scheme` 依 `location.protocol` 为 `https:` 决定 `wss:` 或 `ws:`。

### 连接状态机

```
IDLE ──connect()──▶ CONNECTING ──成功──▶ OPEN
                          │失败              │断线
                          ▼                  ▼
                    RECONNECTING ◀───────────┘
                       │  │
                 成功──┘  └──重试上限──▶ FAILED
```

**指数退避**：`[1000, 2000, 4000, 8000, 16000, 30000]` 毫秒，最多 **8** 次。

`stopReconnect()` 把 `retries` 设为上限但**不断线**——供登录逾时止血用。

### `onState` 回呼参数

```javascript
{ state, host, port, retries, nextRetryMs?, lastError? }
```

### Tauri 后端细节

```javascript
await t.event.listen('mud://line',  e => onLine(String(e.payload ?? '')));
await t.event.listen('mud://state', e => { /* closed|error → onClosed */ });
await t.core.invoke('mud_connect', { host, port });
```

**每次 `open()` 必须先解除旧监听**，否则重连后同一行会被处理多次。

### WebSocket 后端细节

第一则消息是连接请求：

```javascript
ws.send(JSON.stringify({ host, port }));
```

之后每一则消息 = 一行指令。

**服务器→客户端的消息有两类**：

| 形态 | 判断 | 处理 |
|------|------|------|
| 状态 | 以 `{"__state"` 开头 | `JSON.parse(...).__state` |
| 游戏数据 | 其他 | 直接 `onLine(text)` |

状态对象：`{ state: 'open'|'closed'|'error', host, port, reason?, message? }`

---

## 028　Rust 端与桥接

**标签**：`#规格` `#Rust` `#Tauri` `#telnet` `#桥接` `#安全`

### 028.1　Tauri 指令（4 个）

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

### 028.3　★ `capabilities/default.json`（**缺了 `event.listen` 会被 ACL 挡**）

```json
{
  "identifier": "default",
  "windows": ["main"],
  "permissions": ["core:default", "core:event:allow-listen",
                  "core:event:allow-unlisten", "core:window:allow-close"]
}
```

`windows` 数组要对得上 `tauri.conf.json` 里窗口的 `label`（本项目明确设为 `"main"`）。

### 028.4　★ `tauri.conf.json` 的两个必要设置

```json
"app": {
  "withGlobalTauri": true,          // 纯 HTML 前端靠 window.__TAURI__，缺省是 false
  "security": { "csp": "… connect-src 'self' ipc: http://ipc.localhost" }
}
```

没有 `withGlobalTauri`，前端会判定「不在 Tauri 环境」而完全无法连接。

### 028.5　读取管线（**不可用 `read_line`**）

```
socket bytes
   → telnet::filter()          剥 IAC；WILL→DONT、DO→WONT 全部拒绝
   → 以 b'\n' 切行             去掉行尾 \r
   → String::from_utf8_lossy   坏字节降级成替换字符
   → emit("mud://line")
```

`MAX_LINE_BYTES = 64 * 1024`。两处保护缺一不可：
**① 切出来的行超过上限 → `truncate`**；
**② 缓冲区没有换行却已超过上限 → `clear()`**（否则异常服务器可以只送不换行就把内存吃光）。

`app.emit` 回传 `Err` 代表前端已关闭，此时**直接 return 结束读取任务**。

写入走 `mpsc::Sender<Vec<u8>>`（**不是 `String`**，因为协商回复是二进位），单一 writer 任务。

### 028.6　`telnet::filter(buf)` → `{ data, reply }`

```
buf[i] != IAC(255)     → data
IAC IAC                → data 推一个 0xFF
IAC SB … IAC SE        → 整段丢弃
IAC WILL(251) opt      → reply: IAC DONT(254) opt
IAC DO(253) opt        → reply: IAC WONT(252) opt
IAC WONT/DONT opt      → 吃掉，不回复
其他两字节命令        → 吃掉
串行不完整（尾端截断）  → 丢弃，不可外泄成数据
```

**验收数据**（实机抓到的开场，两份实作都用它当测试输入）：

```
输入：ff fd 18 ff fd 1f ff fd 27 ff fb 56 ff fb 46 ff fb 2a 0d 0a "ver1.0:" 0d 0a
data ：\r\nver1.0:\r\n
reply：IAC WONT 18, IAC WONT 1f, IAC WONT 27, IAC DONT 56, IAC DONT 46, IAC DONT 2a
```

> `0x56` = MCCP2。**必须拒绝**，答应了之后整条连接会变成 zlib 串流。

### 028.7　`bridge/server.mjs`

| 参数 | 缺省 | 说明 |
|------|------|------|
| `--port` | 8080 | HTTP + WebSocket |
| `--mud-host` | 127.0.0.1 | 转发目标 |
| `--mud-port` | 5001 | 同上 |
| `--bind` | 0.0.0.0 | 监听地址 |
| `--allow-any` | 关 | 允许前端指定其他地址 |

**三个职责**：供应 `src/` 静态档、接 WebSocket、代开 TCP 并做 IAC 剥除与分行。
**它不解析 ZJMUD 协议**——opcode 解析全在前端，与 Tauri 版共用同一份 `protocol.js`。

**两条安全规则**：

```javascript
// ① 防目录穿越：解析后必须仍在 WEB_ROOT 底下
const file = path.resolve(WEB_ROOT, '.' + rel);
if (!file.startsWith(WEB_ROOT)) return res.writeHead(403).end('Forbidden');

// ② 不是开放代理：缺省只允许连到启动时设置的目标
if (!ALLOW_ANY && want.host && (want.host !== MUD_HOST || Number(want.port) !== MUD_PORT)) {
  sendState({ state: 'error',
    message: `此桥接只允许连到 ${MUD_HOST}:${MUD_PORT}（要放宽请以 --allow-any 启动）` });
  ws.close();
}
```

两条都有专门的测试守着。

> 💡 君之一席话
> **双形态的代价不是写两份，而是要证明两份行为一致——所以那 42 个实机字节，必须同时是 Rust 测试和 JS 测试的输入。**

> 🔍 老手视角──真正的门道
> 这一节最容易被低估的是 026.3 和 026.4 那两个 JSON 设置：它们不在任何原代码里，却能让一个编译零错误、测试全绿的程序**完全无法连接**。老手交接跨框架项目时，会刻意把「设置档里的关键开关」单独列一节，而不是散落在架构说明中——因为设置档的错误不会出现在编译期、不会出现在单元测试，只会在运行期以一句语焉不详的错误现身。本项目的 Tauri ACL 问题就是这样：`event.listen not allowed`，而 `event.listen` 明明存在。可落地的做法：任何「不写就会坏、但编译器不管」的设置，都值得在规格里加一个 ★ 标记与一句「没有它会怎样」。
