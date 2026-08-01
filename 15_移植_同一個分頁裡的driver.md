## 040　三個沒有瀏覽器對應物的東西 — Transport、反轉的迴圈、VFS

**標籤**：`#WebAssembly` `#移植` `#介面抽取` `#事件迴圈` `#連結期分派`
**證據等級**：🟡 上游原始碼與文件（`fluffos/src/wasm/README.md`、`src/net/transport.h`）

**起源**：§032 從**客戶端**這一側寫了「伺服器搬進分頁」這件事——
症狀、三次誤判、真因。這一節換到**driver**那一側：
一個 1989 年設計、假設自己獨佔一個 Unix 行程的東西，
要怎麼搬進一個「不准阻塞、沒有 socket、沒有檔案系統」的分頁裡？

上游的答案很乾淨：**只有三樣東西在瀏覽器裡沒有對應物，就只抽三個接縫。**

**技術核心**：

### 接縫一：socket → `Transport`

原本 `comm.cc` 直接對 fd 讀寫。移植的做法是抽出一個抽象位元組管道：

```cpp
// net/transport.h — 每個 interactive_t 擁有一個
write() / flush() / schedule_command() / close()
```

三個實作，**連結期擇一**：

| 實作 | 檔案 | 用在哪 |
|------|------|--------|
| `SocketTransport` | `net/transport_libevent.cc` | 原生 telnet（bufferevent／TLS） |
| `WebsocketTransport` | 同上 | 原生 WebSocket（libwebsockets） |
| `WasmConsoleTransport` | `wasm/comm_wasm.cc` | **瀏覽器分頁** |

關鍵是它上面的東西**一行都沒改**：`comm.cc`（使用者、指令佇列、`input_to`、
提示、snoop）與 `net/telnet.cc`（完整的 telnet 協商，走 libtelnet）
在每個平台上編譯的是同一份原始碼。

`WasmConsoleTransport` 的兩個方向：
- **出**：位元組交給 JS 的 `Module.fluffos.onOutput(id, bytes)`；
- **入**：`fluffos_input(id, bytes)` → `comm_telnet_received()`——
  **和 socket 讀取走的是同一條路徑**。

而 `fluffos_connect()` 則是原生 accept 流程的鏡像：
`user_add()` → telnet 初始化與初始協商 → `master::connect(port)` → `logon()`。
沒有監聽 socket（`init_user_conn` 是空實作）。

> **這件事對客戶端的直接後果**：driver 送出的是**真正的 telnet 串流**。
> 沒有中間程序可以代勞剝 IAC，瀏覽器得自己來——
> 這就是本專案 `src/js/telnet.js` 存在的理由（§附錄 B）。

### 接縫二：事件迴圈被反轉

原生 driver 永遠阻塞在 libevent 的 `event_base_loop()` 裡。
**分頁不能阻塞——事件迴圈是頁面的，不是你的。**

上游的處理方式很值得學：**只把「推進時間的那個迴圈」換掉，排程核心留在原地**。
gametick 佇列、`add_gametick_event()`、各種維護事件全都還在共用的 `backend.cc`；
`wasm/backend_wasm.cc` 只換掉迴圈本身，用一個純粹的 wall-time 優先佇列，
**完全不用 libevent**。頁面在計時器裡呼叫 `fluffos_tick(now_ms)`，
它排掉所有到期的事件，並把時間推進「已經過了幾個 gametick」
（分頁被暫停時會補跑，但**有上限**——最多 100 拍）。

心跳、`call_out`、reset、回收——driver 的所有排程本來就走這個 API，
**所以它們上面的一切都不知道自己被移植了**。

### 接縫三：檔案系統 → MEMFS

driver 的檔案 I/O（編譯、`save_object`、`read_file`、`ed`、log）
是普通的 POSIX 加 `ghc::filesystem`，Emscripten 直接把它映到記憶體檔案系統上——
**driver 端零修改**。

mudlib 用 Emscripten 的 `file_packager` 打包成單一 `mudlib.data` 映像
加一個 `mudlib.js` 載入器，在 runtime 起來之前掛載好。
寫入落在 MEMFS，**只活到分頁關掉為止**。

