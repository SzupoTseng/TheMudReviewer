# 第十二篇　英文線：MudOS 的社群史

> 附錄 D.3 記了中文線：東方故事（1993）→ 俠客行（1995）→ 外流（1996）→ 本專案的 17 份封存。
> **但那條線是從 TMI-2 mudlib 分出去的支流。** 主流在英文世界，而書裡一直沒寫。
>
> 這一篇補上 1992–2010 年代的英文 MudOS 社群史，四個單元：
> **一個為了寫 driver 而開的 MUD、三份 mudlib 的競賽、
> MudOS 的三次死而復生、以及 FluffOS 的建國宣言。**
>
> 它回答一個本書一直沒問的問題：
> **本專案手上那台 driver，中間經過了哪些人的手？**

---

## 071　TMI — 一個為了寫 driver 而開的 MUD

**標籤**：`#TMI` `#MudOS` `#社群史` `#intermud起源` `#一手史料`
**證據等級**：🟡 一手史料（`Credits.MudOS`、`History.MudOS`）＋ 🟡 George Reese（Descartes）1995 年整理的 LPMud 編年
　　　　　　　※ 編年的作者本人列名於 `Credits.MudOS`——**他是參與者，不是旁觀者**。這既是它可信的理由，也是它有立場的理由。

**起源**：§034 引用了 1992 年那篇 Usenet 貼文，講 MudOS 為什麼分家。
但貼文裡有一個縮寫沒有解釋，而它是整條英文線的起點：

> Originally, people at **TMI** got together and asked everyone who was
> modifying their driver to get together and discuss bringing all of those
> changes together.

**TMI = The MUD Institute。**

**技術核心**：TMI 是一個很不尋常的東西——**一個為了開發軟體而開的 MUD**。

> **1992 年 2 月**：_TMI, The MUD Institute_ 對公眾開放，
> 作為一個**「用來開發新的 LPMud server 與 mudlib」的 MUD**。
> 除此之外，TMI 還立了一個章程：**教 LPC**。

它同時是三樣東西：一個遊戲、一個開發專案、一所學校。
**而這個組合直接決定了 MudOS 的性格。**

### 時間軸：TMI 的十八個月

| 日期 | 事件 |
|------|------|
| **1992-02** | TMI 開放。目標：新的 server + 新的 mudlib + 教 LPC |
| **1992-02-18** | **LPMud 3.1.2-A 專案改名為 MudOS** ← §034 那場改名的**精確日期** |
| **1992-04-23** | **LPC socket 加入 MudOS driver**。TMI 據此做出一個很粗糙的 TCP intermud 網路 |
| 1992-06 | Blackthorn 把 Genocide 搬到新的 MudOS driver。「當時 driver 充滿新功能，也**同樣充滿 bug**」——Genocide 整個夏天成了 MudOS 的測試場 |
| 1992-07 | Descartes 接手 Nightmare 的 mudlib，**丟掉它那份過時的 LPMud 2.4.5 mudlib 與 driver**，改用 MudOS |
| **1992-08** | **TMI 關站。** |
| 1992-08 | **TMI-2 開放**——目標更窄，**放棄了教學的章程** |

**三件事值得停下來看：**

### ① 改名的日期，補上了 §034 的空缺

§034 引的那篇貼文寫於 1992-08-28，講的是**改名之後**的爭議。
現在知道改名發生在 **1992-02-18**——**罵戰是在半年後才爆發的**。

這個時間差本身就說明了問題：改名當下沒有人在意，
是等到「用我們 driver 的人一直在問為什麼 compat mode 不能用」（§034）
累積了半年，事情才炸開。**相容性的爭議從來不是在改動當下發生的。**

### ② ★ 1992-04-23：本專案那 15 份封存裡的 intermud，源頭在這裡

> LPC sockets are added to the MudOS driver. **This allows TMI to create a
> very rough TCP intermud network.** This protocol is later replaced first by
> the **CDlib UDP protocols**, and later by **Intermud 3**.

把這一行和 §049 對起來看：

