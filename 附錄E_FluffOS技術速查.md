# 附錄 E　FluffOS / LPC 技術速查

> 第八、九篇把 driver 的構造分散在各節解釋。這份附錄把**可查表的部分**集中起來：
> efun 分類、apply 全表、config 全旋鈕、telnet 選項、套件矩陣、WASM 匯出 API。
>
> 證據等級：🟡 上游 `fluffos` 倉庫（`docs/`、`src/net/telnet.cc`、`src/CMakeLists.txt`）
> ＋ 🟢 本專案對 `v2026.0729.0` 的實測。
> **數量全部是撰寫當下數出來的，不是估計。**

---

## E.1　三個呼叫方向（§036 的速查版）

| 方向 | 機制 | 誰定義 | 找不到時 |
|------|------|--------|---------|
| mudlib → driver | **efun** | driver（C++） | 編譯錯誤 |
| driver → mudlib | **apply** | mudlib（LPC） | **靜默略過** |
| mudlib → mudlib | **simul_efun** | mudlib 的單一物件 | 編譯錯誤 |

**記住這一條就夠了：apply 打錯字不會有人告訴你。**

---

## E.2　efun 目錄（451 篇文件，26 類）

| 類別 | 篇數 | 代表 efun |
|------|-----:|-----------|
| `contrib` | 64 | `abs` `base_name` `copy` `debug_message` `assemble_class` `compressedp` |
| `interactive` | 54 | `add_action` `command` `enable_commands` `exec` `ed` `act_mxp` |
| `system` | 33 | `call_out_info` `ctime` `error` `eval_cost` `function_exists` `deep_inherit_list` |
| `objects` | 30 | `clone_object` `destruct` `environment` `file_name` `find_object` `deep_inventory` |
| `general` | 29 | `filter` `explode_reversible` `distance` `compress` `id_matrix` |
| `internals` | 28 | `debug_info` `cache_stats` `check_memory` `dump_jemalloc` `destructed_objects` |
| `strings` | 26 | **`crypt`** `explode` `implode` `sprintf` `capitalize` `hash` |
| `ffi` | 18 | `ffi_load` `ffi_call` `ffi_callback` `ffi_alloc` |
| `mudlib` | 16 | `getuid` `geteuid` `export_uid` `living` `find_living` `author_stats` |
| `floats` | 15 | `sin` `cos` `log` `exp` `floor` `ceil` `floatp` |
| `sockets` | 14 | `socket_create` `socket_bind` `socket_connect` `socket_accept` `socket_write` |
| `filesystem` | 13 | `read_file` `write_file` `get_dir` `mkdir` `rm` `read_bytes` |
| `calls` | 12 | **`call_other`** **`call_out`** `catch` `previous_object` `origin` `remove_call_out` |
| `parsing` | 12 | **`parse_command`** `parse_add_rule` `parse_init` `parse_sentence` |
| `arrays` | 10 | `allocate` `member_array` `sort_array` `filter_array` `map_array` `shuffle` |
| `mappings` | 9 | `allocate_mapping` `keys` `values` `map_delete` `filter_mapping` |
| `pcre` | 8 | `pcre_match` `pcre_replace` `pcre_extract` `pcre_match_all` |
| `buffers` | 7 | `allocate_buffer` `read_buffer` `write_buffer` `buffer_transcode` `crc32` |
| `db` | 7 | `db_connect` `db_exec` `db_fetch` `db_commit` `db_rollback` |
| `async` | 4 | `async_read` `async_write` `async_getdir` `async_db_exec` |
| `functions` | 4 | `bind` `defer` `evaluate` `functionp` |
| `numbers` | 4 | `random` `secure_random` `intp` `to_float` |
| `ed` | 3 | `ed_start` `ed_cmd` `query_ed_mode` |
| **`jsbridge`** | 3 | **`js_eval` `js_call` `js_export`**（**僅 WASM**，§042） |
| `crypto` | 1 | `hash` |
| `external` | 1 | `external_start` |

> ⚠️ **`crypt()` 在 `strings` 而不是 `crypto`。** 這件事救了本專案 17 份 mudlib——
> 關掉 crypto 套件不會讓登入流程消失（§041、§044）。
> **efun 住在哪個套件，比它叫什麼名字重要。**

---

## E.3　apply 全表（69 個）

### E.3.1　object applies（11 個）— 每個物件的生命週期

