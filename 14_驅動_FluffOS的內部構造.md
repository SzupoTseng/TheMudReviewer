## 036　三個方向的呼叫 — efun、apply 與 simul_efun

**標籤**：`#FluffOS` `#driver` `#efun` `#apply` `#simul_efun` `#邊界`
**證據等級**：🟡 FluffOS 官方文件（`docs/concepts/general/*`、`docs/apply/*`）

**起源**：讀 mudlib 原始碼時，最先卡住人的不是 LPC 語法，是**一個函式到底是誰在呼叫誰**。
`write()` 是誰？`create()` 是誰呼叫的？`is_chinese()` 又是哪來的？
這三個問題的答案分別是三種不同的機制，而它們的**方向不一樣**。

**技術核心**：FluffOS 的三層模型，以及層與層之間的三條呼叫路徑。

```
        ┌───────────────────────────────────────────┐
        │   mudlib（LPC 檔案：房間、NPC、登入流程）      │
        └───────────────────────────────────────────┘
              │  ①efun          ↑ ②apply       ↺ ③simul_efun
              │  往下呼叫         │ 往上回呼        （同層，繞道 simul_efun 物件）
              ▼                 │
        ┌───────────────────────────────────────────┐
        │   driver（C++：LPC 編譯器 + 位元組碼 VM      │
        │           + 網路 + 600 多個 efun）           │
        └───────────────────────────────────────────┘
```

| # | 機制 | 方向 | 誰定義 | 例子 | 找不到時 |
|---|------|------|--------|------|---------|
| ① | **efun** | mudlib → driver | driver（C++） | `write()`、`clone_object()`、`crypt()`、`set_encoding()` | 編譯錯誤 |
| ② | **apply** | driver → mudlib | mudlib（LPC） | `create()`、`init()`、`heart_beat()`、`logon()`、`connect()` | **靜默略過**（沒定義就是不做） |
| ③ | **simul_efun** | mudlib → mudlib | mudlib（單一物件） | `is_chinese()`、mudlib 自訂的一切工具函式 | 編譯錯誤 |

**② 是最容易誤解的一個**。apply 不是「你去呼叫 driver」，是「**driver 在特定時刻來呼叫你**」。
它沒有註冊步驟、沒有介面宣告——**你只要把函式名字寫對，driver 就會找到它**。
反過來說，名字打錯就是靜默失效：driver 找不到 `heart_beat`，
不會報錯，只會安靜地什麼都不做。

這個「靜默」的性質，和本書 §015 的七次事故是同一個家族的東西。
apply 的失效不會有例外、不會有 log、不會有紅字——
**它只是不發生**。這是讀 LPC mudlib 最需要建立的直覺。

**三類 apply，按呼叫時機分**：

| 類別 | 定義在哪個物件 | 典型成員 | 何時被呼叫 |
|------|--------------|---------|-----------|
| **object applies** | 每個一般物件 | `create` `init` `reset` `heart_beat` `clean_up` `id` `move_or_destruct` `on_destruct` `virtual_start` | 物件生命週期事件 |
| **master applies** | master 物件（本專案是 `/adm/single/master`） | `connect` `logon` `error_handler` `crash` `preload` `compile_object` `valid_read` `valid_write` `valid_shadow` `valid_override` `valid_seteuid` `get_root_uid` `include_file` `inherit_program` | 全域事件、安全決策、編譯決策 |
| **interactive applies** | 玩家連線所附著的物件 | `logon` `net_dead` `process_input` `write_prompt` `terminal_type` `window_size` `telnet_suboption` `catch_tell` `receive_message` `gmcp` `msdp` `mxp_tag` `zmp` | 該連線的網路事件 |

第三列請特別看一眼：**`gmcp` / `msdp` / `mxp_tag` / `zmp` 是 driver 提供的 apply**，
也就是說這四套 MUD 擴充協議 **driver 是知情的**——它會做 telnet 子協商、
把解出來的資料交給 mudlib。
**而 ZJMUD 不在這張表上。** 這件事的全部後果，§043 會展開。

