## 063　CD gamedriver — 最接近原版的那一支

**标签**：`#CD` `#Genesis` `#hook` `#alarm` `#auth` `#VBFC`
**证据等级**：🟡 上游 `cotillion/cd-gamedriver`（`doc/CREDITS`、`doc/hooks/`、`doc/efun/`、原代码树）＋ 🟡 公开史料

**起源**：§035 那张四条血脉的表里，CD 只有一句话带过：
「Genesis 那边改叫 CD」。

这句话漏掉了一件重要的事——**CD 不是 MudOS 的旁支，它是另一条主干**，
而且**它是唯一还在跑原始那个世界的 driver**：Genesis，1989 年那个 mud，至今仍在营运。

**名字的来历本身就是证据**：CD＝**Chalmers Datorförening**，
Lars Pensjö 所属的查尔摩斯理工学院电脑社。
翻回 §033 引的那份 `Credits.LPmud` 第一行：

```
Lars Pensjö, April 1989 (lars@cd.chalmers.se)
                          ↑↑
```

**`cd` 从第一天就在那里**——它不是后来取的名字，是这个项目的出生地。

1991 年底 Lars 从 Genesis 退休，其余管理者从他的开发版（LPMud 3.0）分出了 CD。
所以在时间上，**CD 的分家比 MudOS（1992-08）还早**。

**技术内核**：CD 在四个地方做了与 MudOS 系**不同的选择**，而每一个都很有意思。

### ① hook 而不是 apply

MudOS／FluffOS 叫 apply（§036）；CD 叫 **hook**，而且机制略有不同：

> The callback functions are **named in `master.n`** and are converted into
> **`M_*` named defines** that you should use within the driver.
> Use `apply_master()` to call the master object with the `M_*` defines.
> **If you do not have a function declared within the master object, 0 will be returned.**

差别在于**名字集中管理**：CD 把所有 hook 名字列在一个档（`master.n`）里，
编译期生成 `M_*` 常数，driver 内部一律用常数而不是字符串。

MudOS 系则是散在各处的字符串 apply。
**这是「命名惯例」与「集中注册」的差别**——
§036 老手视角提过的那个问题（靠命名生效的系统没人告诉你名字打错了），
CD 在 driver 这一侧解掉了一半：**driver 不会拼错，因为它用常数**。
mudlib 那一侧仍然是惯例。

CD 的 hook 清单也更小、更聚焦于生命周期：

```
start_boot  preload_boot  final_boot
start_shutdown  cleanup_shutdown  final_shutdown
runtime_error  log_error  parse_exception
object_name  flag  get_vbfc_object
```

**开机与关机各分三阶段**——比 MudOS 的 `preload` + `epilog` 细致得多。

### ② alarm 而不是 call_out ＋ heart_beat

这是 CD 最漂亮的一个设计决定。MudOS 系有两套周期机制（§038）：
`call_out()`（一次性）与 `heart_beat()`（固定间隔的 apply）。

**CD 把它们合成一个：**

```c
int set_alarm(float delay, float repeat, function func);
```

- `repeat` 为 0 → 一次性，等同 `call_out`；
- `repeat` 为正 → 重复运行，等同心跳，**而且间隔可以自订**；
- 回传 alarm id，可用 `remove_alarm` / `get_alarm` / `get_all_alarms` 管理。

> 例：`delay=3.0, repeat=4.0` → 3 秒后第一次，之后每 4 秒一次。

**两个机制变成一个 API 的两个参数。**
而且 delay 是 `float`——**次秒级精度**，MudOS 的 `call_out` 是整数秒。

文档里还留着两条很实在的警告：

> **NOTA BENE**：`set_alarm()` 调用的函数里，`this_player()` 与
> `this_interactive()` **都没有定义**——在那里 `write()` 的输出会跑进游戏日志。
>
> **WARNING**：**永远不要在重复 alarm 里再建一个重复 alarm。**
> 产生的 alarm 会呈指数成长，**不到一分钟整台 mud 就会停摆。**

第一条与 FluffOS 的 `this_player in call_out` 设置（附录 E.4.5，缺省 1）
是同一个问题的两种答案：**CD 选择不定义，FluffOS 选择提供一个开关。**

### ③ auth 而不是 uid / euid

§061 讲的 uid/euid 是照抄 Unix 的。**CD 没有照抄。**

