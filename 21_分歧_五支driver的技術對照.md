## 063　CD gamedriver — 最接近原版的那一支

**標籤**：`#CD` `#Genesis` `#hook` `#alarm` `#auth` `#VBFC`
**證據等級**：🟡 上游 `cotillion/cd-gamedriver`（`doc/CREDITS`、`doc/hooks/`、`doc/efun/`、原始碼樹）＋ 🟡 公開史料

**起源**：§035 那張四條血脈的表裡，CD 只有一句話帶過：
「Genesis 那邊改叫 CD」。

這句話漏掉了一件重要的事——**CD 不是 MudOS 的旁支，它是另一條主幹**，
而且**它是唯一還在跑原始那個世界的 driver**：Genesis，1989 年那個 mud，至今仍在營運。

**名字的來歷本身就是證據**：CD＝**Chalmers Datorförening**，
Lars Pensjö 所屬的查爾摩斯理工學院電腦社。
翻回 §033 引的那份 `Credits.LPmud` 第一行：

```
Lars Pensjö, April 1989 (lars@cd.chalmers.se)
                          ↑↑
```

**`cd` 從第一天就在那裡**——它不是後來取的名字，是這個專案的出生地。

1991 年底 Lars 從 Genesis 退休，其餘管理者從他的開發版（LPMud 3.0）分出了 CD。
所以在時間上，**CD 的分家比 MudOS（1992-08）還早**。

**技術核心**：CD 在四個地方做了與 MudOS 系**不同的選擇**，而每一個都很有意思。

### ① hook 而不是 apply

MudOS／FluffOS 叫 apply（§036）；CD 叫 **hook**，而且機制略有不同：

> The callback functions are **named in `master.n`** and are converted into
> **`M_*` named defines** that you should use within the driver.
> Use `apply_master()` to call the master object with the `M_*` defines.
> **If you do not have a function declared within the master object, 0 will be returned.**

差別在於**名字集中管理**：CD 把所有 hook 名字列在一個檔（`master.n`）裡，
編譯期生成 `M_*` 常數，driver 內部一律用常數而不是字串。

MudOS 系則是散在各處的字串 apply。
**這是「命名慣例」與「集中註冊」的差別**——
§036 老手視角提過的那個問題（靠命名生效的系統沒人告訴你名字打錯了），
CD 在 driver 這一側解掉了一半：**driver 不會拼錯，因為它用常數**。
mudlib 那一側仍然是慣例。

CD 的 hook 清單也更小、更聚焦於生命週期：

```
start_boot  preload_boot  final_boot
start_shutdown  cleanup_shutdown  final_shutdown
runtime_error  log_error  parse_exception
object_name  flag  get_vbfc_object
```

**開機與關機各分三階段**——比 MudOS 的 `preload` + `epilog` 細緻得多。

### ② alarm 而不是 call_out ＋ heart_beat

這是 CD 最漂亮的一個設計決定。MudOS 系有兩套週期機制（§038）：
`call_out()`（一次性）與 `heart_beat()`（固定間隔的 apply）。

**CD 把它們合成一個：**

```c
int set_alarm(float delay, float repeat, function func);
```

- `repeat` 為 0 → 一次性，等同 `call_out`；
- `repeat` 為正 → 重複執行，等同心跳，**而且間隔可以自訂**；
- 回傳 alarm id，可用 `remove_alarm` / `get_alarm` / `get_all_alarms` 管理。

> 例：`delay=3.0, repeat=4.0` → 3 秒後第一次，之後每 4 秒一次。

**兩個機制變成一個 API 的兩個參數。**
而且 delay 是 `float`——**次秒級精度**，MudOS 的 `call_out` 是整數秒。

文件裡還留著兩條很實在的警告：

> **NOTA BENE**：`set_alarm()` 呼叫的函式裡，`this_player()` 與
> `this_interactive()` **都沒有定義**——在那裡 `write()` 的輸出會跑進遊戲日誌。
>
> **WARNING**：**永遠不要在重複 alarm 裡再建一個重複 alarm。**
> 產生的 alarm 會呈指數成長，**不到一分鐘整台 mud 就會停擺。**

