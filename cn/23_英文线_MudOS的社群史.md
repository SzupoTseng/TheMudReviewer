# 第十二篇　英文线：MudOS 的社群史

> 附录 D.3 记了中文线：东方故事（1993）→ 侠客行（1995）→ 外流（1996）→ 本项目的 17 份封存。
> **但那条线是从 TMI-2 mudlib 分出去的支流。** 主流在英文世界，而书里一直没写。
>
> 这一篇补上 1992–2010 年代的英文 MudOS 社群史，四个单元：
> **一个为了写 driver 而开的 MUD、三份 mudlib 的竞赛、
> MudOS 的三次死而复生、以及 FluffOS 的建国宣言。**
>
> 它回答一个本书一直没问的问题：
> **本项目手上那台 driver，中间经过了哪些人的手？**

---

## 071　TMI — 一个为了写 driver 而开的 MUD

**标签**：`#TMI` `#MudOS` `#社群史` `#intermud起源` `#一手史料`
**证据等级**：🟡 一手史料（`Credits.MudOS`、`History.MudOS`）＋ 🟡 George Reese（Descartes）1995 年整理的 LPMud 编年
　　　　　　　※ 编年的作者本人列名于 `Credits.MudOS`——**他是参与者，不是旁观者**。这既是它可信的理由，也是它有立场的理由。

**起源**：§034 引用了 1992 年那篇 Usenet 贴文，讲 MudOS 为什么分家。
但贴文里有一个缩写没有解释，而它是整条英文线的起点：

> Originally, people at **TMI** got together and asked everyone who was
> modifying their driver to get together and discuss bringing all of those
> changes together.

**TMI = The MUD Institute。**

**技术内核**：TMI 是一个很不寻常的东西——**一个为了开发软件而开的 MUD**。

> **1992 年 2 月**：_TMI, The MUD Institute_ 对公众开放，
> 作为一个**「用来开发新的 LPMud server 与 mudlib」的 MUD**。
> 除此之外，TMI 还立了一个章程：**教 LPC**。

它同时是三样东西：一个游戏、一个开发项目、一所学校。
**而这个组合直接决定了 MudOS 的性格。**

### 时间轴：TMI 的十八个月

| 日期 | 事件 |
|------|------|
| **1992-02** | TMI 开放。目标：新的 server + 新的 mudlib + 教 LPC |
| **1992-02-18** | **LPMud 3.1.2-A 项目改名为 MudOS** ← §034 那场改名的**精确日期** |
| **1992-04-23** | **LPC socket 加入 MudOS driver**。TMI 据此做出一个很粗糙的 TCP intermud 网络 |
| 1992-06 | Blackthorn 把 Genocide 搬到新的 MudOS driver。「当时 driver 充满新功能，也**同样充满 bug**」——Genocide 整个夏天成了 MudOS 的测试场 |
| 1992-07 | Descartes 接手 Nightmare 的 mudlib，**丢掉它那份过时的 LPMud 2.4.5 mudlib 与 driver**，改用 MudOS |
| **1992-08** | **TMI 关站。** |
| 1992-08 | **TMI-2 开放**——目标更窄，**放弃了教学的章程** |

**三件事值得停下来看：**

### ① 改名的日期，补上了 §034 的空缺

§034 引的那篇贴文写于 1992-08-28，讲的是**改名之后**的争议。
现在知道改名发生在 **1992-02-18**——**骂战是在半年后才爆发的**。

这个时间差本身就说明了问题：改名当下没有人在意，
是等到「用我们 driver 的人一直在问为什么 compat mode 不能用」（§034）
累积了半年，事情才炸开。**兼容性的争议从来不是在改动当下发生的。**

### ② ★ 1992-04-23：本项目那 15 份封存里的 intermud，源头在这里

> LPC sockets are added to the MudOS driver. **This allows TMI to create a
> very rough TCP intermud network.** This protocol is later replaced first by
> the **CDlib UDP protocols**, and later by **Intermud 3**.

把这一行和 §049 对起来看：