**① efun 的規模**：FluffOS 官方 README 的說法是「600 多個內建函式」，
按主題分成套件（package）：`core`、`math`、`matrix`、`parser`、`sockets`、`db`、
`crypto`、`compress`、`pcre`、`async`、`external`、`ffi`、`uids`、`mudlib_stats`……
套件是**編譯期開關**——關掉的套件，它的 efun 就**不存在**，
mudlib 用到就是編譯錯誤。§041 會看到這件事在 WebAssembly 版上的實際形狀。

**③ simul_efun 的門道**：mudlib 可以定義一個物件（本專案是 `/adm/single/simul_efun`），
裡面的 **public 函式會被當成「假 efun」全域可見**。
編譯器遇到一個既不是本物件函式、也不是內建 efun 的裸呼叫時，
會把它編成對 simul_efun 物件的**專用指令**（不是一般的 `call_other`），
而且因為原型在編譯期已知，**回傳值不必轉型**，即使開了 `#pragma strict_types`。

它最有意思的用法是**覆蓋 efun**：定義一個和 efun 同名的 simul_efun，
全 mudlib 的呼叫就都會走你的版本；你在裡面做完檢查，再用 `efun::move_object()`
呼叫真正的 driver 實作。哪些物件能用 `efun::` 前綴繞過，
由 master 的 `valid_override()` 決定。

**解決的痛點**：這三條路徑合起來，回答了「driver 與 mudlib 的邊界到底畫在哪」——
**不是畫在檔案上，是畫在呼叫方向上**。
同一個名字（比如 `move_object`）可以同時是 efun 與 simul_efun，
真正決定行為的是誰在什麼方向上呼叫它。

**踩過的坑**：本專案 §032 的 `is_killing(ob)` 就是這個機制的受害者。
它宣告成收 `string`，實際被傳了 object。
如果它是 efun，driver 的參數檢查會擋下來；
但它是 **simul_efun**，型別檢查發生在 LPC 編譯期，
而舊 MudOS 的檢查比較寬鬆——於是這個錯誤在原伺服器上跑了很多年沒事，
換到 FluffOS 才爆。**「它在原機上跑得好好的」不構成它是對的證據。**

**優點 / 罩門**：優點是 mudlib 可以在不碰 C 的情況下改變幾乎任何行為，
連 efun 都能換掉。罩門是**除錯路徑會變長**：
看到 `write(...)` 你不能假設它是 driver 的 `write`——
得先去 simul_efun 物件裡確認有沒有人蓋掉它。

**效益**：對逆向工作的直接效益是**知道去哪裡找**。
本專案要回答「這個 ESC 開頭的字串是誰送出去的」時，路徑是固定的：
先在 mudlib 裡 grep 巨集名，找到 `write()` 呼叫點，
再確認 `write` 有沒有被 simul_efun 蓋掉——三步，不必碰 driver 一行 C++。

> 💡 君之一席話
> **搞清楚一個系統，先搞清楚它有幾種呼叫方向**——往下呼叫的是能力，往上回呼的是契約，同層繞道的是慣例；三者混在一起看，永遠看不出邊界在哪。

> 🔍 老手視角──真正的門道
> apply 這個機制值得單獨想一想：它是**用命名慣例取代介面宣告**。好處是零樣板——你不必 implement 任何 interface、不必註冊任何 callback，寫個叫 `heart_beat` 的函式就會被呼叫。代價是打錯字沒有人告訴你。這個取捨在現代生態裡反覆出現：pytest 的 `test_*`、Go 的 `TestXxx`、React 的 `useXxx`、Rails 的 convention over configuration，全是同一招。它們共同的緩解手段也一樣——**linter**：既然編譯器不管，就讓另一個工具管。老手看到「靠命名生效」的系統，第一個動作是問「有沒有東西會告訴我名字打錯了」；沒有的話，就自己加一條檢查進 CI。可落地的做法：任何靠慣例綁定的專案，**寫一支腳本把所有慣例名字列出來與實際定義比對**，成本一個下午，能擋掉一整類「它就是不動，也不知道為什麼」的問題。