| apply | 何時被呼叫 |
|-------|-----------|
| `__INIT` | 編譯器產生，初始化有初值的變數 |
| **`create`** | 物件建立時（相當於建構子；**LPC 沒有 `main()`**） |
| **`init`** | 每次有 living 物件進入時（典型用途：`add_action`） |
| **`reset`** | 週期性（預設 900 秒）——重生怪物、補貨 |
| **`heart_beat`** | 週期性（預設 1000 ms），**要 `set_heart_beat(1)` 才會跑** |
| `clean_up` | 長時間沒人碰（預設 600 秒） |
| `id` | 「這個名字指的是我嗎」 |
| `is_living` | 是不是生物 |
| `move_or_destruct` | 環境被摧毀時：跟著死還是搬家 |
| `on_destruct` | 被摧毀時的收尾 |
| `virtual_start` | virtual object 的起始 |

### E.3.2　master applies（37 個）— 全域事件、安全與編譯決策

| 分組 | apply |
|------|-------|
| **連線** | **`connect`**（§044）、`epilog`、`preload`、`crash`、`flag` |
| **編譯** | `compile_object`（virtual object）、**`include_file`**、**`inherit_program`**（熱重載相依圖，§038）、`get_include_path`、`make_path_absolute` |
| **錯誤** | `error_handler`、`log_error`、`parser_error_message` |
| **身分** | `creator_file`、`author_file`、`domain_file`、`privs_file`、`get_root_uid`、`get_bb_uid`、`object_name`、`get_save_file_name` |
| **安全（`valid_*`，14 個）** | `valid_read` `valid_write` `valid_object` `valid_shadow` `valid_override` `valid_seteuid` `valid_socket` `valid_bind` `valid_link` `valid_hide` `valid_database` `valid_ffi` `valid_save_binary` |
| **其他** | `get_mud_stats`、`retrieve_ed_setup`、`save_ed_setup` |

> 🟢 §032 那個「建角時 `Denied write permission`」的坑就在 `valid_write`：
> 找不到 SECURITY_D 就一律拒絕（fail-closed），
> 而**同一個檔案裡的 `valid_read` 是 fail-open**——兩個相反的預設。

### E.3.3　interactive applies（21 個）— 一條連線的網路事件

| 分組 | apply |
|------|-------|
| **登入／斷線** | **`logon`**（§044）、`net_dead` |
| **輸入** | `process_input`、`write_prompt`、`receive_ed` |
| **輸出** | `catch_tell`、`receive_message`、`receive_snoop`、`terminal_colour_replace` |
| **telnet 協商** | `telnet_suboption`、`terminal_type`、`window_size`、`receive_environ` |
| **擴充協議（§056）** | `gmcp` `gmcp_enable` `msdp` `msdp_enable` `msp_enable` `mxp_enable` `mxp_tag` `zmp` |

> **最後一列就是「driver 層協議」與「mudlib 層協議」的分界線**（§043）：
> GMCP／MSDP／MXP／ZMP 有 apply，driver 知情；
> **ZJMUD 沒有——driver 從頭到尾不知道它存在。**

---

## E.4　config 旋鈕（§039 的完整版）

### E.4.1　必填與核心

| 設定 | 說明 |
|------|------|
| `name` | **必填**。這台 mud 的名字 |
| `mudlib directory` | **必填**。mudlib 根目錄（**唯一不相對於 mudlib 的路徑**） |
| `log directory` | **必填**。debug.log 與統計檔的位置 |
| `include directories` | **必填**。`#include <...>` 的搜尋路徑，冒號分隔 |
| `master file` | **必填**。master 物件（弄錯就完全開不了機） |
| `simulated efun file` | simul_efun 物件 |
| `global include file` | **每個檔案自動 include** ——「這個巨集哪來的」通常在這 |
| `debug log file` | log 檔名 |
| `mud ip` | 多 IP 主機上要綁哪一個 |
| `websocket http dir` | WebSocket 埠兼靜態網頁根目錄 |

### E.4.2　時序與生命週期

| 設定 | 預設 |
|------|-----:|
| `gametick msec` | 1000 |
| `heartbeat interval msec` | 1000 |
| `time to reset` | 900 |
| `time to clean up` | 600 |
| `time to swap` | 300（0 = 停用） |

### E.4.3　限制（沙盒，§037）