```c
void set_auth(object ob, mixed info);
mixed query_auth(object ob);
```

> It is possible to store authority information **in any format** in the
> hidden authority variable of all objects. […] The gamedriver calls
> `valid_set_auth` in the master object, to determine if the call is legal
> **and possibly to modify the information**. The return value of
> `valid_set_auth` **is stored** in the hidden authority variable.

三个差异，每一个都是往「更泛用」的方向走：

| | MudOS 的 uid/euid | **CD 的 auth** |
|---|-----------------|---------------|
| 类型 | 字符串（借 Unix 语意） | **`mixed`——任何格式** |
| 谁决定内容 | mudlib 调用 `seteuid()`，master 只能准或不准 | **master 的 `valid_set_auth` 回传值就是最终存进去的东西**——它可以**改写** |
| 语意 | 用户／群组 | **由 mudlib 自己定义** |

**第二列是关键**：MudOS 的 `valid_seteuid()` 是一个布尔闸门
（§061 说的那个「布尔值不足以表达授权」的问题）；
**CD 的 `valid_set_auth` 是一个转换函数**——它不只决定准不准，还决定**存什么**。

**这个设计把授权策略彻底交给 mudlib**：
你想用 Unix 式的 uid，可以；想用能力清单（capability list），可以；
想存一个带到期时间的 mapping，也可以。

### ④ VBFC — 把函数调用嵌在字符串里

这是 CD 最特别、也最危险的一个机制：

```c
mixed process_value(string calldescription, void|int security);
```

字符串的格式是：

```
"function[:filename][|arg1|arg2|...|argN]"
```

也就是说，**一段字符串可以描述一次函数调用**，`process_string()` 会把它替换掉。
mudlib 用它做「值由函数调用决定」（value by function call）——
房间描述里可以嵌一段会动态求值的东西，不必写程序。

文档自己标了警示：

> **CAVEAT**：It is wise to set the security parameter […] as **any function
> in any object can be called with almost any arguments**. You probably don't
> want to have this done with your privileges.

所以有了 `security` 参数与 `get_vbfc_object` hook——
**用一个没有权限的对象当发起者**。

> 🔬 **本书的角度**：这正是 §004 那个「没有跳脱机制的分隔符」问题的
> **最极端版本**——ZJMUD 把 `$zj#` 混进文本流最坏的后果是解析错乱；
> **VBFC 把函数调用混进文本流，最坏的后果是任意代码运行。**
> 两者是同一种设计（在人类可读串流里埋机器语意）在不同风险等级上的实例。

### ⑤ 其他值得记的分歧

| 面向 | CD | MudOS 系 |
|------|-----|---------|
| mapping API | `m_delete` `m_indexes` `m_values` `m_sizeof` `mkmapping` | `map_delete` `keys` `values` `sizeof` `allocate_mapping` |
| 存盘 | `save_object`／`m_save_object`／**`save_map`／`restore_map`** | `save_object` / `restore_object` |
| 调用形式 | `call_other` `call_otherv` **`call_self` `call_selfv`** `applyv` `papplyv` | `call_other`、`evaluate` |
| **JSON** | **`val2json` / `json2val`**（driver 依赖 `libjson-c`） | 无（mudlib 自己实作） |
| 配置器 | **bibopmalloc**（Lennart Augustsson 写的 BiBOP） | smalloc → jemalloc |
| 哈希 | **siphash** | 一般字符串哈希 |
| **intermud** | **在 driver 里**：`udpsvc.c`、`tcpsvc.c`、`hname.c` | **在 mudlib 里**：socket efun ＋ LPC（§049） |

**最后一列的对比非常值得停下来看。**

§049 讲过 TMI-2 用 LPC 写了整套 intermud（`_q`／`_a` 成对、`dns_master` 名录），
那是因为 MudOS 给了 socket efun，**把能力推到上层**。

**CD 反过来：把 intermud 服务做进 driver。**
它的启动旗标里就有 `-u<port>`（UDP 端口）与 `-p<port>`（service 端口，供 ftp 等用途）——
**这些是 driver 级的设施，不是 mudlib 写出来的。**

**同一个需求，两个方向相反的答案**：
MudOS 说「我给你 socket，你自己写」；CD 说「我帮你做好」。
§047 那句「引擎加的不是功能，是可能性的边界」在这里有了对照组——
**边界推到哪里，决定了上层要不要重新发明。**