---

## 037　雙堆疊虛擬機 — LPC 是怎麼被執行的

**標籤**：`#虛擬機` `#位元組碼` `#堆疊機` `#引用計數` `#沙盒`
**證據等級**：🟡 FluffOS 官方文件 `docs/driver/stackmachine.md`（**這份文件本身就是史料**）

**起源**：§033 引的那句「Lennart Augustsson 說服我去實作一個虛擬堆疊機的編譯器」，
它的產物就是這一節要講的東西。而最有意思的是——
**FluffOS 倉庫裡那份描述它的文件，文風與內容都還停在 Lars 的年代**，
開頭甚至還是全大寫的：

> THIS IS A DOCUMENT THAT DESCRIBES HOW A VIRTUAL STACK MACHINE HAS BEEN DEFINED,
> TO EXECUTE COMPILED LPC CODE.

三十七年，執行模型沒有換過。

**技術核心**：兩個堆疊。

```
    值堆疊（value stack）                 控制堆疊（control stack）
    ┌─────────────────────┐              ┌──────────────────┐
    │  ...                │              │  返回位址          │
 fp→│  參數 0             │              │  前一個 frame ptr  │
    │  參數 1             │              │  參數個數          │
    │  ...                │              │  區域變數個數      │
    │  區域變數 0          │              │  extern_call 旗標  │
    │  區域變數 1          │              └──────────────────┘
    │  ...                │
    │  暫時值              │
    ▼                     │
```

- **值堆疊**放所有東西：求值中間結果、區域變數、**以及參數**——
  參數就是區域變數，只是編號在前面。存取靠 **frame pointer 加偏移量**，
  所以是 O(1) 的索引，不是查表。
- **控制堆疊**放返回位址、前一個 frame pointer、參數與區域變數的個數。
  把個數存在這裡，`F_RETURN` 才知道要釋放多少東西，
  指令本身不必攜帶這個資訊。

每個值堆疊元素的型別是 `struct svalue`——這個名字在 FluffOS 的 C++ 原始碼裡至今還在。

**幾個關鍵指令的形狀**（原文件的位元組佈局）：

| 指令 | 格式 | 語意 |
|------|------|------|
| `F_FUNCTION` | `b0=F_FUNCTION, b1b2=函式編號, b3=引數個數` | 呼叫本物件函式。**同時初始化 frame pointer**，並把引數個數「調整」成被呼叫者要的數量——不足補 0、過多彈掉 |
| `F_RETURN` | `b0` | 釋放整個 frame，只留堆疊頂端一個值。若 `extern_call` 旗標為真就從求值器返回 |
| `F_CALL_OTHER` | `b1=F_CALL_OTHER, b2=引數個數` | 呼叫**其他物件**的函式（`ob->fn()`） |
| `F_AGGREGATE` | `b1, b2b3=陣列大小（最大 0xffff）` | 從堆疊頂端取 N 個元素組成陣列 |
| `F_CATCH` | `F_CATCH, b1b2=catch 之後的位址` | 執行時做 `setjmp()` 並**遞迴呼叫 `eval_instruction()`**；`F_THROW` 做 `longjmp()` |
| `F_SSCANF` | 引數個數為單一位元組 | 特例：前兩個引數傳值，其餘用 **`T_LVALUE`** 型別傳參考 |

三個細節值得停一下：

1. **`F_FUNCTION` 會自動調整引數個數**。LPC 呼叫函式時給多給少都不會錯——
   少的補 0、多的丟掉。這解釋了為什麼老 mudlib 裡到處是引數個數對不上的呼叫
   卻能跑；也解釋了為什麼**它們錯得很安靜**。
