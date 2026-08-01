# 附录 A　opcode 总索引

> 本索引是全书的键。每一行都可回溯到第二篇（内核）或第三篇（扩充）的说明。

> 证据等级：内核 27 个为 🟡 服务器 `zjmud.h` ＋ 🟢 实机；扩充为 🟡 header ∩ `.c` 使用统计。

---

## A.1　内核 27 个（14 个 mudlib 共用）

| opcode | 宏 | 语意 | 备注 |
|--------|------|------|------|
| `ESC000` | `SYSY` | 连接／登录状态 | 见 §008 |
| `ESC001` | `INPUTTXT` | 文本输入面板 | `说明$zj#指令样板` |
| `ESC002` | `ZJTITLE` | 房间标题 | **有清空副作用** |
| `ESC003` | `ZJEXIT` | 出口列表 | `dir:label[:cmd]` |
| `ESC004` | `ZJLONG` | 房间描述 | 样式文本 |
| `ESC005` | `ZJOBIN` | 房间对象 | `label:cmd` |
| `ESC006` | `ZJBTSET` | 自订按钮 | `slot:label:cmd`，slot=bs/b1–b17 |
| `ESC007` | `ZJOBLONG` | 目标详情 | **服务器用最多，531 次** |
| `ESC008` | `ZJOBACTS` | 动作列① | 带版面前缀 |
| `ESC009` | `ZJOBACTS2` | 动作列② | **448 次** |
| `ESC010` | `ZJYESNO` | NPC 对话框 | `$dh#` 区块 |
| `ESC011` | `ZJMAPTXT` | 地图 | `$br#` 换行 |
| `ESC012` | `ZJHPTXT` | 属性条群 | `║` 分隔，色码可能是 ARGB |
| `ESC013` | `ZJMORETXT` | 长文本分页 | `$br#` 换行 |
| `ESC014` | `ZJFORCECMD` | **服务器代送指令** | 98 次，不实作会断流程 |
| `ESC015` | `ZJTMPSAY` | 顶部飘字 | 同时进系统频道 |
| `ESC016` | `ZJFMSG` | 战斗消息 | — |
| `ESC017` | `ZJFNOMSG` | 关闭战斗面板 | — |
| `ESC020` | `ZJPOPMENU` | 弹出菜单 | **用 `$z2#` 不是 `$zj#`** |
| `ESC021` | `ZJTTMENU` | 标题列按钮 | `label:cmd` |
| `ESC022` | `ZJCHARHP` | 对象血条更新 | `tag$zj#a:b:max` |
| `ESC023` | `ZJLONGXX` | 描述显示开关 | `屏蔽描述` |
| `ESC024` | `—` | 伤害飘字 | 客户端反编译独有 |
| `ESC045` | `—` | 打开网页 | 客户端反编译独有 |
| `ESC100` | `ZJCHANNEL` | 聊天消息 | — |
| `ESC900` | `—` | 换服务器 | `ip:port` |
| `ESC903` | `ZJEXITRM` | 删除单一出口 | `0xx` 的反向 |
| `ESC905` | `ZJOBOUT` | 删除单一对象 | 同上 |
| `ESC913` | `ZJEXITCL` | 清空所有出口 | 无对应正向操作 |
| `ESC997` | `—` | 单行指令模式 | `\n` → `;` |
| `ESC998` | `—` | 多行指令模式 | 缺省 |
| `ESC999` | `SYSEXIT` | 强制退出 | — |

> **实作优先级**：`007`／`009`（详情＋动作列）占服务器发送量绝大多数，其次是 `014`（代送指令）。先把这三个做扎实，玩家 95% 的时间都在看它们。

---

## A.2　扩充 opcode（依 profile 分组）


### 大梦江湖（新协议） — 73 个