```
1992-04-23  MudOS 加入 socket efun
     ↓      （§047 說的「引擎加的不是功能，是可能性的邊界」）
1992       TMI 用 LPC 寫出第一版 TCP intermud
     ↓
1993-08-15 Grendel@Tmi-2 寫下 tcp.c 的檔頭
     ↓      「This file is part of the tmi mudlib. Please keep this header intact.」
1993 年底   東方故事採用 MudOS + TMI mudlib（附錄 D.3）
     ↓
2026       🟢 本專案 17 份封存中的 15 份，仍帶著那批檔案
```

**§049 那組 `_q`／`_a` 成對的服務、`dns_master` 名錄、
以及 `nitan7` 封存裡那份 118 筆的中文 MUD 名錄快照（附錄 D.4）——
它們的祖先是 1992 年 4 月 TMI 那個「很粗糙的 TCP intermud」。**

而那一行還記了它後來被取代兩次：先是 **CDlib 的 UDP 協議**
（就是 §063 那支 CD driver 的 mudlib——**它把 intermud 做進了 driver**），
再來是 **Intermud 3**。

**中文線接手的是 TMI 那一版**，而且**再也沒有升級**——
所以三十三年後，本專案在封存裡挖到的是 1993 年的實作。

### ③ 「為了寫軟體而開的 MUD」是那個年代的協作方式

沒有 GitHub、沒有 CI、沒有 issue tracker。
`Credits.MudOS` 的致謝名單裡列的不是貢獻者，是**測試場**：

> We would like to thank the following muds for extensive testing and
> numerous bug reports:
> **TMI, Portals, Overdrive, Genocide, TMI-2, Vincent's Hollow,
> DreamShadow, Nightmare, Nanvaent**, and many others.

**九個 MUD 就是九個生產環境的測試叢集。**
「driver 有 bug」的回報路徑是：玩家在 Genocide 掉線 →
管理者 Blackthorn 回報 → Truilkan／Jacques／Wayfarer 在 Portals 上改 code。

而那個「開發站」是誰提供的，寫在同一份檔案的最後一行：

> Thanks to **Adam Beeman (Buddha)** for making numerous suggestions on
> improving the MudOS driver and **for providing the TMI site as the original
> MudOS development site**.

**解決的痛點**：TMI 解的是 1992 年一個非常實際的問題——
**LPMud 的 driver 由一個人維護，而那個人很忙**（§034）。
社群的答案不是 fork 一份 code，是**開一個地方讓大家在裡面一起改**。

**踩過的坑**：TMI 開了六個月就關站。
§034 那篇貼文的自述解釋了為什麼：

> About 10 people showed up and we all talked for a while. For a month or two,
> we had a bulletin board where all of us posted ideas / suggestions / etc.
> **However, nobody really had the time to get together and work on the code.**

**開會的人很多，寫 code 的只有兩個。**
TMI-2 接手時明確**放棄了教學章程**——把目標收窄到只做 mudlib。

**優點 / 罩門**：優點是它真的產出了東西——MudOS、TMI-2 mudlib、
以及第一版 intermud，三樣都活了三十年以上。
罩門是**它把「社群治理」與「軟體開發」綁在一起**，
而這兩件事的節奏完全不同：治理需要共識，開發需要有人動手。
TMI 死於前者，MudOS 活下來是因為後者被少數幾個人扛走了。

**效益**：對本專案而言，這一節解釋了一件事——
**為什麼本專案手上的 mudlib 帶著一批 1993 年、來自一個已經關站的 MUD 的檔案。**
不是因為誰偷懶沒更新，是因為**那批檔案在它被複製出去的那一刻，就已經是孤兒**：
TMI 1992 年就關了，TMI-2 的 mudlib 1993 年發布，
而中文線 1993 年底接手——**接手時上游已經不在了。**

> 💡 君之一席話
> **一個為了開發軟體而開的社群，最後留下來的往往不是社群，是軟體**——TMI 存在了六個月，而它寫的那批檔案還在三十三年後的封存裡。