2. **`F_CATCH` 是靠 `setjmp`/`longjmp` 加遞迴求值器實作的**。
   所以 LPC 的 `catch()` 不只是語法糖，它會真的開一個新的求值 frame。
   本專案的 `master.c` 就用它包住登入物件的建立：
   `err = catch(login_ob = new(LOGIN_OB));`——**登入物件炸了，連線還在，
   還能寫一行「現在有人正在修改使用者連線部份的程式」給玩家**。
3. **`sscanf` 需要一個專門的 lvalue 型別**，因為 LPC 沒有指標。
   這是「語言少了一個特性，就得在 VM 裡補一個型別」的教科書級例子。

**記憶體模型**：沒有 `malloc`、沒有 `free`、沒有 `delete`。
`allocate(n)` 的單位是**元素**不是位元組。回收靠**引用計數**：
計數歸零就立刻回收。所以 LPC 沒有 GC 停頓，
但**有循環引用洩漏**——FluffOS 為此專門有一份 `docs/concepts/general/reference_loops.md`。

**沙盒**：LPC 是給不受信任的作者寫的（想想 1989 年的原始情境：讓玩家來蓋世界），
所以 driver 必須能中止失控的程式碼。手段是 **evaluation cost**：
每執行一條指令扣一點額度，額度用完就中斷該次執行並拋 LPC 錯誤。
`config` 裡的預設值是：

| 限制 | 預設值 | 擋的是什麼 |
|------|--------|-----------|
| `maximum evaluation cost` | 30,000,000 | 無窮迴圈 |
| `maximum call depth` | 150 | 無窮遞迴 |
| `evaluator stack size` | 65,536 | 堆疊爆掉 |
| `inherit chain size` | 30 | 繼承鏈失控 |
| `maximum array size` | 15,000 | 記憶體炸彈 |
| `maximum mapping size` | 150,000 | 同上 |
| `maximum string length` | 1,048,576 位元組 | 同上 |

**踩過的坑**：這套沙盒**在 WebAssembly 版上是關掉的**。
原生版的 eval limit 實作依賴 POSIX 的 `SIGVTALRM` 計時器，
WASM 沒有——官方文件把它列為已知限制的第一條：
「`while(1);` 在 LPC 裡會卡死整個分頁」。
（macOS 與 Windows 上其實也沒有。）
本專案的 `boot-test.mjs` 因此**必須自己帶逾時**：
不能假設 driver 會替你中止跑瘋的 mudlib。

**優點 / 罩門**：優點是這套設計極度耐久——三十七年、四個分支、
從 32 位元 Unix 到 64 位元 WebAssembly，執行模型一行沒改。
堆疊機的簡單性是它活下來的原因：**它沒有什麼可以過時的東西**。
罩門是效能天花板：純解譯的堆疊機沒有 JIT、沒有暫存器分配，
所有效能都靠 efun（也就是把熱點推回 C++）。
這正是為什麼一個 MUD driver 會有 600 多個內建函式——
**efun 數量是解譯器慢的直接後果**。

**效益**：對逆向工作而言，知道執行模型能回答一類具體問題：
「這段 LPC 執行時，`this_player()` 是誰？」——答案在控制堆疊；
「為什麼這個引數是 0？」——因為 `F_FUNCTION` 補的；
「為什麼這個錯誤沒有讓伺服器死掉？」——因為某個上層有 `catch()`。

> 💡 君之一席話
> **活得最久的設計通常不是最先進的那個，是最沒有東西可以過時的那個**——一台雙堆疊機沒有依賴任何一代硬體的特性，所以它可以從 1989 年的 Unix 搬到 2026 年的瀏覽器分頁裡，一行都不用改。