```
1992-04-23  MudOS 加入 socket efun
     ↓      （§047 说的「引擎加的不是功能，是可能性的边界」）
1992       TMI 用 LPC 写出第一版 TCP intermud
     ↓
1993-08-15 Grendel@Tmi-2 写下 tcp.c 的档头
     ↓      「This file is part of the tmi mudlib. Please keep this header intact.」
1993 年底   东方故事采用 MudOS + TMI mudlib（附录 D.3）
     ↓
2026       🟢 本项目 17 份封存中的 15 份，仍带着那批文件
```

**§049 那组 `_q`／`_a` 成对的服务、`dns_master` 名录、
以及 `nitan7` 封存里那份 118 笔的中文 MUD 名录快照（附录 D.4）——
它们的祖先是 1992 年 4 月 TMI 那个「很粗糙的 TCP intermud」。**

而那一行还记了它后来被取代两次：先是 **CDlib 的 UDP 协议**
（就是 §063 那支 CD driver 的 mudlib——**它把 intermud 做进了 driver**），
再来是 **Intermud 3**。

**中文线接手的是 TMI 那一版**，而且**再也没有升级**——
所以三十三年后，本项目在封存里挖到的是 1993 年的实作。

### ③ 「为了写软件而开的 MUD」是那个年代的协作方式

没有 GitHub、没有 CI、没有 issue tracker。
`Credits.MudOS` 的致谢名单里列的不是贡献者，是**测试场**：

> We would like to thank the following muds for extensive testing and
> numerous bug reports:
> **TMI, Portals, Overdrive, Genocide, TMI-2, Vincent's Hollow,
> DreamShadow, Nightmare, Nanvaent**, and many others.

**九个 MUD 就是九个生产环境的测试集群。**
「driver 有 bug」的回报路径是：玩家在 Genocide 掉线 →
管理者 Blackthorn 回报 → Truilkan／Jacques／Wayfarer 在 Portals 上改 code。

而那个「开发站」是谁提供的，写在同一份文件的最后一行：

> Thanks to **Adam Beeman (Buddha)** for making numerous suggestions on
> improving the MudOS driver and **for providing the TMI site as the original
> MudOS development site**.

**解决的痛点**：TMI 解的是 1992 年一个非常实际的问题——
**LPMud 的 driver 由一个人维护，而那个人很忙**（§034）。
社群的答案不是 fork 一份 code，是**开一个地方让大家在里面一起改**。

**踩过的坑**：TMI 开了六个月就关站。
§034 那篇贴文的自述解释了为什么：

> About 10 people showed up and we all talked for a while. For a month or two,
> we had a bulletin board where all of us posted ideas / suggestions / etc.
> **However, nobody really had the time to get together and work on the code.**

**开会的人很多，写 code 的只有两个。**
TMI-2 接手时明确**放弃了教学章程**——把目标收窄到只做 mudlib。

**优点 / 罩门**：优点是它真的产出了东西——MudOS、TMI-2 mudlib、
以及第一版 intermud，三样都活了三十年以上。
罩门是**它把「社群治理」与「软件开发」绑在一起**，
而这两件事的节奏完全不同：治理需要共识，开发需要有人动手。
TMI 死于前者，MudOS 活下来是因为后者被少数几个人扛走了。

**效益**：对本项目而言，这一节解释了一件事——
**为什么本项目手上的 mudlib 带着一批 1993 年、来自一个已经关站的 MUD 的文件。**
不是因为谁偷懒没更新，是因为**那批文件在它被拷贝出去的那一刻，就已经是孤儿**：
TMI 1992 年就关了，TMI-2 的 mudlib 1993 年发布，
而中文线 1993 年底接手——**接手时上游已经不在了。**

> 💡 君之一席话
> **一个为了开发软件而开的社群，最后留下来的往往不是社群，是软件**——TMI 存在了六个月，而它写的那批文件还在三十三年后的封存里。