> 🔍 老手視角──真正的門道
> 「用九個生產環境當測試叢集」這件事，在今天看起來很原始，但它有一個現代 CI 給不了的性質：**那些 bug 是真的玩家在真的世界裡踩出來的**。Genocide 整個夏天當測試場，回報的不是「單元測試失敗」，是「三十個人同時上線的時候崩了」。老手都知道這類 bug 最難用測試覆蓋——它們是負載、時序、狀態累積的交互作用。今天的對應物是金絲雀部署、影子流量、feature flag 逐步放量，而它們解決的是同一個問題：**在真實環境裡取得早期訊號，但不要一次賠上全部使用者**。1992 年的做法是靠「有幾個管理者願意當白老鼠」，2026 年的做法是靠流量切分——**機制變了，原理沒變**。可落地的判準：如果你的系統只在 CI 裡被驗證過，那你其實只驗證了你想得到的那些情況；**永遠要有一個「真實但可犧牲」的環境**。

---

## 072　三份 mudlib 的競賽 — 而競賽的主軸是指令解析器

**標籤**：`#mudlib` `#Nightmare` `#Discworld` `#TMI-2` `#LIMA` `#指令解析`
**證據等級**：🟡 George Reese 的 LPMud 編年 ＋ 🟢 本專案封存的實測對照

**起源**：§050 用檔名集合量出本專案的 17 份封存來自兩個家族，
而兩族的共同祖先是 **TMI-2**。
但 TMI-2 在英文世界只是**三份之一**——而且不是最強的那一份。

**技術核心**：MudOS 的 mudlib 生態，1992–1995 的四年。

| 日期 | mudlib | 定位 |
|------|--------|------|
| **1992-12** | **Nightmare Mudlib**（Descartes） | **第一份公開可用的 MudOS mudlib**。當時 MudOS「在眾多 driver 中還被視為新來的」 |
| **1993-03** | **Discworld Mudlib** | 「**當時最先進的指令解析器與使用者介面**」——mudlib 的選擇幫助推高了 MudOS 的人氣 |
| **1993-04** | **TMI-2 Mudlib** | 「讓 MudOS 擁有三份廣泛可得的 mudlib」 ← **🟢 本專案 15/17 份封存的祖先** |
| 1995-05-15 | **Foundation II** | **第一份為非遊戲用途設計的 LPC 函式庫**；**第一份可商業授權的 MudOS mudlib** |
| 1995-06-15 | **Nightmare IV** | 完全重寫，**清掉一路追溯到 LPMud 2.4.5 的遺留程式碼** |
| 1995-07-21 | **LIMA**（pre-alpha） | 「**LPMud 上見過最先進的指令解析**」，**基於老 Infocom 遊戲（Zork）的解析方式** |

另外還有兩本教科書，說明這個生態已經大到需要教材：

| 日期 | 書 |
|------|---|
| 1993-04 | **LPC Basics** —— 第一本涵蓋 LPC 所有實作的完整教科書 |
| 1993-11 | **Intermediate LPC** |

**★ 這張表最重要的訊息在「先進」這個詞指的是什麼。**

Discworld（1993）與 LIMA（1995）被稱讚的都是**指令解析器**。
這不是巧合——**在文字遊戲裡，指令解析器就是使用者介面。**

```
玩家打：  get the red sword from the wooden chest
          └───────────── 這一行要被拆成 動詞／限定詞／物件／介系詞／容器
```

MudOS driver 有 `parse_command()` efun（§047 那張表裡的「自然語言式指令」），
但**它只是機制**——把它變成「玩家可以自然地說話」需要 mudlib 做非常多工作。

**LIMA 直接對標 Infocom 的 Zork**，那是文字冒險解析的天花板。

> 🔬 **本書的角度**：這正好是 §054 那條線的另一端。
> §054 說 MUD 客戶端三十年的四件套（trigger／alias／macro／script）
> 是因為「伺服器只吐文字」；
> **這裡看到的是同一個限制在伺服器那一側的形狀——
> 既然只有文字，那就把文字的「輸入」與「輸出」兩端都做到極致。**
>
> 英文 mudlib 把力氣花在**輸入端**（指令解析器）；
> 而 §043 那套 ZJMUD 方言把力氣花在**輸出端**（把狀態明講）。
> **兩條路線都是對「純文字」這個前提的回應，只是選了不同的那一端。**

**Discworld 這條線特別值得標記**，因為它會在 §074 再出現一次：
1993 年做出最先進解析器的那份 mudlib，
它的核心開發者 **David Bennett（Pinkfish）** 同時在改 MudOS driver 本身——
`Credits.MudOS` 列他的貢獻是：