**优点 / 罩门**：CD 的优点是**它一直有一个真实的用户**——Genesis。
一个 driver 只服务一个世界，设计可以很聚焦、很少妥协，
所以它是「所有变体里最接近原版 LPMud 精神」的那一支。
罩门也是同一件事：**生态极小**。
32 颗星、174 个 commit（撰写当下），没有 MudOS 系那种数千个 mudlib 的规模。
**本项目 17 份封存没有一份是 CD 系的**，这不是巧合。

**效益**：对本书而言，CD 提供的是**对照组**。
第八、九篇所有关于「LPMud 家族长什么样」的叙述，
如果只看 MudOS→FluffOS 这一条，很容易把某些设计当成必然。
CD 证明了它们不是：
**apply 可以集中注册、call_out 与 heart_beat 可以合并、
uid/euid 可以换成任意格式的 auth、intermud 可以做在 driver 里。**

> 💡 君之一席话
> **看一个生态的设计是不是必然，最快的方法是找到那个做了相反选择、而且活得好好的实作。**

> 🔍 老手视角──真正的门道
> CD 的 `set_alarm` 把 `call_out` 与 `heart_beat` 合成一个 API，这件事值得单独想一下：它们在 MudOS 里之所以是两个东西，是因为**实作机制不同**（一个是调度队列、一个是每 tick 扫过所有开了心跳的对象），而不是因为用户的需求不同。用户的需求其实只有一个——「过一段时间叫我」——差别只在要不要重复。**MudOS 把实作的差异泄漏成了 API 的差异**，于是每个 mudlib 作者都得学两套、记两套的陷阱、写两套管理代码。这是 API 设计里最常见的一种泄漏，而它几乎总是发生在「两个功能是不同时期加的」的地方。老手在合并 API 时的判准是：**用户在选择时，问的是不是一个关于实作的问题？**——「我该用 call_out 还是 heart_beat」是关于实作的；「延迟多久、要不要重复」才是关于需求的。可落地的做法：任何有两个相似 API 的地方，试着写出「用一个 API 加参数」的版本，看看参数是不是自然——是的话，你原本那两个 API 就是实作泄漏。

---

## 064　DGD — 从头重写的那一支，以及它证明了什么

**标签**：`#DGD` `#持久化` `#轻量对象` `#原子函数` `#JIT` `#状态快照`
**证据等级**：🟡 上游官方站（dworkin.nl/dgd）与公开版本史

**起源**：§035 说 DGD 是「概念衍生，非代码衍生——重写的」。
这一节解释**重写换到了什么**——因为 DGD 的技术清单，
有一半是其他三支 driver 至今没有的东西。

**技术内核**：Felix “Dworkin” Croes 从零重写 LPMud，
而重写的第一个决定就定调了后面全部：

> **DGD does away with all of LPMud's game-oriented features,
> which can be implemented in LPC.**

**把所有游戏导向的功能从 driver 里拿掉。**

对照 §047 那张 MudOS 能力面的表——
MudOS 的路线是「batteries-included」，把能力面推到最宽；
**DGD 走了完全相反的方向：内核极小，其余用 LPC 自己盖。**

这个选择的结果是 DGD 通用到不像个 MUD driver：
**1990 年代末 Yahoo 的聊天室就是跑在 DGD 上的。**

### DGD 的五项技术创新

| 创新 | 是什么 | 其他 driver 有吗 |
|------|--------|----------------|
| **完整持久化** | **所有数据、程序与运行状态都存在一个数据库里**；可做快照、可从快照恢复、重开机续跑 | ❌ MudOS 系只有 `save_object` 这种手动存盘 |
| **轻量对象** | 两种对象：**轻量对象**（像一般 OOP 的对象，无 id、可大量产生）与**持久对象**（有全域唯一 id、能连网、能调度） | ❌（LDMud 后来借走了这个概念） |
| **原子函数（atomic function）** | **出错就把这次调用的所有改动全部回卷**——保证一致性 | ❌ 这是数据库的交易语意，出现在一个 MUD driver 上 |
| **就地重编＋hotboot** | 活着升级程序；**hotboot 换掉整个运行档而不断线** | ⚠️ MudOS 系有热重载（§060），但没有「换运行档不断线」 |
| **字节码 VM ＋选用 JIT** | 可编成机器码 | ❌ 其余三支都是纯解译（§037） |

