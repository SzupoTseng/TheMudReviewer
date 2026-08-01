# 附錄 A　opcode 總索引

> 本索引是全書的鍵。每一行都可回溯到第二篇（核心）或第三篇（擴充）的說明。

> 證據等級：核心 27 個為 🟡 伺服器 `zjmud.h` ＋ 🟢 實機；擴充為 🟡 header ∩ `.c` 使用統計。

---

## A.1　核心 27 個（14 個 mudlib 共用）

| opcode | 巨集 | 語意 | 備註 |
|--------|------|------|------|
| `ESC000` | `SYSY` | 連線／登入狀態 | 見 §008 |
| `ESC001` | `INPUTTXT` | 文字輸入面板 | `說明$zj#指令樣板` |
| `ESC002` | `ZJTITLE` | 房間標題 | **有清空副作用** |
| `ESC003` | `ZJEXIT` | 出口列表 | `dir:label[:cmd]` |
| `ESC004` | `ZJLONG` | 房間描述 | 樣式文字 |
| `ESC005` | `ZJOBIN` | 房間物件 | `label:cmd` |
| `ESC006` | `ZJBTSET` | 自訂按鈕 | `slot:label:cmd`，slot=bs/b1–b17 |
| `ESC007` | `ZJOBLONG` | 目標詳情 | **伺服器用最多，531 次** |
| `ESC008` | `ZJOBACTS` | 動作列① | 帶版面前綴 |
| `ESC009` | `ZJOBACTS2` | 動作列② | **448 次** |
| `ESC010` | `ZJYESNO` | NPC 對話框 | `$dh#` 區塊 |
| `ESC011` | `ZJMAPTXT` | 地圖 | `$br#` 換行 |
| `ESC012` | `ZJHPTXT` | 屬性條群 | `║` 分隔，色碼可能是 ARGB |
| `ESC013` | `ZJMORETXT` | 長文本分頁 | `$br#` 換行 |
| `ESC014` | `ZJFORCECMD` | **伺服器代送指令** | 98 次，不實作會斷流程 |
| `ESC015` | `ZJTMPSAY` | 頂部飄字 | 同時進系統頻道 |
| `ESC016` | `ZJFMSG` | 戰鬥訊息 | — |
| `ESC017` | `ZJFNOMSG` | 關閉戰鬥面板 | — |
| `ESC020` | `ZJPOPMENU` | 彈出選單 | **用 `$z2#` 不是 `$zj#`** |
| `ESC021` | `ZJTTMENU` | 標題列按鈕 | `label:cmd` |
| `ESC022` | `ZJCHARHP` | 物件血條更新 | `tag$zj#a:b:max` |
| `ESC023` | `ZJLONGXX` | 描述顯示開關 | `屏蔽描述` |
| `ESC024` | `—` | 傷害飄字 | 客戶端反編譯獨有 |
| `ESC045` | `—` | 開啟網頁 | 客戶端反編譯獨有 |
| `ESC100` | `ZJCHANNEL` | 聊天訊息 | — |
| `ESC900` | `—` | 換伺服器 | `ip:port` |
| `ESC903` | `ZJEXITRM` | 刪除單一出口 | `0xx` 的反向 |
| `ESC905` | `ZJOBOUT` | 刪除單一物件 | 同上 |
| `ESC913` | `ZJEXITCL` | 清空所有出口 | 無對應正向操作 |
| `ESC997` | `—` | 單行指令模式 | `\n` → `;` |
| `ESC998` | `—` | 多行指令模式 | 預設 |
| `ESC999` | `SYSEXIT` | 強制退出 | — |

> **實作優先順序**：`007`／`009`（詳情＋動作列）佔伺服器發送量絕大多數，其次是 `014`（代送指令）。先把這三個做扎實，玩家 95% 的時間都在看它們。

---

## A.2　擴充 opcode（依 profile 分組）


### 大梦江湖（新協議） — 73 個