> provided enhancements for **parse_command** and for the **telnet negotiation
> code in comm.c**, and the **resolve()** efun.

**mudlib 的需求直接推動了 driver 的功能。**
Discworld 要更好的解析器，於是 `parse_command` 被加強；
要更好的終端支援，於是 telnet 協商被改寫——
**而本書 §055 量到的那些協商位元組，走的就是那段被 Pinkfish 改過的程式碼的後裔。**

**解決的痛點**：三份 mudlib 並存解的是 §034 沒解的那個問題——
**「拿一份現成的東西開站」**。
MudOS 在 1992 年被視為「新來的」，
而它在 1993 年變得流行，靠的不是 driver 本身變好，
是**同時有三份可用的 mudlib**。

**踩過的坑（結構性的）**：三份並存的代價是**分裂**。
Nightmare、Discworld、TMI-2 的 API 彼此不相容，
一份為 Nightmare 寫的區域搬不到 Discworld。
§050 量到的那個現象（兩個家族、Jaccard 0.012）在英文世界是三倍嚴重。

而且——**中文線在 1993 年接走了 TMI-2 那一份之後，
就與後面的所有發展斷了聯繫**：
Foundation II（1995）、Nightmare IV（1995）、LIMA（1995）
在本專案的 17 份封存裡**一個痕跡都沒有**。

**優點 / 罩門**：優點是競爭真的推動了技術——
1993 到 1995 年，MudOS mudlib 的指令解析從「動詞＋名詞」
走到了 Infocom 等級。
罩門是**每一份 mudlib 都是一個孤島**，而孤島越多，
單一孤島的維護人力越少。

**效益**：對本專案而言，這一節校正了一個容易產生的誤解。
§049–§053 把 TMI-2 的架構講得很細（因為那是本專案手上的東西），
但**TMI-2 從來不是英文世界的主流選擇**——
它是三份之一，而且不是解析器最強的那一份。

**本專案的 17 份封存繼承的，是 1993 年那個生態的一個切片，
而不是那個生態的最終狀態。**
這件事在做任何「中文 MUD 的技術水準如何」的判斷時都必須先說清楚。

> 💡 君之一席話
> **一個平台流行起來，通常不是因為它自己變好了，是因為它上面終於有了三個可以直接拿來用的東西。**

> 🔍 老手視角──真正的門道
> 「mudlib 的需求推動 driver 的功能」這個方向值得注意，因為它和多數人的直覺相反：**不是引擎先提供能力、上層才長出應用，而是上層先痛、再回頭改引擎。** Pinkfish 為了 Discworld 的解析器去改 MudOS 的 `parse_command`，為了終端體驗去改 `comm.c` 的 telnet 協商——他是使用者，也是貢獻者。這種「使用者即貢獻者」的迴路是開源生態最健康的形態，而它有一個前提：**引擎與應用的作者要在同一個溝通半徑內**。TMI 那種「開一個 MUD 讓大家在裡面一起改」的做法，本質上就是在人為製造這個半徑。老手評估一個開源專案的健康度時，看的不是 star 數，而是**「最近半年的 commit 裡，有多少來自真的在用它的人」**——全部來自核心維護者的專案，通常已經在解決想像中的問題了。可落地的做法：如果你在做平台，**想辦法讓自己成為它的一個真實使用者**；做不到的話，至少要有一條讓真實使用者的痛點在兩週內傳到你手上的通道。

---

## 073　MudOS 的三次死而復生

**標籤**：`#MudOS` `#BeekOS` `#LPC到C` `#維護者交接` `#compat_buster`
**證據等級**：🟡 George Reese 編年 ＋ 🟡 一手史料（`Credits.MudOS`、`README.MudOS`、`ChangeLog.mudos` 2,217 行）

**起源**：本書從 §034 之後就一直把 MudOS 當成一條連續的線在講。
**它不是。** 它至少停過三次，而且有兩次差點就沒有下文了。

**技術核心**：

### 第一次：1993 年底 —— 「MudOS 正在死去」

> **1993 年 11 月**：由於 LPMud 的速度、DGD 的優雅，
> 以及**MudOS 這邊單純缺乏進展**，MudOS 正在死去。
> **Beek 於是做了 BeekOS。**