| 設定 | 預設 |
|------|-----:|
| `maximum evaluation cost` | 30,000,000 |
| `maximum call depth` | 150 |
| `evaluator stack size` | 65,536 |
| `inherit chain size` | 30 |
| `maximum local variables` | 64（範圍 64–255） |
| `maximum array size` | 15,000 |
| `maximum mapping size` | 150,000 |
| `maximum string length` | 1,048,576 位元組 |
| `maximum buffer size` | 1,048,576 |
| `maximum bits in a bitfield` | 12,000 |
| `maximum byte transfer` | 262,144 |
| `maximum read file size` | 262,144 |

### E.4.4　雜湊表（調校用）

| 設定 | 預設 | 建議值 |
|------|-----:|--------|
| `hash table size` | 65,536 | 相異字串數的 **1/5**，且應為質數 |
| `object table size` | 4,096 | 物件數的 **1/4**（最小 1024） |
| `living hash table size` | 256 | **只能是 4/16/64/256/1024/4096** |

### E.4.5　語意開關（相容性層的真面目，§039）

| 設定 | 預設 | 意義 |
|------|-----:|------|
| `sane explode string` | 1 | `explode()` 最多剝一個前導分隔符 |
| `reversible explode string` | 0 | 讓 `implode(explode(x,y),y) == x`；覆蓋上一項 |
| `sane sorting` | 1 | 排序有明確且穩定的順序 |
| `old range behavior` | 0 | 負索引從尾端算（**舊行為**） |
| `warn old range behavior` | 1 | 用到舊行為就警告 |
| `this_player in call_out` | 1 | `call_out` 回呼裡 `this_player()` 可用 |
| `enable_commands call init` | 1 | `enable_commands()` 時呼叫 `init()` |
| `reverse defer` | 0 | `defer()` 反序執行 |
| `sprintf add_justified ignore ANSI colors` | 1 | 對齊時忽略 ANSI 色碼的寬度 |
| `call_out(0) nest level` | 1000 | `call_out(0)` 連鎖的巢狀上限 |
| `call other type check` / `call other warn` / `old type behavior` | 0 / 0 / 0 | `->` 的型別檢查強度 |

> **`old range behavior` 與 `old type behavior` 這種旗標的存在本身就是史料**——
> driver 知道自己以前的行為是錯的，但不敢直接改，只能給一個開關（§035）。

### E.4.6　reset 行為／診斷／玩家 I/O

| 設定 | 預設 |
|------|-----:|
| `no resets` / `lazy resets` / `randomized resets` | 0 / 0 / **1** |
| `no ansi`（把輸入裡的 ESC 換成空白） | **1** |
| `strip before process input` | 1 |
| `interactive catch tell` | 0 |
| `receive snoop` / `snoop shadowed` | 1 / 0 |
| `trace` / `trace code` | 1 / 0 |
| `display preload progress` | 1 |
| `mudlib error handler` / `trap crashes` | 1 / 1 |

### E.4.7　★ 協議開關 — 直接決定線路上的協商位元組

| 設定 | 預設 | 對應 telnet 選項 |
|------|-----:|----------------|
| `enable mssp` | **1** | 70 |
| `enable msp` | **1** | 90 |
| `enable mxp` | 0 | 91 |
| `enable gmcp` | 0 | 201 |
| `enable zmp` | 0 | 93 |
| `enable msdp` | 0 | 69 |

> 🟢 **本專案實測驗證了這張表**：WASM driver 的首包裡出現了
> `IAC WILL 70`（MSSP）與 `IAC WILL 90`（MSP）——**兩個預設為 1 的**；
> 而 MXP／GMCP／ZMP／MSDP **一個都沒出現**——**四個預設為 0 的**（§055）。
> **設定檔的預設值，在連線的第 10 個位元組上就看得見。**

### E.4.8　埠與安全

```ini
external_port_1 : telnet 5001          # 型別：telnet / websocket / ascii / binary
external_port_2 : telnet 5003
external_port_3 : websocket 5004
# external_port_2_tls : cert=cert.crt key=cert.key
```

| 安全設定 | 說明 |
|---------|------|
| `ffi allowed libraries` | `ffi_load()` 可開啟的 .so 白名單（空 = 全交給 `valid_ffi` apply） |
| `allowed os environment variables` | `get_os_env()` 可讀的環境變數白名單（**空 = 全部拒絕**） |
| `writable os environment variables` | `set_os_env()` 可寫的白名單（**空 = 全部拒絕**） |

> ⚠️ **格式陷阱（§039，本專案最貴的一次錯誤）**：
> **只有整行 `#` 註解，沒有行尾註解**——值取的是冒號後**整行**。

---

## E.5　telnet 選項速查（§055）