**解決的痛點**：三個接縫換來一件事——**共用邏輯裡沒有任何 `#ifdef __EMSCRIPTEN__`**。
上游文件明講：平台條件編譯只剩兩處（`net/net_compat.h` 的型別宣告，
以及 `tracing.cc` 裡一個執行緒能力判斷）。
其他 per-target 的單例全部用同一招——**連結期選實作**：

| 介面 | 原生 | WASM |
|------|------|------|
| 事件迴圈（`backend.h`） | `backend_libevent.cc` | `wasm/backend_wasm.cc` |
| 連線傳輸 | `net/transport_libevent.cc` | `wasm/comm_wasm.cc` |
| TLS（`net/tls.h`） | `net/tls.cc` | **完全不連結**（唯一的共用呼叫點 `sys_reload_tls` 從該平台排除） |
| DNS 解析 | `dns_libevent.cc` | `dns_stub.cc` |
| 當機處理 | `crash_handler.cc`（backward-cpp） | `crash_handler_wasm.cc` |

**踩過的坑（上游的，不是本專案的）**：TLS 那一列的處理方式特別誠實——
**不是給一個假的實作，是連呼叫它的那個 efun 都從這個平台移除**。
所以 WASM 版的 mudlib 呼叫 `sys_reload_tls()` 會得到「這個函式不存在」的編譯錯誤，
而不是一個安靜地什麼都不做的樁。
**在移植層，「明確地不存在」比「存在但無效」好**——
後者會讓 mudlib 以為自己開了 TLS。

**優點 / 罩門**：優點是這個移植沒有製造第二份 driver：
telnet 協商、使用者管理、`input_to`、提示、snoop、編譯器、VM 全部是同一份 code，
所以「原生會動、WASM 不會動」這種分岔幾乎不可能出現在上層。
罩門是**下層分岔全部集中在三個檔案裡**，而這三個檔案的行為差異
（沒有網路延遲、沒有 eval limit、沒有真 DNS）會以非常間接的方式冒出來——
§042 的三個陷阱全部屬於這一類。

**效益**：對本專案而言，這三個接縫直接決定了客戶端要負責什麼：

| 接縫 | 客戶端因此必須做的事 | 本專案的哪個檔案 |
|------|-------------------|-----------------|
| Transport 是位元組管道 | 自己剝 telnet IAC、自己分行 | `src/js/telnet.js` |
| 事件迴圈由頁面驅動 | 自己用 `setInterval` 泵 `fluffos_tick` | `src/js/wasmdriver.js` |
| 檔案系統是 MEMFS | 自己下載映像、掛載、清楚告知「重整即失」 | `src/js/mudlibimage.js`、`wasmboot.js` |

> 💡 君之一席話
> **移植一個東西時，先數清楚「目標環境真的沒有的東西」有幾樣**——通常比想像的少；抽那幾個接縫，其餘一行都別碰，否則你得到的不是移植，是分岔。

> 🔍 老手視角──真正的門道
> 「連結期分派」這一招在今天有點被遺忘，值得重新提一下：同一個標頭、多個 `.cc`、建置系統選一個連進去——沒有虛擬函式表的成本、沒有 `#ifdef` 把共用檔案切得七零八落、也沒有執行期的分支。它的隱形好處是**編譯器會逼你把介面補完**：少實作一個函式就是連結錯誤，而 `#ifdef` 少寫一段只會安靜地少一段行為。老手在做跨平台時的順序通常是：先用 `#ifdef` 快速跑通（探索期），再把 `#ifdef` 收斂成介面 + 多實作（穩定期）——**跳過第一步會過度設計，停在第一步會腐爛**。可落地的判準：當同一個 `#ifdef` 在三個以上的檔案裡出現，就該把它換成一個介面了。

---

## 041　被拿掉的東西 — 套件矩陣，與 mudlib 上看得見的形狀

**標籤**：`#套件` `#編譯期開關` `#ICU` `#編碼` `#落差揭露`
**證據等級**：🟡 上游 `src/wasm/README.md` 的套件矩陣 ＋ 🟢 本專案實測（17 個 lib 的 `NOTES.md`）

**起源**：§036 說過，efun 是按套件（package）打包的，套件是**編譯期開關**。
WASM 版關掉了一批。這一節要回答的是：
**關掉的那些，在 mudlib 上長什麼樣子？**