BeekOS 是什麼：

> BeekOS 基本上是一個 MudOS 核心，加上**LPC→C 的動態編譯**，
> 把編譯出來的機器碼**動態連結進正在執行的 server**。

**這是 §037 那個「純解譯堆疊機的效能天花板」的第一次突圍嘗試。**
本書 §037 說過：「所有效能都靠 efun（把熱點推回 C++），
這正是為什麼一個 MUD driver 會有 600 多個內建函式」。
**Beek 的答案是：那就把 LPC 本身編成 C。**

值得注意的是**時序**：DGD 在 **1993-09-16** 的 1.0.a4 版
成為第一個支援 LPC→C 編譯的 driver；
**兩個月後 Beek 就在 MudOS 上做出了對應的東西。**
編年裡明說 BeekOS 的動機之一就是「DGD 的優雅」——
**競爭產生的技術壓力，在這裡有明確的日期可以對上。**

這些改動後來在 Beek 接手 MudOS 之後被併回主線。

### 第二次：1994 年初 —— 唯一的維護者也停手了

> **1994 年初**：Genocide 為了速度轉回 LPMud。
> 結果 **Blackthorn 停止了那涓滴般的 bug 修復，
> 而那正是當時 MudOS 開發的全部。**

**「that trickle of bug-fixes which had been the whole of MudOS development」**——
一句話說完那個時期的狀態：整個 MudOS 的開發，
就是一個人在他自己的 MUD 需要時順手修幾個 bug。
他的 MUD 一換 driver，**開發就歸零**。

### 第三次：1994 年中 —— 三個人接手

> **1994-06-26**：**Beek、Robocoder、Symmetry 接手 MudOS 開發。**
> 這標誌著在**將近九個月的無開發狀態**之後，
> MudOS 重新引起興趣。

`Credits.MudOS` 對這三個人的記載，正好對應他們接手後做的事：

| 人 | 貢獻（`Credits.MudOS` 原文） |
|----|---------------------------|
| **Tim Hollebeek (Beek)** | **LPC→C and patches, compiler rewrite, new function pointers, etc** |
| **Anthon Pang (Robocoder)** | various patches, **optimize function table search**, amiga support |
| **Ti-Ming Chiang (Symmetry, Cloud)** | **Countless optimizations; array, mapping and save_object rewrites** |

**三個人分別接手了編譯器、函式表、與資料結構**——
這是一次很典型的「重寫核心」分工。

**Beek 那一行的 “compiler rewrite” 與 “new function pointers”
就是今天 FluffOS 裡那個編譯器與 `(: :)` 語法的來源。**

### 版本編號與「compat buster」

`README.MudOS` 記下了 MudOS 的發行慣例，而其中一條很值得學：

> In particular, note the changes marked **"compat buster"** since these
> describe changes to the driver which **may cause pre-existing mudlibs
> (such as Lil) to break** in some minor fashion (the ChangeLog should give
> enough information to allow affected mudlibs to be fixed).

**他們在 ChangeLog 裡替破壞性變更加了一個標籤。**

對照 §059 講 FluffOS 拆 `static` 時說的「**可偵測的破壞性變更**」——
MudOS 在 1990 年代用的是更原始但同樣有效的手段：
**在人類可讀的變更紀錄裡，把會弄壞你的那幾行標出來。**

其他發行慣例：

| 慣例 | 內容 |
|------|------|
| 版號 | `v20` / `v21` / `v22`（單一數字＝含文件的完整包）；`v20.26`（小版號＝只有原始碼） |
| alpha/beta | **只以 patch 形式提供**，且有獨立的 `ChangeLog.alpha` / `ChangeLog.beta` |
| COMPAT | **`README.MudOS` 開宗明義：「It does not support COMPAT mode」**——§034 那個不相容宣告，寫在使用者第一眼會看到的地方 |
| 支援站 | `http://www.mudos.org/`，含 bug 資料庫；當機時先讀 `IT_CRASHED` 收集資訊 |

### 移植狂潮：MudOS 被搬上了所有東西

`Credits.MudOS` 裡有一整批貢獻者的功勞是**移植**，
把它們排出來就是一份 1990 年代的作業系統清單：