第一條與 FluffOS 的 `this_player in call_out` 設定（附錄 E.4.5，預設 1）
是同一個問題的兩種答案：**CD 選擇不定義，FluffOS 選擇提供一個開關。**

### ③ auth 而不是 uid / euid

§061 講的 uid/euid 是照抄 Unix 的。**CD 沒有照抄。**

```c
void set_auth(object ob, mixed info);
mixed query_auth(object ob);
```

> It is possible to store authority information **in any format** in the
> hidden authority variable of all objects. […] The gamedriver calls
> `valid_set_auth` in the master object, to determine if the call is legal
> **and possibly to modify the information**. The return value of
> `valid_set_auth` **is stored** in the hidden authority variable.

三個差異，每一個都是往「更泛用」的方向走：

| | MudOS 的 uid/euid | **CD 的 auth** |
|---|-----------------|---------------|
| 型別 | 字串（借 Unix 語意） | **`mixed`——任何格式** |
| 誰決定內容 | mudlib 呼叫 `seteuid()`，master 只能准或不准 | **master 的 `valid_set_auth` 回傳值就是最終存進去的東西**——它可以**改寫** |
| 語意 | 使用者／群組 | **由 mudlib 自己定義** |

**第二列是關鍵**：MudOS 的 `valid_seteuid()` 是一個布林閘門
（§061 說的那個「布林值不足以表達授權」的問題）；
**CD 的 `valid_set_auth` 是一個轉換函式**——它不只決定准不准，還決定**存什麼**。

**這個設計把授權策略徹底交給 mudlib**：
你想用 Unix 式的 uid，可以；想用能力清單（capability list），可以；
想存一個帶到期時間的 mapping，也可以。

### ④ VBFC — 把函式呼叫嵌在字串裡

這是 CD 最特別、也最危險的一個機制：

```c
mixed process_value(string calldescription, void|int security);
```

字串的格式是：

```
"function[:filename][|arg1|arg2|...|argN]"
```

也就是說，**一段字串可以描述一次函式呼叫**，`process_string()` 會把它替換掉。
mudlib 用它做「值由函式呼叫決定」（value by function call）——
房間描述裡可以嵌一段會動態求值的東西，不必寫程式。

文件自己標了警示：

> **CAVEAT**：It is wise to set the security parameter […] as **any function
> in any object can be called with almost any arguments**. You probably don't
> want to have this done with your privileges.

所以有了 `security` 參數與 `get_vbfc_object` hook——
**用一個沒有權限的物件當發起者**。

> 🔬 **本書的角度**：這正是 §004 那個「沒有跳脫機制的分隔符」問題的
> **最極端版本**——ZJMUD 把 `$zj#` 混進文字流最壞的後果是解析錯亂；
> **VBFC 把函式呼叫混進文字流，最壞的後果是任意程式碼執行。**
> 兩者是同一種設計（在人類可讀串流裡埋機器語意）在不同風險等級上的實例。

### ⑤ 其他值得記的分歧

| 面向 | CD | MudOS 系 |
|------|-----|---------|
| mapping API | `m_delete` `m_indexes` `m_values` `m_sizeof` `mkmapping` | `map_delete` `keys` `values` `sizeof` `allocate_mapping` |
| 存檔 | `save_object`／`m_save_object`／**`save_map`／`restore_map`** | `save_object` / `restore_object` |
| 呼叫形式 | `call_other` `call_otherv` **`call_self` `call_selfv`** `applyv` `papplyv` | `call_other`、`evaluate` |
| **JSON** | **`val2json` / `json2val`**（driver 依賴 `libjson-c`） | 無（mudlib 自己實作） |
| 配置器 | **bibopmalloc**（Lennart Augustsson 寫的 BiBOP） | smalloc → jemalloc |
| 雜湊 | **siphash** | 一般字串雜湊 |
| **intermud** | **在 driver 裡**：`udpsvc.c`、`tcpsvc.c`、`hname.c` | **在 mudlib 裡**：socket efun ＋ LPC（§049） |