> 🔍 老手视角──真正的门道
> 「用九个生产环境当测试集群」这件事，在今天看起来很原始，但它有一个现代 CI 给不了的性质：**那些 bug 是真的玩家在真的世界里踩出来的**。Genocide 整个夏天当测试场，回报的不是「单元测试失败」，是「三十个人同时上线的时候崩了」。老手都知道这类 bug 最难用测试覆盖——它们是负载、时序、状态累积的交互作用。今天的对应物是金丝雀部署、影子流量、feature flag 逐步放量，而它们解决的是同一个问题：**在真实环境里取得早期信号，但不要一次赔上全部用户**。1992 年的做法是靠「有几个管理者愿意当白老鼠」，2026 年的做法是靠流量切分——**机制变了，原理没变**。可落地的判准：如果你的系统只在 CI 里被验证过，那你其实只验证了你想得到的那些情况；**永远要有一个「真实但可牺牲」的环境**。

---

## 072　三份 mudlib 的竞赛 — 而竞赛的主轴是指令解析器

**标签**：`#mudlib` `#Nightmare` `#Discworld` `#TMI-2` `#LIMA` `#指令解析`
**证据等级**：🟡 George Reese 的 LPMud 编年 ＋ 🟢 本项目封存的实测对照

**起源**：§050 用文件名集合量出本项目的 17 份封存来自两个家族，
而两族的共同祖先是 **TMI-2**。
但 TMI-2 在英文世界只是**三份之一**——而且不是最强的那一份。

**技术内核**：MudOS 的 mudlib 生态，1992–1995 的四年。

| 日期 | mudlib | 定位 |
|------|--------|------|
| **1992-12** | **Nightmare Mudlib**（Descartes） | **第一份公开可用的 MudOS mudlib**。当时 MudOS「在众多 driver 中还被视为新来的」 |
| **1993-03** | **Discworld Mudlib** | 「**当时最先进的指令解析器与用户接口**」——mudlib 的选择帮助推高了 MudOS 的人气 |
| **1993-04** | **TMI-2 Mudlib** | 「让 MudOS 拥有三份广泛可得的 mudlib」 ← **🟢 本项目 15/17 份封存的祖先** |
| 1995-05-15 | **Foundation II** | **第一份为非游戏用途设计的 LPC 函数库**；**第一份可商业授权的 MudOS mudlib** |
| 1995-06-15 | **Nightmare IV** | 完全重写，**清掉一路追溯到 LPMud 2.4.5 的遗留代码** |
| 1995-07-21 | **LIMA**（pre-alpha） | 「**LPMud 上见过最先进的指令解析**」，**基于老 Infocom 游戏（Zork）的解析方式** |

另外还有两本教科书，说明这个生态已经大到需要教材：

| 日期 | 书 |
|------|---|
| 1993-04 | **LPC Basics** —— 第一本涵盖 LPC 所有实作的完整教科书 |
| 1993-11 | **Intermediate LPC** |

**★ 这张表最重要的消息在「先进」这个词指的是什么。**

Discworld（1993）与 LIMA（1995）被称赞的都是**指令解析器**。
这不是巧合——**在文本游戏里，指令解析器就是用户接口。**

```
玩家打：  get the red sword from the wooden chest
          └───────────── 这一行要被拆成 动词／限定词／对象／介系词／容器
```

MudOS driver 有 `parse_command()` efun（§047 那张表里的「自然语言式指令」），
但**它只是机制**——把它变成「玩家可以自然地说话」需要 mudlib 做非常多任务作。

**LIMA 直接对标 Infocom 的 Zork**，那是文本冒险解析的天花板。

> 🔬 **本书的角度**：这正好是 §054 那条线的另一端。
> §054 说 MUD 客户端三十年的四件套（trigger／alias／macro／script）
> 是因为「服务器只吐文本」；
> **这里看到的是同一个限制在服务器那一侧的形状——
> 既然只有文本，那就把文本的「输入」与「输出」两端都做到极致。**
>
> 英文 mudlib 把力气花在**输入端**（指令解析器）；
> 而 §043 那套 ZJMUD 方言把力气花在**输出端**（把状态明讲）。
> **两条路线都是对「纯文本」这个前提的回应，只是选了不同的那一端。**