| 平台 | 誰 |
|------|-----|
| AIX 3.2 | Dwayne Fontenot (Jacques) |
| Sequent Symmetry（System V R3） | Dave Richards (Cynosure/Cygnus) |
| System V R4 | Michael Bundy |
| 386BSD | Luke Mewborn (Zak) |
| **Linux 0.99.3** | Olav Kolbu (Aragorn) |
| DEC Alpha | Bob Farmer (Blackthorn) |
| Amiga | John Fehr (Wildcard)、Anthon Pang、Maarten de Jong |
| Win32（原始版） | **George Reese (Descartes)** |

**`Linux 0.99.3`** 那一行是個時間戳：那是 1993 年的核心版本。
**MudOS 在 Linux 1.0 出來之前就跑在上面了。**

而 §035 那張「FluffOS 支援 Linux／macOS／Windows／Alpine／WebAssembly」的表，
其實是這個傳統的延續——**這條血脈從一開始就有很強的移植文化**。

**解決的痛點**：這三次死而復生解的其實是同一個問題——
**開源專案的維護者風險（bus factor）**。

三次的模式一模一樣：

```
維護只靠一個人（或一個 MUD 的需求）
   → 那個人 / 那個 MUD 的處境改變
     → 開發歸零
       → 有人受不了，自己做一個分支（BeekOS）
         → 分支被併回，做分支的人成為新維護者
```

**而 §074 會看到，這個循環在十年後又跑了一次。**

**踩過的坑**：`ChangeLog.mudos` 有 2,217 行，
而它的開頭第一句話很坦白：

> This file is the ***brief*** version of what has been changed;
> see `ChangeLog.alpha`, `ChangeLog.beta`, and `ChangeLog.old/`
> for all the gory details.

**這是「簡短版」。** 一個沒有 issue tracker、沒有 PR、
沒有自動化測試的年代，變更紀錄就是專案的全部記憶——
所以它必須非常詳細，而詳細到 2,217 行還只是簡短版。

**優點 / 罩門**：優點是這條線**每次都有人接住**。
罩門是每次接住的人都不一樣，而**每一次交接都伴隨一次核心重寫**
（BeekOS 的 LPC→C、Beek 的 compiler rewrite、Symmetry 的 array/mapping/save_object 重寫）。
§035 說 FluffOS 承諾與 MudOS 相容——
**而 MudOS 自己在 1994 年就把編譯器、陣列、mapping、存檔全部重寫過一輪。**

**效益**：對本書而言，這一節補上了 §035 那張血脈表看不到的東西：
**一條線「還活著」不代表它一直有人在推。**
MudOS 從 1992 到 1997 這五年裡，至少有九個月是完全停擺的，
而它能撐過去，靠的是三個人在 1994 年 6 月 26 日那天決定接手。

> 💡 君之一席話
> **一個開源專案的健康度，不看它有多少功能，看它換過幾次維護者而還活著**——換得過去的，才證明它不是某一個人的私有物。

> 🔍 老手視角──真正的門道
> BeekOS 這個模式很值得單獨記：**當上游停滯而你有真實需求時，正確的做法往往不是等，也不是抱怨，是做一個分支證明它可行**——然後那個分支通常會被併回，而你會成為維護者。這在開源史上反覆出現（io.js 之於 Node、Blink 之於 WebKit、本書 §074 的 FluffOS 之於 MudOS），而它成功的關鍵條件只有一個：**分支必須解決一個真實且可展示的問題**。BeekOS 解的是「太慢」，而它拿得出 LPC→C 的實作；純粹因為「上游不理我」而開的分支，幾乎都會死。老手看到自己在等上游時，會問三個問題：**這個需求真實嗎？我做得出來嗎？做出來之後上游有理由不要嗎？**三個都是 yes，就該動手了——而且要從一開始就以「將來要被併回」的方式寫。可落地的做法：分支時**保持與上游的可合併性**（不要順手重排目錄、不要換 code style），你的分支能不能回家，八成取決於這一點。

---

## 074　FluffOS 的建國宣言 — 以及 Discworld 這條暗線