**「原子函数」这一项特别值得停一下。**

§060 讲过 MudOS 系最难查的一类 bug：程序与数据可以独立地过期、
重载会让变量归零、世界上同时有两个版本的同一种剑。
**这些问题的共同根源是「没有交易边界」**——
一次改动不是全成功就是全失败，而是可能改到一半。

DGD 的原子函数直接把数据库的交易语意搬进了语言层。
**这不是性能优化，是把一整类 bug 从可能变成不可能。**

### 与 LPMud 的兼容关系

> DGD maintains backward compatibility with **LPMud 2.4.5** and mostly
> supports **LPMud 3.1.2**, though some features function differently.

注意它对标的是 **2.4.5 与 3.1.2**——也就是 **Lars 的官方线**，
而不是 MudOS。§034 说过 MudOS 明讲自己与 3.0 mudlib 不兼容；
**DGD 选择了另一边。**

### 版本史（技术转折点）

| 版本 | 年份 | 转折 |
|------|------|------|
| 1.0.a3 | 1993-08 | 第一个公开版 |
| 1.0.8 | 1994-08 | 第一个稳定版 |
| **1.4** | **2010-02** | **开源**（AGPL）——商业版权方无力维护后由 Dworkin B.V. 取得 |
| 1.5 | 2014-04 | 动态扩充模块、**hotboot**、字节码 VM v2 |
| 1.6 | 2017-04 | **C++ 重写**；放弃 1.5.9 之前的快照兼容性 |
| 1.7 | 2023-01 | 字节码 VM v2.4，强化 JIT 支持 |

**2010 年才开源**——这解释了为什么 DGD 的生态远小于 MudOS 系：
在中文 MUD 大爆发的 1996–2005 年，**DGD 是闭源的**（§056 说的时间轴问题，
在这里是另一个实例）。

另有一个封闭分支 **Hydra**：针对多内核优化、与 DGD 功能对等。

**解决的痛点**：DGD 解的是 LPMud 家族一个长期被忽略的问题——
**世界是易失的**。

MudOS 系的 mud 崩了，玩家会回到上一次 `save_object` 的状态；
没存的东西就没了。这在 1990 年代被当成「就是这样」。
**DGD 把它当成一个可以解决的问题，而且解法是「整个世界就是数据库」。**

**踩过的坑（生态层面）**：DGD 的技术优势从来没有转换成生态优势，
原因有三，而且没有一个是技术的：

1. **1993 年出现，2010 年才开源**——错过整个 MUD 黄金期；
2. **不兼容于 MudOS 的 mudlib**——当时绝大多数现成 mudlib 都是 MudOS 系的；
3. **极小内核意味着要自己盖很多东西**——对「拿一份 mudlib 开站」的人来说门槛更高。

**这三条在本项目身上是可以验证的**：17 份封存全部是 MudOS 系，
一份 DGD 系都没有。

**优点 / 罩门**：优点是它在 1993 年就做对了很多要到 2010 年代
才被主流理解的事（持久化、交易语意、轻量对象、hotboot、JIT）。
罩门是**它证明了技术领先与生态成功是两件事**——
而且证明得非常彻底。

**效益**：对本书而言，DGD 是第二个对照组，而且是更极端的那个：
CD 证明了「MudOS 的某些设计不是必然」，
**DGD 证明了「连 driver/mudlib 的分界线本身都可以重画」**——
把游戏导向的东西全部推上去，内核只留语言与持久化。

> 💡 君之一席话
> **技术领先十七年而生态没有跟上，往往不是技术的问题，是时机与兼容性的问题**——一个 2010 年才开源的 1993 年好东西，错过的不是市场，是整整一代人选型的那一刻。

> 🔍 老手视角──真正的门道
> DGD 的「原子函数」值得从一个更高的角度看：**它把一致性从一个纪律问题变成了一个机制问题。** MudOS 系要保证「改到一半不会留下烂摊子」，靠的是作者自己小心（先检查再改、失败要手动回复）；DGD 直接给你一个保证。这个转换在软件史上反复出现——内存管理从纪律（`free`）变成机制（GC）、数据一致性从纪律变成交易、资源释放从纪律变成 RAII／`defer`、并行安全从纪律变成类型系统（Rust 的所有权）。它们的共同模式是：**先有一类反复出现、靠纪律无法根治的 bug，然后有人把纪律编进机制里。** 老手看到一个团队反复在同一类 bug 上跌倒时，第一个念头不是「加强 code review」，而是「这个纪律能不能变成机制」——因为纪律的失效率不会归零，而机制的失效率可以。可落地的做法：把团队过去半年的事故分类，找出**同一种纪律失效超过三次**的那一类，然后问「有没有一个机制可以让它不可能发生」。