**最後一列的對比非常值得停下來看。**

§049 講過 TMI-2 用 LPC 寫了整套 intermud（`_q`／`_a` 成對、`dns_master` 名錄），
那是因為 MudOS 給了 socket efun，**把能力推到上層**。

**CD 反過來：把 intermud 服務做進 driver。**
它的啟動旗標裡就有 `-u<port>`（UDP 埠）與 `-p<port>`（service 埠，供 ftp 等用途）——
**這些是 driver 級的設施，不是 mudlib 寫出來的。**

**同一個需求，兩個方向相反的答案**：
MudOS 說「我給你 socket，你自己寫」；CD 說「我幫你做好」。
§047 那句「引擎加的不是功能，是可能性的邊界」在這裡有了對照組——
**邊界推到哪裡，決定了上層要不要重新發明。**

**優點 / 罩門**：CD 的優點是**它一直有一個真實的使用者**——Genesis。
一個 driver 只服務一個世界，設計可以很聚焦、很少妥協，
所以它是「所有變體裡最接近原版 LPMud 精神」的那一支。
罩門也是同一件事：**生態極小**。
32 顆星、174 個 commit（撰寫當下），沒有 MudOS 系那種數千個 mudlib 的規模。
**本專案 17 份封存沒有一份是 CD 系的**，這不是巧合。

**效益**：對本書而言，CD 提供的是**對照組**。
第八、九篇所有關於「LPMud 家族長什麼樣」的敘述，
如果只看 MudOS→FluffOS 這一條，很容易把某些設計當成必然。
CD 證明了它們不是：
**apply 可以集中註冊、call_out 與 heart_beat 可以合併、
uid/euid 可以換成任意格式的 auth、intermud 可以做在 driver 裡。**

> 💡 君之一席話
> **看一個生態的設計是不是必然，最快的方法是找到那個做了相反選擇、而且活得好好的實作。**

> 🔍 老手視角──真正的門道
> CD 的 `set_alarm` 把 `call_out` 與 `heart_beat` 合成一個 API，這件事值得單獨想一下：它們在 MudOS 裡之所以是兩個東西，是因為**實作機制不同**（一個是排程佇列、一個是每 tick 掃過所有開了心跳的物件），而不是因為使用者的需求不同。使用者的需求其實只有一個——「過一段時間叫我」——差別只在要不要重複。**MudOS 把實作的差異洩漏成了 API 的差異**，於是每個 mudlib 作者都得學兩套、記兩套的陷阱、寫兩套管理程式碼。這是 API 設計裡最常見的一種洩漏，而它幾乎總是發生在「兩個功能是不同時期加的」的地方。老手在合併 API 時的判準是：**使用者在選擇時，問的是不是一個關於實作的問題？**——「我該用 call_out 還是 heart_beat」是關於實作的；「延遲多久、要不要重複」才是關於需求的。可落地的做法：任何有兩個相似 API 的地方，試著寫出「用一個 API 加參數」的版本，看看參數是不是自然——是的話，你原本那兩個 API 就是實作洩漏。

---

## 064　DGD — 從頭重寫的那一支，以及它證明了什麼

**標籤**：`#DGD` `#持久化` `#輕量物件` `#原子函式` `#JIT` `#狀態快照`
**證據等級**：🟡 上游官方站（dworkin.nl/dgd）與公開版本史

**起源**：§035 說 DGD 是「概念衍生，非程式碼衍生——重寫的」。
這一節解釋**重寫換到了什麼**——因為 DGD 的技術清單，
有一半是其他三支 driver 至今沒有的東西。

**技術核心**：Felix “Dworkin” Croes 從零重寫 LPMud，
而重寫的第一個決定就定調了後面全部：

> **DGD does away with all of LPMud's game-oriented features,
> which can be implemented in LPC.**

**把所有遊戲導向的功能從 driver 裡拿掉。**