這件事本專案有第一手數據——17 個 mudlib 各自的 `NOTES.md` 就是逐個量出來的落差表。

**技術核心**：先看第三方函式庫這一層。

| 元件 | WASM 版 | 為什麼 |
|------|--------|--------|
| libevent | **移除** | 換成 host 驅動的 tick 佇列（§040） |
| libwebsockets、`net/websocket.cc` | **移除** | 頁面**就是**客戶端，沒有監聽 socket |
| OpenSSL、`net/tls.cc` | **移除** | TLS 由瀏覽器負責，不需要在這裡終結 |
| **libtelnet、`net/telnet.cc`** | **保留** | 純 C、可攜；**頁面講的是真 telnet** |
| **ICU** | **保留**（交叉編譯） | 字素迭代、字集轉換、`sprintf` 寬度計算 |
| libpcre（8.x） | **保留**（JIT 關閉——wasm 沒有可執行頁） | `pcre_*` efun |
| zlib | **移除** | MCCP 與 compress 套件都關了 |
| jemalloc | **移除** | 換成 emscripten 的 dlmalloc |
| backward-cpp | **移除** | wasm 沒有原生 unwinder |
| POSIX eval-limit 計時器 | **自動停用** | 只支援 `__linux__` |

再看套件矩陣：

| 開著 | 關掉 | 只有 WASM 才有 |
|------|------|--------------|
| core、ops、math、matrix、trim、uids、sha1、parser、contrib、develop、mudlib_stats、**pcre** | **sockets**、compress、external、async、**db**、crypto、ffi | **jsbridge**（`js_eval()` / `js_call()` / `js_export()`：LPC ↔ 頁面 JavaScript 雙向） |

**踩過的坑（本專案實測）：關掉的套件不會在開機時告訴你。**

`libs/lpmudname/NOTES.md` 記的是這樣：

> WASM build 沒有 `package sockets`，以下 preload daemon 在載入時編譯失敗
> （`Undefined function socket_create` 等共 13 處）：
>
> | 物件 | 原本做什麼 | 在 WASM 上的後果 |
> |------|-----------|-----------------|
> | `/adm/daemons/kuafu` | 跨服連線（socket 監聽） | 跨服功能不存在 |
> | `/adm/daemons/qqd` | QQ 相關對外服務 | 不存在 |
> | `/adm/daemons/miraid` | 對外通報 | 不存在 |
>
> driver **照常完成開機**——這三個是 `preload` 清單裡的項目，
> 載入失敗只會讓那個物件不存在，不會中止啟動。

這正是 §038 講的：**preload 失敗是靜默的**。
所以「開得起來」與「功能完整」是兩件事，而 driver 不會替你區分。
本專案的分級（`playable` / `limited` / `noboot`）
之所以要靠**實際跑一遍註冊建角**來決定，原因就在這裡。

**★ 最值得注意的一項落差：ICU 被裁掉了表格式字集。**

上游為了體積把 ICU 的資料檔（原本約 30 MB）用 `icupkg` 裁到約 780 KB，
只留下斷詞規則（brkitr）與轉換器別名表。理由是 driver 只從資料檔裡拿斷詞資料——
字元屬性與 NFC 已編進 `libicuuc`，而 UTF-8／UTF-16／Latin-1／ASCII 的轉換是**演算法式**的，
不需要資料表。

**代價是：GBK、Big5、Shift-JIS 這些表格式字集消失了。**
`set_encoding()`／`string_encode()`／`buffer_transcode()` 一旦指向它們就會 raise LPC error。
（LPC 可以用 `__WASM__` 判斷並自行調整。）

對一本講**中文 MUD 協議**的書來說，這一條不是細節，是主線：
本專案的 mudlib 在 `master.c` 第 14 行寫著

```c
if (port == 5003) { set_encoding("GBK"); }
```

而 §039 已經說過它為什麼沒炸——`wasm_console_connect()`
把連線標成來自第一個 `external_port`，也就是 UTF-8 的 5001。
**兩個獨立的限制剛好對消了**，而這種「剛好沒事」必須被寫下來，
否則有一天有人調換兩個埠的順序，會得到一個非常難查的錯誤。