---

## 065　五支 driver 的技术对照 — 以及一份 CREDITS 的两个分支

**标签**：`#对照` `#设计空间` `#分歧` `#文献化石` `#选型`
**证据等级**：🟡 四份上游仓库与官方站的交叉比对 ＋ 🟢 本项目对 FluffOS 的实测

**起源**：§033–§035 排了时间轴，§063–§064 拆了 CD 与 DGD。
这一节把五支放在同一张表上——**不是为了排名，是为了看清楚设计空间有多大**。

**技术内核**：五支 driver 的分歧总表。

| 面向 | **LPMud**（1989） | **MudOS→FluffOS**（1992→今） | **LDMud**（1997） | **DGD**（1993） | **CD**（1991） |
|------|------------------|---------------------------|------------------|----------------|---------------|
| 血缘 | 本体 | 从 3.0.5x 分家，**明讲不兼容** | 接手 3.2（Amylaar 线） | **从零重写**（概念衍生） | 从 Lars 的 3.0 开发版分家 |
| 路线 | — | **batteries-included**（能力面最宽） | 官方线的现代化延续 | **极小内核**（游戏功能全推上层） | **服务单一世界**（Genesis） |
| 兼容对标 | — | MudOS | LPMud 3.2 | **LPMud 2.4.5 / 3.1.2** | Lars 的 3.0 |
| driver→mudlib 回呼 | apply | **apply**（字符串名，69 个） | apply | — | **hook**（`master.n` → `M_*` 常数） |
| 周期机制 | `call_out` + `heart_beat` | 同左（＋`gametick`） | 同左 | 任务模型 | **`set_alarm`（delay + repeat 合一，float 精度）** |
| 安全模型 | uid/euid | **uid/euid + 14 个 `valid_*`** | uid/euid | LPC 自己实作 | **`set_auth`（任意格式，master 可改写）** |
| 持久化 | `save_object` | `save_object`（＋swap） | `save_object` | **★ 整个世界是数据库，可快照可续跑** | `save_object` / `save_map` |
| 交易语意 | ❌ | ❌ | ❌ | **★ 原子函数（出错全回卷）** | ❌ |
| 对象种类 | 一种 | 一种（＋virtual object） | **＋轻量对象**（借自 DGD） | **★ 轻量 + 持久两种** | 一种 |
| 运行 | 解译 | 解译 | 解译 | **★ 字节码 + 选用 JIT** | 解译 |
| 热更新 | `update` | 重编对象（§060） | 同左 | **★ 就地重编 + hotboot 换运行档不断线** | 重编对象 |
| intermud | — | **mudlib 用 socket efun 自己写**（§049） | mudlib | LPC | **★ 做在 driver 里**（`udpsvc.c`/`tcpsvc.c`） |
| 字符串内嵌求值 | — | — | — | — | **★ VBFC（`process_string`/`process_value`）** |
| 现代化 | — | **C++、UTF-8/ICU、TLS、WebSocket、WASM、CI** | 持续维护 | C++ 重写（1.6）、JIT | libjson-c、siphash、bibopmalloc |
| 开源时间 | 一开始就是 | 一开始就是 | 一开始就是 | **2010-02（AGPL）** | 一开始就是 |
| 生态规模 | — | **最大**（本项目 17 份封存全在此系） | 中 | 小（但用在非 MUD 场景） | **小**（主要服务 Genesis） |

**从这张表可以读出三件事：**

### ① 五支解的其实是三个不同的问题

```
MudOS/FluffOS  →「怎么让最多人能开站」   答案：能力面推宽 + 死守兼容
LDMud          →「怎么把官方线接下去」   答案：稳定演进
CD             →「怎么把一个世界养好」   答案：为 Genesis 量身做决定
DGD            →「LPMud 这个概念的正确实作是什么」 答案：从零重写，内核极小
```

**它们不是同一场竞赛的参赛者。**
把它们放在一起比「谁比较好」没有意义；
有意义的是**看它们在同一个设计问题上做了什么不同的选择**——
那才是设计空间。