**Discworld 这条线特别值得标记**，因为它会在 §074 再出现一次：
1993 年做出最先进解析器的那份 mudlib，
它的内核开发者 **David Bennett（Pinkfish）** 同时在改 MudOS driver 本身——
`Credits.MudOS` 列他的贡献是：

> provided enhancements for **parse_command** and for the **telnet negotiation
> code in comm.c**, and the **resolve()** efun.

**mudlib 的需求直接推动了 driver 的功能。**
Discworld 要更好的解析器，于是 `parse_command` 被加强；
要更好的终端支持，于是 telnet 协商被改写——
**而本书 §055 量到的那些协商字节，走的就是那段被 Pinkfish 改过的代码的后裔。**

**解决的痛点**：三份 mudlib 并存解的是 §034 没解的那个问题——
**「拿一份现成的东西开站」**。
MudOS 在 1992 年被视为「新来的」，
而它在 1993 年变得流行，靠的不是 driver 本身变好，
是**同时有三份可用的 mudlib**。

**踩过的坑（结构性的）**：三份并存的代价是**分裂**。
Nightmare、Discworld、TMI-2 的 API 彼此不兼容，
一份为 Nightmare 写的区域搬不到 Discworld。
§050 量到的那个现象（两个家族、Jaccard 0.012）在英文世界是三倍严重。

而且——**中文线在 1993 年接走了 TMI-2 那一份之后，
就与后面的所有发展断了联系**：
Foundation II（1995）、Nightmare IV（1995）、LIMA（1995）
在本项目的 17 份封存里**一个痕迹都没有**。

**优点 / 罩门**：优点是竞争真的推动了技术——
1993 到 1995 年，MudOS mudlib 的指令解析从「动词＋名词」
走到了 Infocom 等级。
罩门是**每一份 mudlib 都是一个孤岛**，而孤岛越多，
单一孤岛的维护人力越少。

**效益**：对本项目而言，这一节校正了一个容易产生的误解。
§049–§053 把 TMI-2 的架构讲得很细（因为那是本项目手上的东西），
但**TMI-2 从来不是英文世界的主流选择**——
它是三份之一，而且不是解析器最强的那一份。

**本项目的 17 份封存继承的，是 1993 年那个生态的一个切片，
而不是那个生态的最终状态。**
这件事在做任何「中文 MUD 的技术水准如何」的判断时都必须先说清楚。

> 💡 君之一席话
> **一个平台流行起来，通常不是因为它自己变好了，是因为它上面终于有了三个可以直接拿来用的东西。**

> 🔍 老手视角──真正的门道
> 「mudlib 的需求推动 driver 的功能」这个方向值得注意，因为它和多数人的直觉相反：**不是引擎先提供能力、上层才长出应用，而是上层先痛、再回头改引擎。** Pinkfish 为了 Discworld 的解析器去改 MudOS 的 `parse_command`，为了终端体验去改 `comm.c` 的 telnet 协商——他是用户，也是贡献者。这种「用户即贡献者」的回路是开源生态最健康的形态，而它有一个前提：**引擎与应用的作者要在同一个沟通半径内**。TMI 那种「开一个 MUD 让大家在里面一起改」的做法，本质上就是在人为制造这个半径。老手评估一个开源项目的健康度时，看的不是 star 数，而是**「最近半年的 commit 里，有多少来自真的在用它的人」**——全部来自内核维护者的项目，通常已经在解决想像中的问题了。可落地的做法：如果你在做平台，**想办法让自己成为它的一个真实用户**；做不到的话，至少要有一条让真实用户的痛点在两周内传到你手上的信道。

---

## 073　MudOS 的三次死而复生

**标签**：`#MudOS` `#BeekOS` `#LPC到C` `#维护者交接` `#compat_buster`
**证据等级**：🟡 George Reese 编年 ＋ 🟡 一手史料（`Credits.MudOS`、`README.MudOS`、`ChangeLog.mudos` 2,217 行）

**起源**：本书从 §034 之后就一直把 MudOS 当成一条连续的线在讲。
**它不是。** 它至少停过三次，而且有两次差点就没有下文了。

**技术内核**：

### 第一次：1993 年底 —— 「MudOS 正在死去」