對照 §047 那張 MudOS 能力面的表——
MudOS 的路線是「batteries-included」，把能力面推到最寬；
**DGD 走了完全相反的方向：核心極小，其餘用 LPC 自己蓋。**

這個選擇的結果是 DGD 通用到不像個 MUD driver：
**1990 年代末 Yahoo 的聊天室就是跑在 DGD 上的。**

### DGD 的五項技術創新

| 創新 | 是什麼 | 其他 driver 有嗎 |
|------|--------|----------------|
| **完整持久化** | **所有資料、程式與執行狀態都存在一個資料庫裡**；可做快照、可從快照恢復、重開機續跑 | ❌ MudOS 系只有 `save_object` 這種手動存檔 |
| **輕量物件** | 兩種物件：**輕量物件**（像一般 OOP 的物件，無 id、可大量產生）與**持久物件**（有全域唯一 id、能連網、能排程） | ❌（LDMud 後來借走了這個概念） |
| **原子函式（atomic function）** | **出錯就把這次呼叫的所有改動全部回捲**——保證一致性 | ❌ 這是資料庫的交易語意，出現在一個 MUD driver 上 |
| **就地重編＋hotboot** | 活著升級程式；**hotboot 換掉整個執行檔而不斷線** | ⚠️ MudOS 系有熱重載（§060），但沒有「換執行檔不斷線」 |
| **位元組碼 VM ＋選用 JIT** | 可編成機器碼 | ❌ 其餘三支都是純解譯（§037） |

**「原子函式」這一項特別值得停一下。**

§060 講過 MudOS 系最難查的一類 bug：程式與資料可以獨立地過期、
重載會讓變數歸零、世界上同時有兩個版本的同一種劍。
**這些問題的共同根源是「沒有交易邊界」**——
一次改動不是全成功就是全失敗，而是可能改到一半。

DGD 的原子函式直接把資料庫的交易語意搬進了語言層。
**這不是效能優化，是把一整類 bug 從可能變成不可能。**

### 與 LPMud 的相容關係

> DGD maintains backward compatibility with **LPMud 2.4.5** and mostly
> supports **LPMud 3.1.2**, though some features function differently.

注意它對標的是 **2.4.5 與 3.1.2**——也就是 **Lars 的官方線**，
而不是 MudOS。§034 說過 MudOS 明講自己與 3.0 mudlib 不相容；
**DGD 選擇了另一邊。**

### 版本史（技術轉折點）

| 版本 | 年份 | 轉折 |
|------|------|------|
| 1.0.a3 | 1993-08 | 第一個公開版 |
| 1.0.8 | 1994-08 | 第一個穩定版 |
| **1.4** | **2010-02** | **開源**（AGPL）——商業版權方無力維護後由 Dworkin B.V. 取得 |
| 1.5 | 2014-04 | 動態擴充模組、**hotboot**、位元組碼 VM v2 |
| 1.6 | 2017-04 | **C++ 重寫**；放棄 1.5.9 之前的快照相容性 |
| 1.7 | 2023-01 | 位元組碼 VM v2.4，強化 JIT 支援 |

**2010 年才開源**——這解釋了為什麼 DGD 的生態遠小於 MudOS 系：
在中文 MUD 大爆發的 1996–2005 年，**DGD 是閉源的**（§056 說的時間軸問題，
在這裡是另一個實例）。

另有一個封閉分支 **Hydra**：針對多核心優化、與 DGD 功能對等。

**解決的痛點**：DGD 解的是 LPMud 家族一個長期被忽略的問題——
**世界是易失的**。

MudOS 系的 mud 崩了，玩家會回到上一次 `save_object` 的狀態；
沒存的東西就沒了。這在 1990 年代被當成「就是這樣」。
**DGD 把它當成一個可以解決的問題，而且解法是「整個世界就是資料庫」。**

**踩過的坑（生態層面）**：DGD 的技術優勢從來沒有轉換成生態優勢，
原因有三，而且沒有一個是技術的：

1. **1993 年出現，2010 年才開源**——錯過整個 MUD 黃金期；
2. **不相容於 MudOS 的 mudlib**——當時絕大多數現成 mudlib 都是 MudOS 系的；
3. **極小核心意味著要自己蓋很多東西**——對「拿一份 mudlib 開站」的人來說門檻更高。