**標籤**：`#FluffOS` `#分支` `#Discworld` `#Pinkfish` `#模式重演`
**證據等級**：🟡 一手史料（`ChangeLog.fluffos` 全文與 `Credits.MudOS`）

**起源**：本專案用的 driver 叫 FluffOS。
它是怎麼從 MudOS 變成 FluffOS 的？答案在 `ChangeLog.fluffos` 的**第一行**——
而那一行是一句宣言：

> **As MudOS is moving too slow to keep our driver hacks apart,
> we now call our own FluffOS :)**

**技術核心**：拆開這一句話，四個資訊全在裡面。

### ① 「MudOS is moving too slow」

**和 1993 年 Beek 做 BeekOS 的理由一模一樣**（§073：
「simple lack of progress on the part of MudOS」）。

**同一個專案、同一個原因、隔了大約十年，發生了第二次。**

### ② 「our driver hacks」——他們早就在改了

分支不是決定，是**追認**。
他們手上已經有一堆改動，而維護「我們的改動」與「上游」之間的差異
（keep them apart）成本越來越高，於是乾脆給它一個名字。

**這與 §034 MudOS 自己的分家理由結構相同**：
1992 年 TMI 那批人的版號變成 `3.0.53.A2.2`，
維護「掛在別人版號後面的自己的線」也是一樣的成本問題。

**兩次分家，同一個機制：差異的維護成本超過了分家的社交成本。**

### ③ 「we now call our own」——先改名，再談其他

§034 的老手視角說過：MudOS 的溝通順序是**先改版號（解決自己的痛）、
再改名（解決使用者的誤解）、最後才解釋動機**。

FluffOS 這一句把前兩步壓成了一步，而且**連解釋都省了**——
只有一個笑臉。

### ④ `:)` ——這是一份 ChangeLog，不是一份公告

沒有 RFC、沒有郵件列表投票、沒有基金會。
**一個人在變更紀錄的最上面加了一行。**

### FluffOS 1.0 的內容，以及它是誰做的

ChangeLog 的最末端是它的起點：

```
FluffOS 1.0:
changes since MudOS v22.2b11 (as far as i can remember)
Type warnings added
compressed save files
eval limit is now based on time taken, rather than substracting random
	numbers to account for actions.

Many bug fixes :)

Wodan.
```

四個訊息：

| 觀察 | 意義 |
|------|------|
| **「changes since MudOS v22.2b11」** | 分支點是 MudOS 的一個 **beta** 版——不是穩定版 |
| **「as far as i can remember」** | **他自己也不確定改了什麼**——這正是「差異維護成本過高」的症狀 |
| **eval limit 改成基於時間** | §037 那個 eval cost 沙盒的重做（後來 FluffOS 1.31 又改用 signals） |
| **署名 `Wodan.`** | 一個人 |

### ★ 暗線：Discworld

`ChangeLog.fluffos` 裡有一行看起來只是普通的 bug 修復：

> **fixes to compile with other settings than the DW ones**（FluffOS 1.28）

**`DW` = Discworld。**
「讓它能用 Discworld 以外的設定編譯」——
**意思是在那之前，FluffOS 的預設設定就是 Discworld 的設定。**

再往下看貢獻者名字：

```
FluffOS 1.4:  added compressedp efun, it shows if an interactive uses MCCP (pinkfish)
FluffOS 1.5:  fixed crasher after errors in save_object. (reported by pinkfish)
FluffOS 1.28: terminal_colour fix (pinkfish)
```

**Pinkfish。**

翻回 §072 引的 `Credits.MudOS`：

> **David Bennett (Pinkfish)** - provided enhancements for **parse_command**
> and for the **telnet negotiation code in comm.c**, and the **resolve()** efun.

**同一個人，出現在 1990 年代的 MudOS 貢獻者名單，
也出現在 2000 年代的 FluffOS 變更紀錄裡。**

而 §049 還記得第三次出現：本專案的封存裡，
`adm/daemons/network/` 底下有 **11 個檔案署名 `Creator : Pinkfish@Discworld`**。

**於是這條暗線完整了：**