> 🔍 老手視角──真正的門道
> `F_FUNCTION` 自動補齊引數這件事，是一個非常值得警惕的設計。它讓 mudlib 極度好寫（加一個參數不會弄壞既有呼叫點），但它把**一整類錯誤變成了資料錯誤**：你不會拿到「引數個數不符」，你會拿到一個 0，然後那個 0 一路流進業務邏輯，最後在離現場很遠的地方變成一個看不懂的行為。這種「寬容輸入」的設計在協議層也有完全一樣的形狀——本書 §004 那個沒有跳脫機制的分隔符、§009 那個「定義了 102 個 opcode 只用 16 個」的 header，都是同一種寬容。老手評估寬容設計時只問一個問題：**寬容的代價由誰承擔？**如果是寫的人省事、除錯的人加倍痛苦，而這兩者又不是同一批人（在 mudlib 的情境裡，往往差了二十年），那這筆交易就是虧的。可落地的做法：在寬容的邊界上加一條可關閉的嚴格模式——LPC 有 `#pragma strict_types`，用它；協議層則是「解析器有一個嚴格模式，CI 跑嚴格模式，正式環境跑寬容模式」。

---

## 038　物件的一生 — clone、apply 與心跳

**標籤**：`#物件模型` `#clone` `#heart_beat` `#call_out` `#熱重載` `#生命週期`
**證據等級**：🟡 FluffOS 官方文件（`docs/apply/object/*`、`docs/concepts/general/hot_reload.md`、`docs/driver/config.md`）

**起源**：LPC 的物件模型和主流 OOP 差一個關鍵概念，
不先講清楚，讀 mudlib 會一路困惑：**檔案本身就是一個物件**。

**技術核心**：**blueprint（藍本）與 clone（複製體）**。

```
/inherit/weapon/sword.c
   │
   ├─ load_object()  →  藍本物件  /inherit/weapon/sword
   │                     （一份，第一次被碰到時編譯並建立）
   │
   └─ clone_object() →  複製體    /inherit/weapon/sword#1234
                          #1235
                          #1236 ...（要幾把有幾把）
```

一個 `.c` 檔就是一個類別**兼**一個實例。
`clone_object()` 產生的複製體共用同一份編譯後的位元組碼（program），
但各自有一份變數。所以「一萬把劍」的記憶體成本是一萬份變數，不是一萬份程式。

**物件的生命週期 apply**，按時間順序：

| 時機 | apply | 說明 |
|------|-------|------|
| 編譯後、變數初始化 | `__INIT` | 由編譯器產生，初始化有初值的變數 |
| 建立時 | **`create()`** | 相當於建構子。**LPC 沒有 `main()`，有的是 `create()`** |
| 每次有 living 物件進入 | **`init()`** | 典型用途是 `add_action()` 註冊指令 |
| 週期性 | **`reset()`** | 重生怪物、補貨。預設 `time to reset : 900` 秒 |
| 週期性（需開啟） | **`heart_beat()`** | 預設 `heartbeat interval msec : 1000`。**要 `set_heart_beat(1)` 才會跑** |
| 長時間沒人碰 | `clean_up()` | 預設 `time to clean up : 600` 秒。物件可以自己決定要不要 `destruct` |
| 記憶體不足／閒置 | （swap） | `time to swap : 300`，0 為停用 |
| 被摧毀時 | `on_destruct()` | 收尾 |
| 環境被摧毀時 | `move_or_destruct()` | 決定跟著死還是搬家 |

**兩種週期機制不要搞混**：

| | `heart_beat()` | `call_out()` |
|---|---------------|-------------|
| 形式 | apply（driver 定時來呼叫你） | efun（你排一個未來的呼叫） |
| 頻率 | 固定間隔，預設 1 秒 | 一次性，指定幾秒後 |
| 開關 | `set_heart_beat(1/0)` | 排一次跑一次 |
| 典型用途 | 戰鬥回合、狀態衰減、飢餓值 | 延遲效果、逾時、排程 |
| 成本 | **每個開了心跳的物件每秒都在跑** | 只在到期時跑 |

第五列是效能上的重點：一個 MUD 卡不卡，
通常直接取決於**有多少物件開著 `heart_beat`**。