**其他已知落差**（上游明列，本專案逐條確認過）：

| 落差 | 具體症狀 |
|------|---------|
| **沒有 eval limit** | LPC 裡 `while(1);` 卡死整個分頁（§037） |
| 沒有真 DNS | `query_ip_number()` 一律回 `127.0.0.1`；`resolve()` 下一拍合成回傳 |
| 沒有 zlib | 壓縮的 `write_file`（flag 2）報錯；壓縮的 `save_object` 退化成純文字存檔；`.gz` 不會被自動解壓 |
| MEMFS 不落地 | 重整即失（持久化是上游的下一階段：IDBFS／OPFS 疊在 `/data` 上） |
| 分頁在背景時計時器被暫停 | 醒來時 gametick 補跑，**上限 100 拍** |
| `crypt()` | 屬 core，**不受 crypto 套件關閉影響**；只會看到 `old crypt() password detected` 警告 |

最後一列是本專案特地去確認的（`libs/lpmudname/NOTES.md` 的「沒有踩到的坑」）：
`crypt()` 是登入流程的關鍵（§044 會看到握手行就是它算出來的），
如果它跟著 crypto 套件一起消失，17 個 mud 一個都登不進去。
**它沒有消失，因為它住在 core 而不是 crypto**（`src/packages/core/core.spec:191`）。
這種「憑什麼沒事」的確認，和「這裡壞了」一樣值得記錄。

**優點 / 罩門**：優點是**體積**——`fluffos.wasm` 約 3.6 MB 原始大小，
過線約 0.8 MB（brotli）／1.1 MB（gzip），加約 110 KB 的 JS glue。
一台完整的 MUD driver 用不到一 MB 就送到瀏覽器裡。
罩門是**落差清單會隨版本改變**，而 mudlib 不會知道。
本專案的緩解方式是把落差變成建置產物：
每個 lib 的 `NOTES.md` 都由 `boot-test.mjs` 實測後寫回，
driver 升版就重跑一次——**落差表不是文件，是測試結果**。

**效益**：這張矩陣把「這個 mud 為什麼有些功能不見了」
從一個要現場除錯的問題，變成一個查表就能回答的問題。
17 個 lib 匯入時，絕大多數「這個 daemon 為什麼載不起來」
都可以在十秒內對應到矩陣上的某一格。

> 💡 君之一席話
> **移植的成品清單有兩份：能跑的東西，以及不見了的東西**——只交前者的人，會在半年後被人問「為什麼跨服聊天沒有反應」，而那時已經沒有人記得答案。

> 🔍 老手視角──真正的門道
> 這一節最該學的是**落差的記錄方式**。多數專案把「已知限制」寫成 README 裡的一段話，三個月後它就過期了——因為沒有東西會在它變錯的時候告訴你。本專案的做法是把它變成**建置產物**：`boot-test.mjs` 實跑一次註冊建角，把載入失敗的物件、收到的 opcode、最終分級寫回每個 lib 的 `NOTES.md`，並在檔案裡標明「由建置產生，勿手改」。這個轉換的價值不在自動化省了多少時間，在於**它讓落差清單擁有了一個過期機制**：driver 升版、mudlib 改壞，下一次建置就自己降級。老手看一份「已知限制」文件，第一個問的是「它上次被驗證是什麼時候」；答不出來的，就當它不存在。可落地的做法：任何長期文件裡的事實性斷言，想辦法讓它由一支會失敗的腳本產生——**能自己出錯的文件，才是活的**。

---

## 042　五個匯出、兩個回呼，以及三個只有這裡才有的陷阱

**標籤**：`#匯出介面` `#重入` `#時鐘` `#靜默失敗` `#閉環驗證`
**證據等級**：🟢 本專案實測（三個陷阱各有 before/after）＋ 🟡 上游 `src/wasm/README.md` §4

**起源**：接縫抽好了、套件選好了，剩下的就是**頁面怎麼開這台 driver**。
介面小得驚人：五個匯出函式、兩個回呼。

**技術核心**：上游文件給的完整形狀——