### ② 每一支都有一项别人没有的东西

| driver | 它独有的那一项 |
|--------|--------------|
| FluffOS | **WebAssembly**——整台 driver 进浏览器分页（第八篇整篇） |
| DGD | **原子函数 + 完整持久化**——一整类 bug 从可能变成不可能 |
| CD | **auth 可任意格式 + `valid_set_auth` 可改写**——授权策略完全外包 |
| LDMud | 官方线的连续性——**唯一能宣称「我是 LPMud 3.x」的那一支** |

**没有一支是另一支的超集。**

### ③ ★ 一份 CREDITS 的两个分支——文献化石

这是本节最有意思的发现，而且它是纯文献证据。

FluffOS 仓库封存的 `docs/archive/Credits.LPmud`（§033 引用过）
与 CD 仓库的 `doc/CREDITS`，**开头完全相同**：

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

**然后 FluffOS 那份就停了**（结尾是 Petri Wessman 修 SCO unix 与 AIX）。

**CD 那份继续往下写了十年**：

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

**同一份文件，在 1991／1992 年分家之后，两边各自长大。**

这件事的价值和 §049 那个 TMI-2 档头、§050 那个 `ken@XAJH` 署名是同一类：
**它是系统自己留下的、没有叙事动机的纪录**（附录 D.4 的判准）。
把两份 CREDITS 并排，分家点就在那个共同前缀的结尾——
**不需要任何人告诉你他们什么时候分开的。**

而且它还顺带证实了两件技术上的事：
- **`mapping` 是后来才加的**（Ronny Wikh 与 Lennart Augustsson 做了第一版）——
  所以 §052 那棵撑起整个游戏状态的属性树，用的是一个**不在最初设计里**的类型；
- **`switch` 与 smalloc 的 AVL 版都是 Joern Renecke 做的**，
  而 Joern Rennecke 的名字也出现在 FluffOS 那份的「找到并修掉大量 bug」名单里——
  **同一个人的贡献跨在分家的两侧。**

**解决的痛点**：这一节解的是选型与理解上的一个常见错误——
**把一个生态里最流行的实作当成那个概念的定义。**

如果只看 MudOS→FluffOS，很容易得出「LPMud 就是这样」的结论：
apply 是字符串、call_out 与 heart_beat 是两个东西、
安全模型是 uid/euid、intermud 是 mudlib 的事。
**五支并排之后，这四条全部变成「MudOS 的选择」，而不是「LPMud 的本质」。**

**优点 / 罩门**：这种横向对照的优点是它会**把隐含假设逼出来**。
罩门是它需要真的去读别人的实作——
本节的每一格都来自实际的仓库与文档，而不是从一支推论其他四支。

**效益**：对本书的收束意义：第八篇问「连接的另一端是什么」，
第九、十篇往下钻了三层，而这一节是最后一次拉远——
**你手上那台 driver 的每一个设计，都有至少一个活着的反例。**

而这正是本书从第一页就在讲的那件事的另一种说法：

> **知道一个东西「是什么」，和知道它「为什么不是别的样子」，是两种不同的理解。**
> 前者让你会用，后者才让你知道哪些东西可以改。

> 💡 君之一席话
> **不要把生态里最流行的实作当成那个概念的定义**——找到那些做了相反选择而且活着的实作，你才会知道哪些是本质、哪些只是某个人在 1992 年的一个决定。

> 🔍 老手视角──真正的门道
> 这一节示范的方法叫**横向对照**，它的成本很低但几乎没有人做。多数人理解一个技术的路径是「读最流行那个的文档 → 读它的原代码 → 觉得自己懂了」，而这条路径有一个结构性的盲点：**你学到的每一个设计决定，都不知道它有没有替代方案。** 修正的方法只有一个——**去读那个做了相反选择的实作**，哪怕只读它的 API 文档与目录结构（本节 CD 的部分几乎全部来自 `doc/` 目录与文件清单，没读一行 C）。半天的阅读量，能把「这就是唯一做法」变成「这是三种做法之一，而它们的取舍是……」。可落地的做法：学任何技术时，**同时打开它的一个竞争者的文档目录**，只比对两件事——**名词表**与**目录结构**。名词不同的地方就是概念分歧点，目录不同的地方就是架构分歧点；这两张图比任何一份 tutorial 都更快让你看见设计空间。