**這三條在本專案身上是可以驗證的**：17 份封存全部是 MudOS 系，
一份 DGD 系都沒有。

**優點 / 罩門**：優點是它在 1993 年就做對了很多要到 2010 年代
才被主流理解的事（持久化、交易語意、輕量物件、hotboot、JIT）。
罩門是**它證明了技術領先與生態成功是兩件事**——
而且證明得非常徹底。

**效益**：對本書而言，DGD 是第二個對照組，而且是更極端的那個：
CD 證明了「MudOS 的某些設計不是必然」，
**DGD 證明了「連 driver/mudlib 的分界線本身都可以重畫」**——
把遊戲導向的東西全部推上去，核心只留語言與持久化。

> 💡 君之一席話
> **技術領先十七年而生態沒有跟上，往往不是技術的問題，是時機與相容性的問題**——一個 2010 年才開源的 1993 年好東西，錯過的不是市場，是整整一代人選型的那一刻。

> 🔍 老手視角──真正的門道
> DGD 的「原子函式」值得從一個更高的角度看：**它把一致性從一個紀律問題變成了一個機制問題。** MudOS 系要保證「改到一半不會留下爛攤子」，靠的是作者自己小心（先檢查再改、失敗要手動回復）；DGD 直接給你一個保證。這個轉換在軟體史上反覆出現——記憶體管理從紀律（`free`）變成機制（GC）、資料一致性從紀律變成交易、資源釋放從紀律變成 RAII／`defer`、並行安全從紀律變成型別系統（Rust 的所有權）。它們的共同模式是：**先有一類反覆出現、靠紀律無法根治的 bug，然後有人把紀律編進機制裡。** 老手看到一個團隊反覆在同一類 bug 上跌倒時，第一個念頭不是「加強 code review」，而是「這個紀律能不能變成機制」——因為紀律的失效率不會歸零，而機制的失效率可以。可落地的做法：把團隊過去半年的事故分類，找出**同一種紀律失效超過三次**的那一類，然後問「有沒有一個機制可以讓它不可能發生」。

---

## 065　五支 driver 的技術對照 — 以及一份 CREDITS 的兩個分支

**標籤**：`#對照` `#設計空間` `#分歧` `#文獻化石` `#選型`
**證據等級**：🟡 四份上游倉庫與官方站的交叉比對 ＋ 🟢 本專案對 FluffOS 的實測

**起源**：§033–§035 排了時間軸，§063–§064 拆了 CD 與 DGD。
這一節把五支放在同一張表上——**不是為了排名，是為了看清楚設計空間有多大**。

**技術核心**：五支 driver 的分歧總表。