> **1993 年 11 月**：由于 LPMud 的速度、DGD 的优雅，
> 以及**MudOS 这边单纯缺乏进展**，MudOS 正在死去。
> **Beek 于是做了 BeekOS。**

BeekOS 是什么：

> BeekOS 基本上是一个 MudOS 内核，加上**LPC→C 的动态编译**，
> 把编译出来的机器码**动态链接进正在运行的 server**。

**这是 §037 那个「纯解译堆栈机的性能天花板」的第一次突围尝试。**
本书 §037 说过：「所有性能都靠 efun（把热点推回 C++），
这正是为什么一个 MUD driver 会有 600 多个内置函数」。
**Beek 的答案是：那就把 LPC 本身编成 C。**

值得注意的是**时序**：DGD 在 **1993-09-16** 的 1.0.a4 版
成为第一个支持 LPC→C 编译的 driver；
**两个月后 Beek 就在 MudOS 上做出了对应的东西。**
编年里明说 BeekOS 的动机之一就是「DGD 的优雅」——
**竞争产生的技术压力，在这里有明确的日期可以对上。**

这些改动后来在 Beek 接手 MudOS 之后被并回主线。

### 第二次：1994 年初 —— 唯一的维护者也停手了

> **1994 年初**：Genocide 为了速度转回 LPMud。
> 结果 **Blackthorn 停止了那涓滴般的 bug 修复，
> 而那正是当时 MudOS 开发的全部。**

**「that trickle of bug-fixes which had been the whole of MudOS development」**——
一句话说完那个时期的状态：整个 MudOS 的开发，
就是一个人在他自己的 MUD 需要时顺手修几个 bug。
他的 MUD 一换 driver，**开发就归零**。

### 第三次：1994 年中 —— 三个人接手

> **1994-06-26**：**Beek、Robocoder、Symmetry 接手 MudOS 开发。**
> 这标志着在**将近九个月的无开发状态**之后，
> MudOS 重新引起兴趣。

`Credits.MudOS` 对这三个人的记载，正好对应他们接手后做的事：

| 人 | 贡献（`Credits.MudOS` 原文） |
|----|---------------------------|
| **Tim Hollebeek (Beek)** | **LPC→C and patches, compiler rewrite, new function pointers, etc** |
| **Anthon Pang (Robocoder)** | various patches, **optimize function table search**, amiga support |
| **Ti-Ming Chiang (Symmetry, Cloud)** | **Countless optimizations; array, mapping and save_object rewrites** |

**三个人分别接手了编译器、函数表、与数据结构**——
这是一次很典型的「重写内核」分工。

**Beek 那一行的 “compiler rewrite” 与 “new function pointers”
就是今天 FluffOS 里那个编译器与 `(: :)` 语法的来源。**

### 版本编号与「compat buster」

`README.MudOS` 记下了 MudOS 的发行惯例，而其中一条很值得学：

> In particular, note the changes marked **"compat buster"** since these
> describe changes to the driver which **may cause pre-existing mudlibs
> (such as Lil) to break** in some minor fashion (the ChangeLog should give
> enough information to allow affected mudlibs to be fixed).

**他们在 ChangeLog 里替破坏性变更加了一个标签。**

对照 §059 讲 FluffOS 拆 `static` 时说的「**可侦测的破坏性变更**」——
MudOS 在 1990 年代用的是更原始但同样有效的手段：
**在人类可读的变更纪录里，把会弄坏你的那几行标出来。**

其他发行惯例：

| 惯例 | 内容 |
|------|------|
| 版号 | `v20` / `v21` / `v22`（单一数字＝含文档的完整包）；`v20.26`（小版号＝只有原代码） |
| alpha/beta | **只以 patch 形式提供**，且有独立的 `ChangeLog.alpha` / `ChangeLog.beta` |
| COMPAT | **`README.MudOS` 开宗明义：「It does not support COMPAT mode」**——§034 那个不兼容声明，写在用户第一眼会看到的地方 |
| 支持站 | `http://www.mudos.org/`，含 bug 数据库；当机时先读 `IT_CRASHED` 收集信息 |