```js
const M = await createFluffOS({ print, printErr, locateFile });
M.FS.chdir('/testsuite');                       // mudlib 掛載點
M.fluffos = {
  onOutput:     (id, bytes) => {...},           // server → client 的線路位元組
  onDisconnect: (id)        => {...},
};
M.ccall('fluffos_boot',    'number', ['string'], ['etc/config.test']);
setInterval(() => M.ccall('fluffos_tick', 'number', ['number'],
                          [performance.now()]), 50);
const id = M.ccall('fluffos_connect', 'number', [], []);
M.ccall('fluffos_input', null, ['number','array','number'], [id, bytes, n]);
// 另外還有：fluffos_flag（對應 master::flag）、fluffos_disconnect、fluffos_shutdown
```

| 匯出／回呼 | 對應原生的什麼 |
|-----------|--------------|
| `fluffos_boot(config)` | `main()` 讀設定、載入 master、preload |
| `fluffos_tick(now_ms)` | libevent 的一圈迴圈 |
| `fluffos_connect()` | accept 一條連線（`user_add` → 協商 → `master::connect` → `logon`） |
| `fluffos_input(id, bytes, n)` | 一次 socket read |
| `fluffos_disconnect(id)` / `fluffos_shutdown()` | 關連線／關機 |
| `onOutput(id, bytes)` | 一次 socket write |
| `onDisconnect(id)` | 對端消失 |

`fluffos_flag` 對應 master 的 `flag()` apply（上游用它跑 LPC 測試套件）。

**還有 `jsbridge`**：只有 WASM 平台才有的套件，讓 LPC 反過來呼叫頁面——
`js_eval("navigator.userAgent")`、`js_call("fetch_json", ({url}), (: cb :))`，
以及 `js_export("inventory", (: ui_inventory :))` 讓頁面 UI 直接觸發 LPC。
**本專案沒有用它**：ZJMUD 客戶端的設計前提是「同一份前端同時服務桌面版、橋接版與 WASM 版」，
用了 `jsbridge` 就會多出一條只有 WASM 才有的路徑，
而本書第四篇整篇都在說為什麼不要那樣做。**記在這裡是為了說清楚那條沒走的路。**

**踩過的坑：三個陷阱，全部只有「伺服器在同一個分頁裡」才會出現。**

### 陷阱一：不能在 `onOutput` 裡呼叫 `fluffos_input`

`fluffos_input()` **會在自己回傳之前就產生輸出**，
而 `onOutput` 是 driver **在自己的輸出路徑中途**回呼頁面的。
此時再呼叫 `fluffos_input`，等於在 telnet 解析器還沒吐完位元組時重入 driver。

上游與官方參考前端都把這件事寫在註解裡：**“sends must be queued”**。
所以正解不是「延遲一點送」，是**一律排隊、只在 tick 的堆疊上送**。

本專案的實測 before/after：直接在回呼裡回覆 → 伺服器只回 2 行、卡在 `authing` 30 秒；
改成排隊 → 一路走到 `ESC000 0008`（要求建角）。

### 陷阱二：`fluffos_connect()` 在回傳之前就開始回呼

```js
connId = M.ccall('fluffos_connect', 'number', [], []);
//        └─ 這一行還沒賦值完，logon 的輸出已經同步回呼進 onOutput 了
```

如果 `send()` 用 `connId === null` 當作「還沒連上」的判準，
那麼**握手行觸發的第一次回覆會被靜默丟棄**——
連佇列都沒進去，`rawSend` 一次都沒被呼叫。

本專案在這裡連續誤判兩次（先怪時序、再加靜置窗），
真正找到它的方法只有一個：**在 `rawSend` 印一行，看它到底有沒有被呼叫**。
修法是把「已關閉」變成一個獨立的旗標，並且**在 `ccall` 之前**就設成 false：

```js
closed = false;                                   // ★ 必須在 ccall 之前
connId = M.ccall('fluffos_connect', 'number', [], []);
```

§032 已經把這個故事寫過一次；這裡補的是它的**通則**——
**同步回呼會在你以為的初始化完成之前抵達，
所以「初始化完成」不能用「賦值語句已執行」來表示。**

### 陷阱三：`fluffos_tick` 要的是「經過多久」，不是「現在幾點」