| opcode | 巨集 | 結構 | 去處 |
|--------|------|------|------|
| `ESC030` | `ZJEXIT1` | 文字 | map |
| `ESC10a` | `XYMRX` | 文字 | escort |
| `ESC111` | `DMJHLOOK` | 文字 | self |
| `ESC11a` | `XYMRU` | 文字 | escort |
| `ESC130` | `XYTISHI` | 訊息 | sys |
| `ESC211` | `XYLIE` | 標題 | list |
| `ESC212` | `XYLIF` | 清單 | list |
| `ESC213` | `XYLIG` | 文字 | list |
| `ESC214` | `XYLIK` | 動作網格 | list |
| `ESC215` | `XYLIL` | 關閉面板 | list |
| `ESC217` | `XYJI` | 文字 | list |
| `ESC234` | `ZJHPTXT1` | 數值條 | stats2 |
| `ESC270` | `XYZFUBEN` | 標題 | instance |
| `ESC271` | `XYZFBTE` | 文字 | instance |
| `ESC272` | `XYZFBLIE` | 清單 | instance |
| `ESC286` | `DMMAP` | 文字 | map |
| `ESC291` | `XYZZHSXBT` | 標題 | attr |
| `ESC292` | `XYZZHSXXX` | 清單 | attr |
| `ESC293` | `XYZZHSXWB` | 文字 | attr |
| `ESC294` | `XYZZHSXBUT` | 動作網格 | attr |
| `ESC295` | `XYZZHSXBUT1` | 動作網格 | attr |
| `ESC296` | `XYZZHSXWB1` | 文字 | attr |
| `ESC297` | `XYZZHSXWB2` | 文字 | attr |
| `ESC2k1` | `XJYS1` | 文字 | attr |
| `ESC2k2` | `XJYS2` | 文字 | attr |
| `ESC2k3` | `XJYS3` | 文字 | attr |
| `ESC2k4` | `XJYS4` | 文字 | attr |
| `ESC308` | `XYTEXT3` | 動作網格 | item |
| `ESC309` | `XYTEXT2` | 文字 | item |
| `ESC310` | `XYTEXT1` | 標題 | item |
| `ESC311` | `XYTEXT` | 文字 | item |
| `ESC331` | `DMJHPAI` | 標題 | rank |
| `ESC332` | `DMJHPAILEI` | 清單 | rank |
| `ESC333` | `DMJHPAIMING` | 文字 | rank |
| `ESC343` | `XYSHOPJZ` | 文字 | shop |
| `ESC344` | `XYSHOPLX` | 清單 | shop |
| `ESC345` | `XYSHOP` | 動作網格 | shop |
| `ESC346` | `XYBBTEXT` | 文字 | bag |
| `ESC347` | `XYBEIBAO` | 動作網格 | bag |
| `ESC348` | `XYCWD` | 動作網格 | store |
| `ESC349` | `XYCWDT` | 文字 | store |
| `ESC350` | `XYBEIBAOT` | 文字 | bag |
| `ESC417` | `XYRWNAME` | 標題 | person |
| `ESC418` | `XYRWMIAO` | 文字 | person |
| `ESC419` | `XYRWBUT1` | 動作網格 | person |
| `ESC420` | `XYRWBUT2` | 動作網格 | person |
| `ESC421` | `XYRWBUT3` | 文字 | person |
| `ESC450` | `YJBUTTON` | 動作網格 | misc |
| `ESC491` | `XYFRIENDS1` | 動作網格 | friends |
| `ESC492` | `XYFRIENDS2` | 動作網格 | friends |
| `ESC493` | `XYFRIENDS3` | 動作網格 | friends |
| `ESC494` | `XYFRIENDS4` | 動作網格 | friends |
| `ESC495` | `XYFRIENDS5` | 文字 | friends |
| `ESC496` | `XYFRIENDS6` | 訊息 | chat |
| `ESC497` | `XYFRIENDS7` | 文字 | friends |
| `ESC498` | `XYFRIENDS8` | 文字 | friends |
| `ESC511` | `XYKILL` | 實體新增 | combat |
| `ESC512` | `XYKILLD` | 實體移除 | combat |
| `ESC513` | `XYKILLDY` | 實體新增 | combat |
| `ESC514` | `XYKILLDT` | 實體移除 | combat |
| `ESC515` | `XYKILLMIAO` | 訊息 | combat |
| `ESC516` | `KILLEND` | 關閉面板 | combat |
| `ESC517` | `KILLJN` | 動作網格 | combat |
| `ESC518` | `KILLKS` | 訊息 | combat |
| `ESC519` | `KILLQL` | 關閉面板 | combat |
| `ESC521` | `XJTILI` | 數值條 | combat |
| `ESC602` | `DMZHUYET` | 文字 | home |
| `ESC603` | `DMZHUYETY` | 文字 | home |
| `ESC604` | `DMZHUJMQH` | 關閉面板 | home |
| `ESC605` | `DMZHUJIU` | 文字 | home |
| `ESC701` | `XCGX` | 文字 | misc |
| `ESC702` | `JHYJMP` | 標題 | sect |
| `ESC703` | `JHYJMP1` | 清單 | sect |