**時間的粒度**由 `gametick msec` 決定（預設 1000 毫秒）——
這是「遊戲內時間的最短可見間隔」。
§040 會看到，WebAssembly 版把這個時鐘的推進權從 libevent 交給了瀏覽器分頁。

**熱重載**：LPMud 的招牌能力，機制比想像中細緻。
`update` 一個檔案不能只重載它自己，因為：

- 每個 `#include` 的標頭是**編譯期併進去**的；
- 每個 `inherit` 的父程式是**編譯期綁進子程式**的——
  子程式連結的是它編譯當下的那一份父程式。

所以改了 `/std/base.c`，只重載自己檔案改過的物件，
**每一個繼承者都還在跑舊 code，而且完全沒有徵兆**。
FluffOS 給的解法是兩個編譯期 master apply：

```c
mixed inherit_program(string from, string path, int priv);
mixed include_file(string compiled, string from, string path);
```

driver 在**每一次編譯**時，對每個 `inherit` 呼叫前者、每個 `#include` 呼叫後者。
mudlib 只要**原樣回傳 `path`**（保持原行為）**並記下這條邊**，
就免費得到一張完整的相依圖，可以據此做出正確的自動熱重載。

這個設計的漂亮之處：**它沒有新增任何機制**——
兩個 apply 本來就是為了安全檢查（決定准不准 include／inherit）而存在的，
熱重載只是把它們的**引數**拿來用。

**踩過的坑**：`create()` 與 `reset()` 的關係在 MudOS 系與 LPMud 官方線上不一樣，
這是跨血脈移植 mudlib 最常撞的一堵牆。
本專案沒有踩到，因為 17 個 lib 全是 MudOS 系；
但 §035 的表提醒過：**知道血脈，才知道該預期哪一套語意**。

另一個真的踩到的是 `preload`：master 的 `preload()` apply 回傳一張開機時要載入的清單。
本專案 `libs/lpmudname` 裡有三個 daemon 因為 WASM 版沒有 `package sockets`
而編譯失敗（`Undefined function socket_create` 等 13 處），
**driver 照常完成開機**——preload 清單裡的項目載入失敗，
只會讓那個物件不存在，不會中止啟動。
這正是本專案 `playable` / `limited` 分級的分界線：**看的是「進不進得了世界」**。

**優點 / 罩門**：優點是「藍本 + 複製體」讓一份 code 服務上萬個實例，
而熱重載讓開發迴圈短到近乎即時。
罩門是**狀態與程式的版本會分岔**：熱重載換掉的是 program，
物件裡既有的變數還在——如果新版程式對變數的假設變了，
你會得到一個「跑著新程式、帶著舊資料」的物件。
這是 LPMud 家族最經典的一類幽靈 bug。

**效益**：對本專案的直接效益是**知道 boot-test 要等什麼**。
`boot-test.mjs` 判斷一個 mud 開得起來的依據不是「driver 沒有退出」，
而是走完「註冊 → 建角 → 進世界」——
因為 preload 失敗、reset 沒跑、心跳沒開，driver 都不會抱怨。
**在一個「失敗是靜默的」系統上，唯一可信的驗證是端到端跑一遍。**

> 💡 君之一席話
> **熱重載真正的難點從來不是「重新編譯」，是「誰依賴了我」**——沒有相依圖的熱重載，只是把一個明顯的錯誤換成一個安靜的錯誤。

> 🔍 老手視角──真正的門道
> `inherit_program` / `include_file` 這一手很值得抄：**FluffOS 沒有為了熱重載新增任何 API**，它把兩個既有的安全檢查點的引數拿來當相依圖的來源。這是「機制複用」的典範——同一個 hook 點，一個消費者用它的**回傳值**（安全決策），另一個消費者用它的**引數**（可觀測性）。老手在設計擴充點時會刻意留這個餘裕：hook 的簽名盡量帶齊上下文，即使當下的用途只需要一個布林值。因為擴充點的真正價值往往不在你設計它時想到的那個用途上。可落地的做法：任何 callback／middleware／攔截器，**簽名裡多帶「這件事發生在什麼情境下」的參數**（誰、為了誰、在編譯／處理哪個東西時），成本幾乎為零，而它會在兩年後救你一次。