| 面向 | **LPMud**（1989） | **MudOS→FluffOS**（1992→今） | **LDMud**（1997） | **DGD**（1993） | **CD**（1991） |
|------|------------------|---------------------------|------------------|----------------|---------------|
| 血緣 | 本體 | 從 3.0.5x 分家，**明講不相容** | 接手 3.2（Amylaar 線） | **從零重寫**（概念衍生） | 從 Lars 的 3.0 開發版分家 |
| 路線 | — | **batteries-included**（能力面最寬） | 官方線的現代化延續 | **極小核心**（遊戲功能全推上層） | **服務單一世界**（Genesis） |
| 相容對標 | — | MudOS | LPMud 3.2 | **LPMud 2.4.5 / 3.1.2** | Lars 的 3.0 |
| driver→mudlib 回呼 | apply | **apply**（字串名，69 個） | apply | — | **hook**（`master.n` → `M_*` 常數） |
| 週期機制 | `call_out` + `heart_beat` | 同左（＋`gametick`） | 同左 | 任務模型 | **`set_alarm`（delay + repeat 合一，float 精度）** |
| 安全模型 | uid/euid | **uid/euid + 14 個 `valid_*`** | uid/euid | LPC 自己實作 | **`set_auth`（任意格式，master 可改寫）** |
| 持久化 | `save_object` | `save_object`（＋swap） | `save_object` | **★ 整個世界是資料庫，可快照可續跑** | `save_object` / `save_map` |
| 交易語意 | ❌ | ❌ | ❌ | **★ 原子函式（出錯全回捲）** | ❌ |
| 物件種類 | 一種 | 一種（＋virtual object） | **＋輕量物件**（借自 DGD） | **★ 輕量 + 持久兩種** | 一種 |
| 執行 | 解譯 | 解譯 | 解譯 | **★ 位元組碼 + 選用 JIT** | 解譯 |
| 熱更新 | `update` | 重編物件（§060） | 同左 | **★ 就地重編 + hotboot 換執行檔不斷線** | 重編物件 |
| intermud | — | **mudlib 用 socket efun 自己寫**（§049） | mudlib | LPC | **★ 做在 driver 裡**（`udpsvc.c`/`tcpsvc.c`） |
| 字串內嵌求值 | — | — | — | — | **★ VBFC（`process_string`/`process_value`）** |
| 現代化 | — | **C++、UTF-8/ICU、TLS、WebSocket、WASM、CI** | 持續維護 | C++ 重寫（1.6）、JIT | libjson-c、siphash、bibopmalloc |
| 開源時間 | 一開始就是 | 一開始就是 | 一開始就是 | **2010-02（AGPL）** | 一開始就是 |
| 生態規模 | — | **最大**（本專案 17 份封存全在此系） | 中 | 小（但用在非 MUD 場景） | **小**（主要服務 Genesis） |

**從這張表可以讀出三件事：**

### ① 五支解的其實是三個不同的問題

```
MudOS/FluffOS  →「怎麼讓最多人能開站」   答案：能力面推寬 + 死守相容
LDMud          →「怎麼把官方線接下去」   答案：穩定演進
CD             →「怎麼把一個世界養好」   答案：為 Genesis 量身做決定
DGD            →「LPMud 這個概念的正確實作是什麼」 答案：從零重寫，核心極小
```

**它們不是同一場競賽的參賽者。**
把它們放在一起比「誰比較好」沒有意義；
有意義的是**看它們在同一個設計問題上做了什麼不同的選擇**——
那才是設計空間。

### ② 每一支都有一項別人沒有的東西

| driver | 它獨有的那一項 |
|--------|--------------|
| FluffOS | **WebAssembly**——整台 driver 進瀏覽器分頁（第八篇整篇） |
| DGD | **原子函式 + 完整持久化**——一整類 bug 從可能變成不可能 |
| CD | **auth 可任意格式 + `valid_set_auth` 可改寫**——授權策略完全外包 |
| LDMud | 官方線的連續性——**唯一能宣稱「我是 LPMud 3.x」的那一支** |

**沒有一支是另一支的超集。**

### ③ ★ 一份 CREDITS 的兩個分支——文獻化石

這是本節最有意思的發現，而且它是純文獻證據。

FluffOS 倉庫封存的 `docs/archive/Credits.LPmud`（§033 引用過）
與 CD 倉庫的 `doc/CREDITS`，**開頭完全相同**：

```
The program was written originally by

Lars Pensjö, April 1989 (lars@cd.chalmers.se).

Other credits:

The regexp package was made by Henry Spencer.
The ed package was not done by me. See the file ed.c for information.
Bit manipulations was implemented by pell@lysator.liu.se …
Roland Dunkerley III fixed for-loops, do-while and the operators ++ and --.
Sean T Barrett made smalloc.c.
Lennart Augustsson convinced me to implement a compiler for a virtual
  stack machine. He also implemented the built-in preprocessor.
John S. Price found and fixed many bugs.
The shadow idea was "forwarded" to me by John S. Price from Team Cthulhu …
```

**然後 FluffOS 那份就停了**（結尾是 Petri Wessman 修 SCO unix 與 AIX）。

**CD 那份繼續往下寫了十年**：