| 編號 | 名稱 | 用途 | FluffOS 立場 | 🟢 出現在實測首包？ |
|-----:|------|------|-------------|------------------|
| 0 | BINARY | 8-bit 透傳 | WILL / DO | — |
| 1 | ECHO | **回顯控制（密碼遮蔽）** | WILL / DO | — |
| 3 | SGA | Suppress Go Ahead——**字元模式訊號** | WILL / DO | — |
| 6 | TM | Timing Mark | WILL / DO | — |
| **24** | **TTYPE** | 終端機型別 | WONT / **DO** | ✅ `ff fd 18` |
| **31** | **NAWS** | 視窗大小（改變時重報） | WILL / DO | ✅ `ff fd 1f` |
| 34 | LINEMODE | 行模式 | WONT / DO | — |
| **39** | **NEW-ENVIRON** | 環境變數 | WONT / **DO** | ✅ `ff fd 27` |
| **42** | **CHARSET** | 字集協商 | WILL / DO | ✅ `ff fb 2a` |
| 69 | MSDP | 頻外型別化 KV | WILL / DO | ❌（預設關） |
| **70** | **MSSP** | MUD 伺服器狀態 | WILL / DO | ✅ `ff fb 46` |
| 85 | COMPRESS | MCCP v1 | — | — |
| **86** | **COMPRESS2** | **MCCP2 壓縮** | WILL / DO | ⚠️ **只有原生版**（`ff fb 56`）——WASM 版 zlib 被移除（§041） |
| **90** | **MSP** | 音效 | WILL / DO | ✅ `ff fb 5a` |
| 91 | MXP | 行內標記 | WILL / DO | ❌（預設關） |
| 93 | ZMP | Zenith MUD Protocol | WILL / DO | ❌（預設關） |
| 200 | ATCP | Achaea Telnet Client Protocol | — | — |
| 201 | GMCP | 頻外 JSON | WILL / DO | ❌（預設關） |

**協商框架**：

```
IAC WILL <opt>   我想啟用（我這邊做）      IAC WONT <opt>   我不做
IAC DO   <opt>   請你啟用（你那邊做）      IAC DONT <opt>   請你別做
IAC SB <opt> …資料… IAC SE                子協商資料通道
IAC IAC                                   跳脫的 0xFF
```

**四個性質**：對稱、雙向獨立（`WILL` 與 `DO` 各談各的）、
預設關閉、**拒絕是合法且無副作用的**——最後一條就是本專案「一律拒絕」策略成立的理由。

---

## E.6　WebAssembly 平台（§040–§042）

### E.6.1　匯出 API

| 匯出 | 簽名 | 對應原生的什麼 |
|------|------|--------------|
| `fluffos_boot` | `(string config) → int` | `main()` 讀設定、載入 master、preload |
| `fluffos_tick` | `(number now_ms) → int` | libevent 的一圈迴圈 |
| `fluffos_connect` | `() → int`（連線 id，負值 = 失敗） | accept 一條連線 |
| `fluffos_input` | `(int id, array bytes, int n)` | 一次 socket read |
| `fluffos_disconnect` | `(int id)` | 關一條連線 |
| `fluffos_shutdown` | `()` | 關機 |
| `fluffos_flag` | `(string)` | master 的 `flag()` apply |
| **回呼** `Module.fluffos.onOutput` | `(id, bytes)` | 一次 socket write |
| **回呼** `Module.fluffos.onDisconnect` | `(id)` | 對端消失 |

### E.6.2　三條必守規則（🟢 本專案各踩過一次）

| # | 規則 | 違反的症狀 |
|---|------|-----------|
| 1 | **絕不在 `onOutput` 裡呼叫 `fluffos_input`**——一律排隊、只在 tick 的堆疊上送 | 伺服器只回 2 行就卡住 |
| 2 | **不能用 `connId === null` 當「尚未連線」**——`fluffos_connect()` 會在回傳前就同步回呼 | 第一次回覆被靜默丟棄（`rawSend` 呼叫次數 0） |
| 3 | **`fluffos_tick` 要餵「經過多久」不是「現在幾點」**——開機當下記原點、之後送差值 | `call_out(…, 30)` 第一拍就到期（「您花在连线进入手续的时间太久了」） |

### E.6.3　套件矩陣

| 開著 | 關掉 | 僅 WASM |
|------|------|--------|
| core、ops、math、matrix、trim、uids、sha1、parser、contrib、develop、mudlib_stats、**pcre** | **sockets**、compress、external、async、**db**、crypto、ffi | **jsbridge** |