### 移植狂潮：MudOS 被搬上了所有东西

`Credits.MudOS` 里有一整批贡献者的功劳是**移植**，
把它们排出来就是一份 1990 年代的操作系统清单：

| 平台 | 谁 |
|------|-----|
| AIX 3.2 | Dwayne Fontenot (Jacques) |
| Sequent Symmetry（System V R3） | Dave Richards (Cynosure/Cygnus) |
| System V R4 | Michael Bundy |
| 386BSD | Luke Mewborn (Zak) |
| **Linux 0.99.3** | Olav Kolbu (Aragorn) |
| DEC Alpha | Bob Farmer (Blackthorn) |
| Amiga | John Fehr (Wildcard)、Anthon Pang、Maarten de Jong |
| Win32（原始版） | **George Reese (Descartes)** |

**`Linux 0.99.3`** 那一行是个时间戳：那是 1993 年的内核版本。
**MudOS 在 Linux 1.0 出来之前就跑在上面了。**

而 §035 那张「FluffOS 支持 Linux／macOS／Windows／Alpine／WebAssembly」的表，
其实是这个传统的延续——**这条血脉从一开始就有很强的移植文化**。

**解决的痛点**：这三次死而复生解的其实是同一个问题——
**开源项目的维护者风险（bus factor）**。

三次的模式一模一样：

```
维护只靠一个人（或一个 MUD 的需求）
   → 那个人 / 那个 MUD 的处境改变
     → 开发归零
       → 有人受不了，自己做一个分支（BeekOS）
         → 分支被并回，做分支的人成为新维护者
```

**而 §074 会看到，这个循环在十年后又跑了一次。**

**踩过的坑**：`ChangeLog.mudos` 有 2,217 行，
而它的开头第一句话很坦白：

> This file is the ***brief*** version of what has been changed;
> see `ChangeLog.alpha`, `ChangeLog.beta`, and `ChangeLog.old/`
> for all the gory details.

**这是「简短版」。** 一个没有 issue tracker、没有 PR、
没有自动化测试的年代，变更纪录就是项目的全部记忆——
所以它必须非常详细，而详细到 2,217 行还只是简短版。

**优点 / 罩门**：优点是这条线**每次都有人接住**。
罩门是每次接住的人都不一样，而**每一次交接都伴随一次内核重写**
（BeekOS 的 LPC→C、Beek 的 compiler rewrite、Symmetry 的 array/mapping/save_object 重写）。
§035 说 FluffOS 承诺与 MudOS 兼容——
**而 MudOS 自己在 1994 年就把编译器、数组、mapping、存盘全部重写过一轮。**

**效益**：对本书而言，这一节补上了 §035 那张血脉表看不到的东西：
**一条线「还活着」不代表它一直有人在推。**
MudOS 从 1992 到 1997 这五年里，至少有九个月是完全停摆的，
而它能撑过去，靠的是三个人在 1994 年 6 月 26 日那天决定接手。

> 💡 君之一席话
> **一个开源项目的健康度，不看它有多少功能，看它换过几次维护者而还活着**——换得过去的，才证明它不是某一个人的私有物。

> 🔍 老手视角──真正的门道
> BeekOS 这个模式很值得单独记：**当上游停滞而你有真实需求时，正确的做法往往不是等，也不是抱怨，是做一个分支证明它可行**——然后那个分支通常会被并回，而你会成为维护者。这在开源史上反复出现（io.js 之于 Node、Blink 之于 WebKit、本书 §074 的 FluffOS 之于 MudOS），而它成功的关键条件只有一个：**分支必须解决一个真实且可展示的问题**。BeekOS 解的是「太慢」，而它拿得出 LPC→C 的实作；纯粹因为「上游不理我」而开的分支，几乎都会死。老手看到自己在等上游时，会问三个问题：**这个需求真实吗？我做得出来吗？做出来之后上游有理由不要吗？**三个都是 yes，就该动手了——而且要从一开始就以「将来要被并回」的方式写。可落地的做法：分支时**保持与上游的可合并性**（不要顺手重排目录、不要换 code style），你的分支能不能回家，八成取决于这一点。