---

## 039　config — driver 的全部旋鈕，以及本專案在上面栽的跟頭

**標籤**：`#設定檔` `#config` `#external_port` `#冪等` `#工具設計`
**證據等級**：🟢 本專案實測（17 個 mudlib 的匯入）＋ 🟡 FluffOS `docs/driver/config.md`

**起源**：driver 啟動時只吃一個東西：一個設定檔的路徑。
`driver.exe config.ini`——之後的一切（mudlib 在哪、master 是誰、開哪些埠）
全部來自那個檔案。所以**要把一份陌生的 mudlib 跑起來，
第一件事永遠是把它的設定檔弄對**。

本專案匯入 17 個 mudlib 的過程，一半的時間花在這個檔案上。

**技術核心**：本專案上游 `LPMud-Name/world/config.ini` 的關鍵欄位，逐條說明——
這份檔案剛好把最重要的幾類旋鈕都用到了：

```ini
name : 江湖论剑

external_port_1 : telnet 5001         # ★ UTF-8
external_port_2 : telnet 5003         # ★ GBK（由 mudlib 自己切換）
external_port_3 : websocket 5004

mudlib directory : ./
include directories : /include
master file : /adm/single/master
simulated efun file : /adm/single/simul_efun
global include file : <globals.h>
log directory : /log
debug log file : debug.log
websocket http dir : www

default fail message : [35m指令错误，请输入 help 查看游戏帮助。[2;37;0m
```

| 欄位 | 作用 | 為什麼重要 |
|------|------|-----------|
| `master file` | §036 那一整排 master apply 住在哪 | **弄錯就完全開不了機** |
| `simulated efun file` | §036 的 ③ 從哪裡來 | 弄錯的話 mudlib 到處是「未定義函式」 |
| `include directories` | `#include <...>` 的搜尋路徑 | 冒號分隔多個 |
| `global include file` | **每個檔案自動 include 這個** | 讀 mudlib 時常見的「這個巨集哪來的」，答案通常在這 |
| `external_port_N` | 開哪些埠、講哪種協議（`telnet`／`websocket`／`ascii`／`binary`，可加 `_tls`） | ★ 見下方 |
| `websocket http dir` | WebSocket 埠同時當靜態網頁伺服器 | driver 本身就能發網頁 |
| `default fail message` | 指令不存在時的訊息 | **值可以直接包含裸 ESC 字元**——本書所有 ANSI 方言的最外層來源之一 |

driver 的旋鈕遠不只這些。與本書關係最近的幾類，附官方預設值：

| 類別 | 代表旋鈕（預設值） |
|------|------------------|
| 時序 | `gametick msec`(1000)、`heartbeat interval msec`(1000)、`time to reset`(900)、`time to clean up`(600)、`time to swap`(300) |
| 限制 | `maximum evaluation cost`(30,000,000)、`maximum call depth`(150)、`evaluator stack size`(65,536)、`maximum array size`(15,000)、`maximum string length`(1 MiB) |
| 雜湊表 | `object table size`(4096，建議是物件數的 1/4)、`hash table size`(65,536，建議是相異字串數的 1/5，且應為質數)、`living hash table size`(256，**只能是 4/16/64/256/1024/4096**) |
| 語意開關 | `sane explode string`(1)、`sane sorting`(1)、`old range behavior`(0)、`this_player in call_out`(1) |
| reset 行為 | `no resets`(0)、`lazy resets`(0)、`randomized resets`(1) |

**「語意開關」那一列是相容性層的真面目**：
`sane explode string`、`old range behavior` 這種旗標的存在本身就是歷史證據——
**driver 知道自己以前的行為是錯的，但不敢直接改，只能給一個開關**。
§035 說的「相容性承諾會凍結錯誤」，在這裡是可以逐行讀出來的。