| opcode | 宏 | 结构 | 去处 |
|--------|------|------|------|
| `ESC030` | `ZJEXIT1` | 文本 | map |
| `ESC10a` | `XYMRX` | 文本 | escort |
| `ESC111` | `DMJHLOOK` | 文本 | self |
| `ESC11a` | `XYMRU` | 文本 | escort |
| `ESC130` | `XYTISHI` | 消息 | sys |
| `ESC211` | `XYLIE` | 标题 | list |
| `ESC212` | `XYLIF` | 清单 | list |
| `ESC213` | `XYLIG` | 文本 | list |
| `ESC214` | `XYLIK` | 动作网格 | list |
| `ESC215` | `XYLIL` | 关闭面板 | list |
| `ESC217` | `XYJI` | 文本 | list |
| `ESC234` | `ZJHPTXT1` | 数值条 | stats2 |
| `ESC270` | `XYZFUBEN` | 标题 | instance |
| `ESC271` | `XYZFBTE` | 文本 | instance |
| `ESC272` | `XYZFBLIE` | 清单 | instance |
| `ESC286` | `DMMAP` | 文本 | map |
| `ESC291` | `XYZZHSXBT` | 标题 | attr |
| `ESC292` | `XYZZHSXXX` | 清单 | attr |
| `ESC293` | `XYZZHSXWB` | 文本 | attr |
| `ESC294` | `XYZZHSXBUT` | 动作网格 | attr |
| `ESC295` | `XYZZHSXBUT1` | 动作网格 | attr |
| `ESC296` | `XYZZHSXWB1` | 文本 | attr |
| `ESC297` | `XYZZHSXWB2` | 文本 | attr |
| `ESC2k1` | `XJYS1` | 文本 | attr |
| `ESC2k2` | `XJYS2` | 文本 | attr |
| `ESC2k3` | `XJYS3` | 文本 | attr |
| `ESC2k4` | `XJYS4` | 文本 | attr |
| `ESC308` | `XYTEXT3` | 动作网格 | item |
| `ESC309` | `XYTEXT2` | 文本 | item |
| `ESC310` | `XYTEXT1` | 标题 | item |
| `ESC311` | `XYTEXT` | 文本 | item |
| `ESC331` | `DMJHPAI` | 标题 | rank |
| `ESC332` | `DMJHPAILEI` | 清单 | rank |
| `ESC333` | `DMJHPAIMING` | 文本 | rank |
| `ESC343` | `XYSHOPJZ` | 文本 | shop |
| `ESC344` | `XYSHOPLX` | 清单 | shop |
| `ESC345` | `XYSHOP` | 动作网格 | shop |
| `ESC346` | `XYBBTEXT` | 文本 | bag |
| `ESC347` | `XYBEIBAO` | 动作网格 | bag |
| `ESC348` | `XYCWD` | 动作网格 | store |
| `ESC349` | `XYCWDT` | 文本 | store |
| `ESC350` | `XYBEIBAOT` | 文本 | bag |
| `ESC417` | `XYRWNAME` | 标题 | person |
| `ESC418` | `XYRWMIAO` | 文本 | person |
| `ESC419` | `XYRWBUT1` | 动作网格 | person |
| `ESC420` | `XYRWBUT2` | 动作网格 | person |
| `ESC421` | `XYRWBUT3` | 文本 | person |
| `ESC450` | `YJBUTTON` | 动作网格 | misc |
| `ESC491` | `XYFRIENDS1` | 动作网格 | friends |
| `ESC492` | `XYFRIENDS2` | 动作网格 | friends |
| `ESC493` | `XYFRIENDS3` | 动作网格 | friends |
| `ESC494` | `XYFRIENDS4` | 动作网格 | friends |
| `ESC495` | `XYFRIENDS5` | 文本 | friends |
| `ESC496` | `XYFRIENDS6` | 消息 | chat |
| `ESC497` | `XYFRIENDS7` | 文本 | friends |
| `ESC498` | `XYFRIENDS8` | 文本 | friends |
| `ESC511` | `XYKILL` | 实体添加 | combat |
| `ESC512` | `XYKILLD` | 实体移除 | combat |
| `ESC513` | `XYKILLDY` | 实体添加 | combat |
| `ESC514` | `XYKILLDT` | 实体移除 | combat |
| `ESC515` | `XYKILLMIAO` | 消息 | combat |
| `ESC516` | `KILLEND` | 关闭面板 | combat |
| `ESC517` | `KILLJN` | 动作网格 | combat |
| `ESC518` | `KILLKS` | 消息 | combat |
| `ESC519` | `KILLQL` | 关闭面板 | combat |
| `ESC521` | `XJTILI` | 数值条 | combat |
| `ESC602` | `DMZHUYET` | 文本 | home |
| `ESC603` | `DMZHUYETY` | 文本 | home |
| `ESC604` | `DMZHUJMQH` | 关闭面板 | home |
| `ESC605` | `DMZHUJIU` | 文本 | home |
| `ESC701` | `XCGX` | 文本 | misc |
| `ESC702` | `JHYJMP` | 标题 | sect |
| `ESC703` | `JHYJMP1` | 清单 | sect |