---

## 074　FluffOS 的建国宣言 — 以及 Discworld 这条暗线

**标签**：`#FluffOS` `#分支` `#Discworld` `#Pinkfish` `#模式重演`
**证据等级**：🟡 一手史料（`ChangeLog.fluffos` 全文与 `Credits.MudOS`）

**起源**：本项目用的 driver 叫 FluffOS。
它是怎么从 MudOS 变成 FluffOS 的？答案在 `ChangeLog.fluffos` 的**第一行**——
而那一行是一句宣言：

> **As MudOS is moving too slow to keep our driver hacks apart,
> we now call our own FluffOS :)**

**技术内核**：拆开这一句话，四个信息全在里面。

### ① 「MudOS is moving too slow」

**和 1993 年 Beek 做 BeekOS 的理由一模一样**（§073：
「simple lack of progress on the part of MudOS」）。

**同一个项目、同一个原因、隔了大约十年，发生了第二次。**

### ② 「our driver hacks」——他们早就在改了

分支不是决定，是**追认**。
他们手上已经有一堆改动，而维护「我们的改动」与「上游」之间的差异
（keep them apart）成本越来越高，于是干脆给它一个名字。

**这与 §034 MudOS 自己的分家理由结构相同**：
1992 年 TMI 那批人的版号变成 `3.0.53.A2.2`，
维护「挂在别人版号后面的自己的线」也是一样的成本问题。

**两次分家，同一个机制：差异的维护成本超过了分家的社交成本。**

### ③ 「we now call our own」——先改名，再谈其他

§034 的老手视角说过：MudOS 的沟通顺序是**先改版号（解决自己的痛）、
再改名（解决用户的误解）、最后才解释动机**。

FluffOS 这一句把前两步压成了一步，而且**连解释都省了**——
只有一个笑脸。

### ④ `:)` ——这是一份 ChangeLog，不是一份公告

没有 RFC、没有邮件列表投票、没有基金会。
**一个人在变更纪录的最上面加了一行。**

### FluffOS 1.0 的内容，以及它是谁做的

ChangeLog 的最末端是它的起点：

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

四个消息：

| 观察 | 意义 |
|------|------|
| **「changes since MudOS v22.2b11」** | 分支点是 MudOS 的一个 **beta** 版——不是稳定版 |
| **「as far as i can remember」** | **他自己也不确定改了什么**——这正是「差异维护成本过高」的症状 |
| **eval limit 改成基于时间** | §037 那个 eval cost 沙盒的重做（后来 FluffOS 1.31 又改用 signals） |
| **署名 `Wodan.`** | 一个人 |

### ★ 暗线：Discworld

`ChangeLog.fluffos` 里有一行看起来只是普通的 bug 修复：

> **fixes to compile with other settings than the DW ones**（FluffOS 1.28）

**`DW` = Discworld。**
「让它能用 Discworld 以外的设置编译」——
**意思是在那之前，FluffOS 的缺省设置就是 Discworld 的设置。**

再往下看贡献者名字：

```
FluffOS 1.4:  added compressedp efun, it shows if an interactive uses MCCP (pinkfish)
FluffOS 1.5:  fixed crasher after errors in save_object. (reported by pinkfish)
FluffOS 1.28: terminal_colour fix (pinkfish)
```

**Pinkfish。**

翻回 §072 引的 `Credits.MudOS`：

> **David Bennett (Pinkfish)** - provided enhancements for **parse_command**
> and for the **telnet negotiation code in comm.c**, and the **resolve()** efun.

**同一个人，出现在 1990 年代的 MudOS 贡献者名单，
也出现在 2000 年代的 FluffOS 变更纪录里。**

而 §049 还记得第三次出现：本项目的封存里，
`adm/daemons/network/` 底下有 **11 个文件署名 `Creator : Pinkfish@Discworld`**。

**于是这条暗线完整了：**