```
In 1992 the main development work was done by
  Johan Andersson (jna@cd.chalmers.se)
Until 06.00 the development was mainly done by
  Anders Chrigström and Jacob Hallen
Current development of this version is mainly done by
  Erik Gävert

Mike McGaughy made the shared string implementation.
Joern Renecke made the original implementation of switch and the AVL tree
  implementation of smalloc.
Ronny Wikh and Lennart Augustsson implemented the first version of mappings.
Lennart Augustsson made bibopmalloc.
Thorsten Lockert fixed numerous bugs, rewrote interpret.c and redid much of
  the build procedure.
Dave Richards corrected many performance problems …
```

**同一份檔案，在 1991／1992 年分家之後，兩邊各自長大。**

這件事的價值和 §049 那個 TMI-2 檔頭、§050 那個 `ken@XAJH` 署名是同一類：
**它是系統自己留下的、沒有敘事動機的紀錄**（附錄 D.4 的判準）。
把兩份 CREDITS 並排，分家點就在那個共同前綴的結尾——
**不需要任何人告訴你他們什麼時候分開的。**

而且它還順帶證實了兩件技術上的事：
- **`mapping` 是後來才加的**（Ronny Wikh 與 Lennart Augustsson 做了第一版）——
  所以 §052 那棵撐起整個遊戲狀態的屬性樹，用的是一個**不在最初設計裡**的型別；
- **`switch` 與 smalloc 的 AVL 版都是 Joern Renecke 做的**，
  而 Joern Rennecke 的名字也出現在 FluffOS 那份的「找到並修掉大量 bug」名單裡——
  **同一個人的貢獻跨在分家的兩側。**

**解決的痛點**：這一節解的是選型與理解上的一個常見錯誤——
**把一個生態裡最流行的實作當成那個概念的定義。**

如果只看 MudOS→FluffOS，很容易得出「LPMud 就是這樣」的結論：
apply 是字串、call_out 與 heart_beat 是兩個東西、
安全模型是 uid/euid、intermud 是 mudlib 的事。
**五支並排之後，這四條全部變成「MudOS 的選擇」，而不是「LPMud 的本質」。**

**優點 / 罩門**：這種橫向對照的優點是它會**把隱含假設逼出來**。
罩門是它需要真的去讀別人的實作——
本節的每一格都來自實際的倉庫與文件，而不是從一支推論其他四支。

**效益**：對本書的收束意義：第八篇問「連線的另一端是什麼」，
第九、十篇往下鑽了三層，而這一節是最後一次拉遠——
**你手上那台 driver 的每一個設計，都有至少一個活著的反例。**

而這正是本書從第一頁就在講的那件事的另一種說法：

> **知道一個東西「是什麼」，和知道它「為什麼不是別的樣子」，是兩種不同的理解。**
> 前者讓你會用，後者才讓你知道哪些東西可以改。

> 💡 君之一席話
> **不要把生態裡最流行的實作當成那個概念的定義**——找到那些做了相反選擇而且活著的實作，你才會知道哪些是本質、哪些只是某個人在 1992 年的一個決定。

> 🔍 老手視角──真正的門道
> 這一節示範的方法叫**橫向對照**，它的成本很低但幾乎沒有人做。多數人理解一個技術的路徑是「讀最流行那個的文件 → 讀它的原始碼 → 覺得自己懂了」，而這條路徑有一個結構性的盲點：**你學到的每一個設計決定，都不知道它有沒有替代方案。** 修正的方法只有一個——**去讀那個做了相反選擇的實作**，哪怕只讀它的 API 文件與目錄結構（本節 CD 的部分幾乎全部來自 `doc/` 目錄與檔案清單，沒讀一行 C）。半天的閱讀量，能把「這就是唯一做法」變成「這是三種做法之一，而它們的取捨是……」。可落地的做法：學任何技術時，**同時打開它的一個競爭者的文件目錄**，只比對兩件事——**名詞表**與**目錄結構**。名詞不同的地方就是概念分歧點，目錄不同的地方就是架構分歧點；這兩張圖比任何一份 tutorial 都更快讓你看見設計空間。