### 指游 MUD（ZY） — 20 个

| opcode | 宏 | 结构 | 去处 |
|--------|------|------|------|
| `ESC1085` | `ZYMAP` | 文本 | map |
| `ESC600` | `ZYRIGHTBTN` | 动作网格 | quickside |
| `ESC601` | `ZYHPTXT` | 实体添加 | status |
| `ESC602` | `ZYCHARHP` | 实体添加 | status |
| `ESC603` | `ZYFIGHTBTN` | 动作网格 | combat |
| `ESC604` | `ZYHANDLERBTN` | 动作网格 | handler |
| `ESC605` | `ZYCLEARSCREEN` | 清空主区 | - |
| `ESC606` | `ZYRIGHTMENU` | 清单 | sidemenu |
| `ESC607` | `ZYVOICE` | 明确忽略 | - |
| `ESC608` | `ZYPROGRESS` | 数值条 | status |
| `ESC609` | `ZYSYSEXIT` | 延时重登 | - |
| `ESC610` | `ZYSTATUSINFO` | 明确忽略 | - |
| `ESC611` | `ZYCLIENTSTATUS` | 网络状态 | - |
| `ESC612` | `ZYSELECT` | 组件提示 | - |
| `ESC613` | `ZYATTACK` | 伤害飘字 | - |
| `ESC614` | `ZYSKILL` | 动作网格 | skill |
| `ESC615` | `ZYSTORYTEXT` | 逐字剧情 | - |
| `ESC616` | `ZYVIBRATE` | 震动 | - |
| `ESC617` | `ZYBTSET` | 动作网格 | quickside |
| `ESC618` | `ZYKJ` | 动作网格 | hotkey |

---

## A.3　⚠️ 跨方言撞号

| opcode | 大梦江湖 | 指游 |
|--------|----------|------|
| `602` | `DMZHUYET` 主页文本 | `ZYCHARHP` NPC 状态 |
| `603` | `DMZHUYETY` 主页文本 2 | `ZYFIGHTBTN` 战斗按钮 |
| `604` | `DMZHUJMQH` 判断界面 | `ZYHANDLERBTN` 操作面板 |
| `605` | `DMZHUJIU` 更改界面 | `ZYCLEARSCREEN` 清空主画面 |

**这四个码证明了「所有方言合并成一张表」在结构上就是错的**——详见 §010。

---

## A.4　特殊格式

| 项目 | 说明 |
|------|------|
| **4 字符 opcode** | 只有 `1085`（`ZYMAP`）。消歧义规则见 §011 |
| **含字母的 opcode** | `10a`、`11a`、`2k1`–`2k4`。正则不可写 `\d{3}` |
| **未知 opcode** | 一律降级成一般消息并**保留完整原文**，标记 `unknownOpcode`。绝不静默丢弃 |

---

## A.5　分隔符速查

| 标记 | 用途 |
|------|------|
| `$zj#` | 记录分隔符（主力） |
| `$z2#` | 记录分隔符（**仅弹出菜单**） |
| `$br#` | 换行 |
| `$dh#` | 对话框区块 |
| `$sock#` | 指令串行（一次送多条） |
| `║` (U+2551) | 属性条记录／登录字段 |
| `$N` | 数量占位符（**仅 `ESC010` 对话框**） |
| `$txt#` | 输入占位符（**仅 `ESC001`**）／动作列的「保持面板打开」旗标 |
| `:` `\|` | 字段／次级字段 |

> ⚠️ `$N` 与 `$txt#` 是两套不同机制，**不可混用**——这是本项目 v1.0 规格书的两个错误之一。