```
Discworld MUD（1992 开站）
   │
   ├─ 1993-03  发布 Discworld Mudlib：「当时最先进的指令解析器与 UI」（§072）
   │
   ├─ 1990s    Pinkfish 改 MudOS 的 parse_command、telnet 协商、resolve()
   │              ↳ 🟢 §055 那 6 组实测协商字节，走的是这段代码的后裔
   │
   ├─ 1993     Pinkfish 的 11 个文件进了 TMI-2 的 network/ 目录
   │              ↳ 🟢 本项目 15/17 份中文封存至今带着它们（§049）
   │
   └─ 2000s    FluffOS 从 Discworld 的 driver hacks 诞生
                  ↳ 🟢 本项目今天用的就是它
```

**本项目手上这台 driver，与封存里那 11 个文件，来自同一个 MUD。**
而中文线在 1993 年接走 TMI-2 的时候，
**根本不知道它同时也接走了 Discworld 的一部分。**

### 之后：从一个人的 hack 到现代 C++ 项目

`ChangeLog.fluffos` 记录的 1.0–1.36 是「一个人修 crasher」的阶段。
今天的 FluffOS（§035）是 C++、UTF-8/ICU、64 比特、TLS/WebSocket、
GitHub Actions CI、跨五个平台含 WebAssembly——
**中间又换过一轮维护者**，而这正是 §073 那个循环的第四次。

**解决的痛点**：这一节解的是本书一个很基本、却一直没回答的问题——
**「我们用的这台 driver，是谁做的？」**

答案是：**没有「谁」。**
1989 年 Lars 写了内核；1992 年 TMI 那批人把它变成 MudOS；
1994 年 Beek 三人重写了编译器与数据结构；
2000 年代 Wodan 从 Discworld 的 hacks 里分出 FluffOS；
之后又有人把它现代化到今天。

**每一段都是「上一个人停下来了，下一个人接手」。**

**踩过的坑（模式层面）**：这条线三十七年里至少分家或停摆五次
（1991 CD、1992 MudOS、1993 BeekOS、1994 停摆九个月、2000s FluffOS），
**而每一次的触发条件都相同：上游太慢，而下游有真实需求。**

§073 的老手视角给了那三个问题（需求真实吗／做得出来吗／上游有理由不要吗），
**FluffOS 三个都是 yes，所以它活了下来。**

**优点 / 罩门**：优点是这个模式让一个没有公司、没有基金会、
没有资金的项目活了三十七年。
罩门是**每一次交接都伴随知识流失**——
「as far as i can remember」这句话就是证据，
而 §067 提到 LDMud 的官方文档承认 ERQ 是「逆向出来的」，
是同一个病在另一条在线的症状。

**效益**：对本书的收束意义——

本书从第一页讲的就是**逆向工程**：
面对一个没有规格书的黑箱，怎么把猜测逼成证据。

而这一节说明了一件事：**那个黑箱不是别人恶意造成的。**
它是三十七年、五次交接、无数次「as far as i can remember」累积出来的。
本项目花在还原 ZJMUD 协议上的力气，
与 LDMud 维护者逆向 Amylaar 的 ERQ、
与 FluffOS 维护者搞清楚 MudOS v22.2b11 到底改了什么，
**是同一件事。**

> 💡 君之一席话
> **「as far as i can remember」是每一个长寿项目最诚实的一行文档**——它承认了知识已经流失，而承认流失，是重新把它找回来的第一步。

> 🔍 老手视角──真正的门道
> 这一篇四个单元排在一起，会看到一个很清楚的循环：**创建社群 → 社群解散但软件活着 → 维护者疲乏 → 有人分支 → 分支成为主线 → 回到第一步。** 三十七年跑了至少五轮。老手在评估一个要依赖很久的开源项目时，看的不是它现在有多活跃，而是**它有没有跑完过这个循环至少一次**——因为第一次交接是最难的，跨过去的项目证明了它的价值不绑在任何一个人身上。反过来说，一个从未换过维护者的项目，无论多活跃，它的 bus factor 都是 1，而你看不出来。可落地的判准：查一个项目**最早的三个 commit 作者是否还在提交**——都在的，要问「如果他们走了会怎样」；都不在而项目还在长，那它已经证明过自己了。**这是唯一一个不用读 code 就能做的架构风险评估。**