```
Discworld MUD（1992 開站）
   │
   ├─ 1993-03  發布 Discworld Mudlib：「當時最先進的指令解析器與 UI」（§072）
   │
   ├─ 1990s    Pinkfish 改 MudOS 的 parse_command、telnet 協商、resolve()
   │              ↳ 🟢 §055 那 6 組實測協商位元組，走的是這段程式碼的後裔
   │
   ├─ 1993     Pinkfish 的 11 個檔案進了 TMI-2 的 network/ 目錄
   │              ↳ 🟢 本專案 15/17 份中文封存至今帶著它們（§049）
   │
   └─ 2000s    FluffOS 從 Discworld 的 driver hacks 誕生
                  ↳ 🟢 本專案今天用的就是它
```

**本專案手上這台 driver，與封存裡那 11 個檔案，來自同一個 MUD。**
而中文線在 1993 年接走 TMI-2 的時候，
**根本不知道它同時也接走了 Discworld 的一部分。**

### 之後：從一個人的 hack 到現代 C++ 專案

`ChangeLog.fluffos` 記錄的 1.0–1.36 是「一個人修 crasher」的階段。
今天的 FluffOS（§035）是 C++、UTF-8/ICU、64 位元、TLS/WebSocket、
GitHub Actions CI、跨五個平台含 WebAssembly——
**中間又換過一輪維護者**，而這正是 §073 那個循環的第四次。

**解決的痛點**：這一節解的是本書一個很基本、卻一直沒回答的問題——
**「我們用的這台 driver，是誰做的？」**

答案是：**沒有「誰」。**
1989 年 Lars 寫了核心；1992 年 TMI 那批人把它變成 MudOS；
1994 年 Beek 三人重寫了編譯器與資料結構；
2000 年代 Wodan 從 Discworld 的 hacks 裡分出 FluffOS；
之後又有人把它現代化到今天。

**每一段都是「上一個人停下來了，下一個人接手」。**

**踩過的坑（模式層面）**：這條線三十七年裡至少分家或停擺五次
（1991 CD、1992 MudOS、1993 BeekOS、1994 停擺九個月、2000s FluffOS），
**而每一次的觸發條件都相同：上游太慢，而下游有真實需求。**

§073 的老手視角給了那三個問題（需求真實嗎／做得出來嗎／上游有理由不要嗎），
**FluffOS 三個都是 yes，所以它活了下來。**

**優點 / 罩門**：優點是這個模式讓一個沒有公司、沒有基金會、
沒有資金的專案活了三十七年。
罩門是**每一次交接都伴隨知識流失**——
「as far as i can remember」這句話就是證據，
而 §067 提到 LDMud 的官方文件承認 ERQ 是「逆向出來的」，
是同一個病在另一條線上的症狀。

**效益**：對本書的收束意義——

本書從第一頁講的就是**逆向工程**：
面對一個沒有規格書的黑箱，怎麼把猜測逼成證據。

而這一節說明了一件事：**那個黑箱不是別人惡意造成的。**
它是三十七年、五次交接、無數次「as far as i can remember」累積出來的。
本專案花在還原 ZJMUD 協議上的力氣，
與 LDMud 維護者逆向 Amylaar 的 ERQ、
與 FluffOS 維護者搞清楚 MudOS v22.2b11 到底改了什麼，
**是同一件事。**

> 💡 君之一席話
> **「as far as i can remember」是每一個長壽專案最誠實的一行文件**——它承認了知識已經流失，而承認流失，是重新把它找回來的第一步。

> 🔍 老手視角──真正的門道
> 這一篇四個單元排在一起，會看到一個很清楚的循環：**建立社群 → 社群解散但軟體活著 → 維護者疲乏 → 有人分支 → 分支成為主線 → 回到第一步。** 三十七年跑了至少五輪。老手在評估一個要依賴很久的開源專案時，看的不是它現在有多活躍，而是**它有沒有跑完過這個循環至少一次**——因為第一次交接是最難的，跨過去的專案證明了它的價值不綁在任何一個人身上。反過來說，一個從未換過維護者的專案，無論多活躍，它的 bus factor 都是 1，而你看不出來。可落地的判準：查一個專案**最早的三個 commit 作者是否還在提交**——都在的，要問「如果他們走了會怎樣」；都不在而專案還在長，那它已經證明過自己了。**這是唯一一個不用讀 code 就能做的架構風險評估。**