**踩過的坑（三個，全部是本專案實測）：**

**① `external_port_1` 決定 WebAssembly 版走哪個編碼分支。**
本專案的 mudlib 在 master 裡寫著：

```c
object connect(int port)
{
    if (port == 5003) {
        set_encoding("GBK");
    }
    ...
}
```

而 WASM 版的 ICU 資料被裁掉了表格式字集（§041），
真的走進 GBK 分支會直接 raise error。
它之所以沒炸，是因為 driver 的 `wasm_console_connect()`
把每條連線都標成「來自**第一個** external_port」
（`src/wasm/comm_wasm.cc:124-126`）——也就是 `telnet 5001`，UTF-8 分支。

> **本專案因此定下一條打包規則：第一個 `external_port` 必須是 UTF-8 的那一個。**
> 這條規則寫在 `libs/lpmudname/NOTES.md` 的「沒有踩到的坑」一節——
> **沒踩到的坑也要記**，否則下一個人把兩個埠的順序調換，會得到一個非常難查的錯誤。

**② 這個設定檔沒有行尾註解。**
官方文件寫得很清楚：*“Lines beginning with `#` are comments”*——
**只有整行註解**。值取的是冒號後面的整行。

本專案在自動修正腳本裡把說明寫成行尾註解：

```ini
external_port_1 : telnet 5001   # 改成 UTF-8 埠
                              └──────┬──────┘
                                 這整段都是值的一部分
```

三個原本已經修好的 lib 因此變回 `noboot`。

**③ 而且那支修正腳本不是冪等的。**
它每重跑一次就再疊一層註解，於是「修一次壞一點」。
這是本專案整批匯入過程中**最貴的一次錯誤**，教訓寫在 §032：

> **自動修正工具本身必須是冪等的，否則它會變成另一種損壞來源。**

第三個坑比前兩個嚴重得多，因為前兩個是**知識**問題（不知道規則），
第三個是**工程**問題（工具的性質不對）。知識可以查，
而一個非冪等的修正工具會在你查資料的時候繼續破壞現場。

**優點 / 罩門**：優點是所有行為集中在一個純文字檔，
沒有環境變數、沒有命令列旗標的組合爆炸，`diff` 兩份 config 就能比較兩台 mud。
罩門是**沒有 schema、沒有驗證**：打錯欄位名不會有人告訴你，
只會得到預設值或一個很遠的錯誤（本專案看過的極端例子是設定裡寫著
原營運者機器上的絕對路徑 `c:\hell`，症狀是 `Bad mudlib directory`）。

**效益**：本專案把「設定檔對不對」變成建置管線的一部分——
`boot-test.mjs` 真的用那份 config 開一次機、走一次註冊建角。
**沒有 schema，就用端到端測試當 schema。**

> 💡 君之一席話
> **修東西的工具，本身必須能安全地重跑**——不能重跑的修正工具，第二次執行時就從解決方案變成故障源。

> 🔍 老手視角──真正的門道
> 那條「第一個 external_port 必須是 UTF-8」的規則，形狀值得注意：它是一條**跨層的隱性契約**——`config.ini`（設定層）的**順序**，決定了 `master.c`（mudlib 層）走哪個分支，而中間的因果藏在 driver 的 C++ 原始碼裡（`comm_wasm.cc` 的三行）。這種契約最危險的地方在於它在任何一層看起來都不像契約：config 看起來只是列了兩個埠，master.c 看起來只是判斷了一個埠號。老手處理這類契約的方法只有一個——**把它寫在會被讀到的地方，並附上「憑什麼」**。本專案的做法是寫進該 lib 的 `NOTES.md` 並附上 `comm_wasm.cc:124-126` 的行號；更強的做法是讓打包腳本直接**檢查**這件事並在違反時拒絕打包。可落地的判準：一條規則如果只存在於某個人的腦袋裡，它的半衰期大約是三個月；寫進註解大約兩年；寫進會失敗的檢查，才真的活得比你久。