**第三方函式庫**：移除 libevent／libwebsockets／OpenSSL／zlib／jemalloc／backward-cpp；
**保留** libtelnet（頁面講真 telnet）、ICU、libpcre、musl crypt。

### E.6.4　已知落差

| 落差 | 症狀 |
|------|------|
| **沒有 eval limit** | LPC 的 `while(1);` **卡死整個分頁**（POSIX 計時器只支援 Linux） |
| **ICU 只剩演算法式字集** | 資料檔由 ~30 MB 裁到 ~780 KB（只留 brkitr）→ **GBK／Big5／Shift-JIS 消失**，`set_encoding("GBK")` 會 raise error |
| 沒有真 DNS | `query_ip_number()` 一律 `127.0.0.1` |
| 沒有 zlib | 壓縮 `write_file`(flag 2) 報錯；壓縮 `save_object` 退化為純文字 |
| MEMFS 不落地 | 重整即失 |
| 背景分頁 | 計時器被暫停，醒來補跑 gametick，**上限 100 拍** |

**體積**：`fluffos.wasm` ~3.6 MB 原始 / **~0.8 MB brotli**（~1.1 MB gzip）+ ~110 KB JS glue。

---

## E.7　VM 指令格式（§037）

| 指令 | 格式 | 語意 |
|------|------|------|
| `F_FUNCTION` | `b0, b1b2=函式編號, b3=引數個數` | 呼叫本物件函式；初始化 frame pointer；**自動調整引數個數**（不足補 0、過多彈掉） |
| `F_RETURN` | `b0` | 釋放整個 frame，只留堆疊頂端一值 |
| `F_CALL_OTHER` | `b1, b2=引數個數` | 呼叫其他物件（`ob->fn()`） |
| `F_AGGREGATE` | `b1, b2b3=陣列大小`（最大 0xffff） | 從堆疊頂端取 N 個組成陣列 |
| `F_CATCH` | `F_CATCH, b1b2=之後的位址` | `setjmp()` + **遞迴呼叫求值器**；`F_THROW` 做 `longjmp()` |
| `F_SSCANF` | 引數個數為單一位元組 | 前兩引數傳值，其餘用 **`T_LVALUE`** 傳參考 |

**兩個堆疊**：值堆疊（`struct svalue`，參數即區域變數，靠 frame pointer 索引）
與控制堆疊（返回位址、前一個 fp、參數／區域變數個數、`extern_call` 旗標）。

**記憶體**：沒有 `malloc`／`free`；`allocate(n)` 的單位是**元素**；回收靠**引用計數**
（沒有 GC 停頓，但**有循環引用洩漏**）。

---

## E.8　LPC 與 C 的差異速查

| 面向 | C | LPC |
|------|---|-----|
| 進入點 | `main()` | **`create()`**（apply） |
| 記憶體 | `malloc`/`free` | **無**；`allocate(元素數)`；引用計數回收 |
| 字串 | `char*` + `strcpy`/`strcat` | **內建型別**，`+` 直接串接（近 BASIC） |
| 結構 | `struct` / `union` | **`class`**（無 union）；或用 mapping |
| `->` | 結構成員 | **兩種用途**：`call_other()` **與** class 成員 |
| `sscanf("%s %s")` | 兩個「詞」 | **第一個詞 + 剩下全部**（⚠️ 最常見的移植地雷） |
| 指標 | 有 | 無（`ref`/`&` 傳參考） |
| 編譯 | 機器碼 | **位元組碼 + 解譯**，可熱重載 |
| 額外型別 | — | `object` `mapping` `mixed` `function` `buffer` |
| 現代語法糖 | — | mapping 的 `m.key`、可選鏈 `m?.key?.deep`（讀取用） |

---

## E.9　本專案用到的版本

| 項目 | 值 |
|------|---|
| FluffOS driver | **`v2026.0729.0`**（官方 wasm release，2.4 MB zip） |
| 映像格式 | 自訂（`webclient/src/js/mudlibimage.js`，`format: 1`） |
| 收錄 mudlib | **17 份**（另有 1 份匯入中） |
| 客戶端測試 | **202 條，12 個檔案，全過** |

> 取得 driver：`node webclient/tools/fetch-driver.mjs`
> 打包全部 mud：`node webclient/tools/build-site.mjs`
> 全鏈路驗證：`node webclient/tools/verify-fullstack.mjs`