這一個 §032 沒寫，因為它是後來才浮出來的，而且**只有在真瀏覽器裡才會踩到**。

症狀：`91书剑` 一連上就收到「您花在连线进入手续的时间太久了」。
那是 `clone/user/login.c` 裡的 `call_out("time_check", 30)`——理應 30 秒後才跑。

| 版本 | 送進 `fluffos_tick` 的值 | 結果 |
|------|------------------------|------|
| 第一版 | `Date.now()`（epoch 毫秒） | driver 第一拍就認為過了 **1.7 兆毫秒** |
| 第二版 | `performance.now()` | 仍然錯，只是小一點——那是**分頁載入**起算的，而使用者先看清單、再等 29 MB 下載完才開機，開機時它已經是幾十萬毫秒 |
| 第三版 | `monotonicNow() - clockOrigin`，`clockOrigin` 壓在**開機那一刻** | ✅ |

**第二版最值得記**：它在 node 測試裡完全正常——
因為那邊開機就在行程起頭，`performance.now()` 還很小。
**這是一個只有在真實使用情境（先看清單、再下載幾十 MB）才會顯現的差異**，
而本專案的驗證迴路一開始沒有涵蓋它。

驗證的方式也很直接：把 `performance.now()` 加上 300000 的偏移量重跑，
**逐字重現了使用者截圖裡的畫面**（只收到握手行 + `ESC015` 逾時訊息）。
**能重現，才叫找到原因。**

**優點 / 罩門**：優點是介面小到可以完整背下來——
五個函式、兩個回呼，沒有狀態機、沒有握手、沒有版本協商。
罩門是這個介面**把原本由作業系統與網路提供的保證全部拿掉了**，
而那些保證原本是隱形的：

| 原本誰提供 | 提供了什麼保證 | WASM 版誰負責 |
|-----------|--------------|--------------|
| TCP／作業系統 | 送出與收到之間有 RTT，不會零延遲重入 | **頁面**（排隊 + 靜置窗） |
| 核心的 socket 緩衝 | 寫入不會同步觸發對端的處理 | **頁面**（只在 tick 裡送） |
| 系統時鐘／libevent | 單調、從行程起算的時間源 | **頁面**（自己記原點送差值） |
| 作業系統的行程隔離 | mudlib 跑瘋不會影響別人 | **沒有人**（eval limit 停用，§041） |

第一列解釋了為什麼本專案在橋接版從來沒踩到陷阱一與陷阱二：
**真實 TCP 的 RTT 天然蓋掉了那個空窗**。
把伺服器搬進分頁，等於把這層天然的緩衝拿掉——
**原本被延遲掩蓋的競態，會一次全部現形。**

**效益**：本專案最終把這三個陷阱與所有相容性修正，
一起固化成一條會自己跑完的驗證鏈：
`node webclient/tools/verify-fullstack.mjs`——真 DOM、真 wasm driver、真 HTTP，
選 mud → 開機 → 連線 → 登入 → 建角 → 進世界 → 換另一台 mud。
**三個陷阱都是靜默失敗，而對付靜默失敗的唯一工具就是一條會自己走完全程的路。**

> 💡 君之一席話
> **當你把兩個原本分開的東西塞進同一個行程，你拿掉的不只是網路——你拿掉的是「時間」這個天然的同步機制**；原本靠延遲僥倖成立的假設，會在那一刻全部到期。

> 🔍 老手視角──真正的門道
> 三個陷阱表面上是三種 bug（重入、狀態判斷、時鐘來源），但它們有同一個形狀：**一個原本由環境提供、從來沒有被寫下來的保證，被移植拿掉了**。這類問題最難的地方在於你沒有東西可以查——沒有人會在文件裡寫「本系統假設送出與收到之間有延遲」，因為在原本的環境裡那不是假設，是物理事實。老手處理移植時會刻意做一件事：**把「環境替我做了什麼」列出來**，一項一項問「新環境還做嗎」。時間、順序、隔離、原子性、失敗模式——這五樣是最常被無聲拿掉的。可落地的做法：移植的第一份文件不要寫「我要改什麼」，先寫「舊環境保證了什麼」；那份清單通常就是你未來三週會踩的坑的目錄，而且**寫它的成本遠低於一個一個踩**。