### 指游 MUD（ZY） — 20 個

| opcode | 巨集 | 結構 | 去處 |
|--------|------|------|------|
| `ESC1085` | `ZYMAP` | 文字 | map |
| `ESC600` | `ZYRIGHTBTN` | 動作網格 | quickside |
| `ESC601` | `ZYHPTXT` | 實體新增 | status |
| `ESC602` | `ZYCHARHP` | 實體新增 | status |
| `ESC603` | `ZYFIGHTBTN` | 動作網格 | combat |
| `ESC604` | `ZYHANDLERBTN` | 動作網格 | handler |
| `ESC605` | `ZYCLEARSCREEN` | 清空主區 | - |
| `ESC606` | `ZYRIGHTMENU` | 清單 | sidemenu |
| `ESC607` | `ZYVOICE` | 明確忽略 | - |
| `ESC608` | `ZYPROGRESS` | 數值條 | status |
| `ESC609` | `ZYSYSEXIT` | 延時重登 | - |
| `ESC610` | `ZYSTATUSINFO` | 明確忽略 | - |
| `ESC611` | `ZYCLIENTSTATUS` | 網路狀態 | - |
| `ESC612` | `ZYSELECT` | 元件提示 | - |
| `ESC613` | `ZYATTACK` | 傷害飄字 | - |
| `ESC614` | `ZYSKILL` | 動作網格 | skill |
| `ESC615` | `ZYSTORYTEXT` | 逐字劇情 | - |
| `ESC616` | `ZYVIBRATE` | 震動 | - |
| `ESC617` | `ZYBTSET` | 動作網格 | quickside |
| `ESC618` | `ZYKJ` | 動作網格 | hotkey |

---

## A.3　⚠️ 跨方言撞號

| opcode | 大梦江湖 | 指游 |
|--------|----------|------|
| `602` | `DMZHUYET` 主頁文本 | `ZYCHARHP` NPC 狀態 |
| `603` | `DMZHUYETY` 主頁文本 2 | `ZYFIGHTBTN` 戰鬥按鈕 |
| `604` | `DMZHUJMQH` 判斷界面 | `ZYHANDLERBTN` 操作面板 |
| `605` | `DMZHUJIU` 更改界面 | `ZYCLEARSCREEN` 清空主畫面 |

**這四個碼證明了「所有方言合併成一張表」在結構上就是錯的**——詳見 §010。

---

## A.4　特殊格式

| 項目 | 說明 |
|------|------|
| **4 字元 opcode** | 只有 `1085`（`ZYMAP`）。消歧義規則見 §011 |
| **含字母的 opcode** | `10a`、`11a`、`2k1`–`2k4`。正則不可寫 `\d{3}` |
| **未知 opcode** | 一律降級成一般訊息並**保留完整原文**，標記 `unknownOpcode`。絕不靜默丟棄 |

---

## A.5　分隔符速查

| 標記 | 用途 |
|------|------|
| `$zj#` | 記錄分隔符（主力） |
| `$z2#` | 記錄分隔符（**僅彈出選單**） |
| `$br#` | 換行 |
| `$dh#` | 對話框區塊 |
| `$sock#` | 指令序列（一次送多條） |
| `║` (U+2551) | 屬性條記錄／登入欄位 |
| `$N` | 數量佔位符（**僅 `ESC010` 對話框**） |
| `$txt#` | 輸入佔位符（**僅 `ESC001`**）／動作列的「保持面板開啟」旗標 |
| `:` `\|` | 欄位／次級欄位 |

> ⚠️ `$N` 與 `$txt#` 是兩套不同機制，**不可混用**——這是本專案 v1.0 規格書的兩個錯誤之一。
