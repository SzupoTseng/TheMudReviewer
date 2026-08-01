## 036　三个方向的调用 — efun、apply 与 simul_efun

**标签**：`#FluffOS` `#driver` `#efun` `#apply` `#simul_efun` `#边界`
**证据等级**：🟡 FluffOS 官方文档（`docs/concepts/general/*`、`docs/apply/*`）

**起源**：读 mudlib 原代码时，最先卡住人的不是 LPC 语法，是**一个函数到底是谁在调用谁**。
`write()` 是谁？`create()` 是谁调用的？`is_chinese()` 又是哪来的？
这三个问题的答案分别是三种不同的机制，而它们的**方向不一样**。

**技术内核**：FluffOS 的三层模型，以及层与层之间的三条调用路径。

```
        ┌───────────────────────────────────────────┐
        │   mudlib（LPC 文件：房间、NPC、登录流程）      │
        └───────────────────────────────────────────┘
              │  ①efun          ↑ ②apply       ↺ ③simul_efun
              │  往下调用         │ 往上回呼        （同层，绕道 simul_efun 对象）
              ▼                 │
        ┌───────────────────────────────────────────┐
        │   driver（C++：LPC 编译器 + 字节码 VM      │
        │           + 网络 + 600 多个 efun）           │
        └───────────────────────────────────────────┘
```

| # | 机制 | 方向 | 谁定义 | 例子 | 找不到时 |
|---|------|------|--------|------|---------|
| ① | **efun** | mudlib → driver | driver（C++） | `write()`、`clone_object()`、`crypt()`、`set_encoding()` | 编译错误 |
| ② | **apply** | driver → mudlib | mudlib（LPC） | `create()`、`init()`、`heart_beat()`、`logon()`、`connect()` | **静默略过**（没定义就是不做） |
| ③ | **simul_efun** | mudlib → mudlib | mudlib（单一对象） | `is_chinese()`、mudlib 自订的一切工具函数 | 编译错误 |

**② 是最容易误解的一个**。apply 不是「你去调用 driver」，是「**driver 在特定时刻来调用你**」。
它没有注册步骤、没有接口声明——**你只要把函数名字写对，driver 就会找到它**。
反过来说，名字打错就是静默失效：driver 找不到 `heart_beat`，
不会报错，只会安静地什么都不做。

这个「静默」的性质，和本书 §015 的七次事故是同一个家族的东西。
apply 的失效不会有例外、不会有 log、不会有红字——
**它只是不发生**。这是读 LPC mudlib 最需要创建的直觉。

**三类 apply，按调用时机分**：

| 类别 | 定义在哪个对象 | 典型成员 | 何时被调用 |
|------|--------------|---------|-----------|
| **object applies** | 每个一般对象 | `create` `init` `reset` `heart_beat` `clean_up` `id` `move_or_destruct` `on_destruct` `virtual_start` | 对象生命周期事件 |
| **master applies** | master 对象（本项目是 `/adm/single/master`） | `connect` `logon` `error_handler` `crash` `preload` `compile_object` `valid_read` `valid_write` `valid_shadow` `valid_override` `valid_seteuid` `get_root_uid` `include_file` `inherit_program` | 全域事件、安全决策、编译决策 |
| **interactive applies** | 玩家连接所附着的对象 | `logon` `net_dead` `process_input` `write_prompt` `terminal_type` `window_size` `telnet_suboption` `catch_tell` `receive_message` `gmcp` `msdp` `mxp_tag` `zmp` | 该连接的网络事件 |

第三列请特别看一眼：**`gmcp` / `msdp` / `mxp_tag` / `zmp` 是 driver 提供的 apply**，
也就是说这四套 MUD 扩充协议 **driver 是知情的**——它会做 telnet 子协商、
把解出来的数据交给 mudlib。
**而 ZJMUD 不在这张表上。** 这件事的全部后果，§043 会展开。

**① efun 的规模**：FluffOS 官方 README 的说法是「600 多个内置函数」，
按主题分成套件（package）：`core`、`math`、`matrix`、`parser`、`sockets`、`db`、
`crypto`、`compress`、`pcre`、`async`、`external`、`ffi`、`uids`、`mudlib_stats`……
套件是**编译期开关**——关掉的套件，它的 efun 就**不存在**，
mudlib 用到就是编译错误。§041 会看到这件事在 WebAssembly 版上的实际形状。

**③ simul_efun 的门道**：mudlib 可以定义一个对象（本项目是 `/adm/single/simul_efun`），
里面的 **public 函数会被当成「假 efun」全域可见**。
编译器遇到一个既不是本对象函数、也不是内置 efun 的裸调用时，
会把它编成对 simul_efun 对象的**专用指令**（不是一般的 `call_other`），
而且因为原型在编译期已知，**回传值不必转型**，即使开了 `#pragma strict_types`。

它最有意思的用法是**覆盖 efun**：定义一个和 efun 同名的 simul_efun，
全 mudlib 的调用就都会走你的版本；你在里面做完检查，再用 `efun::move_object()`
调用真正的 driver 实作。哪些对象能用 `efun::` 前缀绕过，
由 master 的 `valid_override()` 决定。

**解决的痛点**：这三条路径合起来，回答了「driver 与 mudlib 的边界到底画在哪」——
**不是画在文件上，是画在调用方向上**。
同一个名字（比如 `move_object`）可以同时是 efun 与 simul_efun，
真正决定行为的是谁在什么方向上调用它。

**踩过的坑**：本项目 §032 的 `is_killing(ob)` 就是这个机制的受害者。
它声明成收 `string`，实际被传了 object。
如果它是 efun，driver 的参数检查会挡下来；
但它是 **simul_efun**，类型检查发生在 LPC 编译期，
而旧 MudOS 的检查比较宽松——于是这个错误在原服务器上跑了很多年没事，
换到 FluffOS 才爆。**「它在原机上跑得好好的」不构成它是对的证据。**

**优点 / 罩门**：优点是 mudlib 可以在不碰 C 的情况下改变几乎任何行为，
连 efun 都能换掉。罩门是**调试路径会变长**：
看到 `write(...)` 你不能假设它是 driver 的 `write`——
得先去 simul_efun 对象里确认有没有人盖掉它。

**效益**：对逆向工作的直接效益是**知道去哪里找**。
本项目要回答「这个 ESC 开头的字符串是谁送出去的」时，路径是固定的：
先在 mudlib 里 grep 宏名，找到 `write()` 调用点，
再确认 `write` 有没有被 simul_efun 盖掉——三步，不必碰 driver 一行 C++。

> 💡 君之一席话
> **搞清楚一个系统，先搞清楚它有几种调用方向**——往下调用的是能力，往上回呼的是契约，同层绕道的是惯例；三者混在一起看，永远看不出边界在哪。

> 🔍 老手视角──真正的门道
> apply 这个机制值得单独想一想：它是**用命名惯例取代接口声明**。好处是零样板——你不必 implement 任何 interface、不必注册任何 callback，写个叫 `heart_beat` 的函数就会被调用。代价是打错字没有人告诉你。这个取舍在现代生态里反复出现：pytest 的 `test_*`、Go 的 `TestXxx`、React 的 `useXxx`、Rails 的 convention over configuration，全是同一招。它们共同的缓解手段也一样——**linter**：既然编译器不管，就让另一个工具管。老手看到「靠命名生效」的系统，第一个动作是问「有没有东西会告诉我名字打错了」；没有的话，就自己加一条检查进 CI。可落地的做法：任何靠惯例绑定的项目，**写一支脚本把所有惯例名字列出来与实际定义比对**，成本一个下午，能挡掉一整类「它就是不动，也不知道为什么」的问题。

---

## 037　双堆栈虚拟机 — LPC 是怎么被运行的

**标签**：`#虚拟机` `#字节码` `#堆栈机` `#引用计数` `#沙盒`
**证据等级**：🟡 FluffOS 官方文档 `docs/driver/stackmachine.md`（**这份文档本身就是史料**）

**起源**：§033 引的那句「Lennart Augustsson 说服我去实作一个虚拟堆栈机的编译器」，
它的产物就是这一节要讲的东西。而最有意思的是——
**FluffOS 仓库里那份描述它的文档，文风与内容都还停在 Lars 的年代**，
开头甚至还是全大写的：

> THIS IS A DOCUMENT THAT DESCRIBES HOW A VIRTUAL STACK MACHINE HAS BEEN DEFINED,
> TO EXECUTE COMPILED LPC CODE.

三十七年，运行模型没有换过。

**技术内核**：两个堆栈。

```
    值堆栈（value stack）                 控制堆栈（control stack）
    ┌─────────────────────┐              ┌──────────────────┐
    │  ...                │              │  返回地址          │
 fp→│  参数 0             │              │  前一个 frame ptr  │
    │  参数 1             │              │  参数个数          │
    │  ...                │              │  区域变量个数      │
    │  区域变量 0          │              │  extern_call 旗标  │
    │  区域变量 1          │              └──────────────────┘
    │  ...                │
    │  暂时值              │
    ▼                     │
```

- **值堆栈**放所有东西：求值中间结果、区域变量、**以及参数**——
  参数就是区域变量，只是编号在前面。访问靠 **frame pointer 加偏移量**，
  所以是 O(1) 的索引，不是查表。
- **控制堆栈**放返回地址、前一个 frame pointer、参数与区域变量的个数。
  把个数存在这里，`F_RETURN` 才知道要释放多少东西，
  指令本身不必携带这个信息。

每个值堆栈元素的类型是 `struct svalue`——这个名字在 FluffOS 的 C++ 原代码里至今还在。

**几个关键指令的形状**（原文档的字节布局）：

| 指令 | 格式 | 语意 |
|------|------|------|
| `F_FUNCTION` | `b0=F_FUNCTION, b1b2=函数编号, b3=参数个数` | 调用本对象函数。**同时初始化 frame pointer**，并把参数个数「调整」成被调用者要的数量——不足补 0、过多弹掉 |
| `F_RETURN` | `b0` | 释放整个 frame，只留堆栈顶端一个值。若 `extern_call` 旗标为真就从求值器返回 |
| `F_CALL_OTHER` | `b1=F_CALL_OTHER, b2=参数个数` | 调用**其他对象**的函数（`ob->fn()`） |
| `F_AGGREGATE` | `b1, b2b3=数组大小（最大 0xffff）` | 从堆栈顶端取 N 个元素组成数组 |
| `F_CATCH` | `F_CATCH, b1b2=catch 之后的地址` | 运行时做 `setjmp()` 并**递归调用 `eval_instruction()`**；`F_THROW` 做 `longjmp()` |
| `F_SSCANF` | 参数个数为单一字节 | 特例：前两个参数传值，其余用 **`T_LVALUE`** 类型传参考 |

三个细节值得停一下：

1. **`F_FUNCTION` 会自动调整参数个数**。LPC 调用函数时给多给少都不会错——
   少的补 0、多的丢掉。这解释了为什么老 mudlib 里到处是参数个数对不上的调用
   却能跑；也解释了为什么**它们错得很安静**。
2. **`F_CATCH` 是靠 `setjmp`/`longjmp` 加递归求值器实作的**。
   所以 LPC 的 `catch()` 不只是语法糖，它会真的开一个新的求值 frame。
   本项目的 `master.c` 就用它包住登录对象的创建：
   `err = catch(login_ob = new(LOGIN_OB));`——**登录对象炸了，连接还在，
   还能写一行「现在有人正在修改用户连接部份的程序」给玩家**。
3. **`sscanf` 需要一个专门的 lvalue 类型**，因为 LPC 没有指针。
   这是「语言少了一个特性，就得在 VM 里补一个类型」的教科书级例子。

**内存模型**：没有 `malloc`、没有 `free`、没有 `delete`。
`allocate(n)` 的单位是**元素**不是字节。回收靠**引用计数**：
计数归零就立刻回收。所以 LPC 没有 GC 停顿，
但**有循环引用泄漏**——FluffOS 为此专门有一份 `docs/concepts/general/reference_loops.md`。

**沙盒**：LPC 是给不受信任的作者写的（想想 1989 年的原始情境：让玩家来盖世界），
所以 driver 必须能中止失控的代码。手段是 **evaluation cost**：
每运行一条指令扣一点额度，额度用完就中断该次运行并抛 LPC 错误。
`config` 里的默认值是：

| 限制 | 默认值 | 挡的是什么 |
|------|--------|-----------|
| `maximum evaluation cost` | 30,000,000 | 无穷循环 |
| `maximum call depth` | 150 | 无穷递归 |
| `evaluator stack size` | 65,536 | 堆栈爆掉 |
| `inherit chain size` | 30 | 继承链失控 |
| `maximum array size` | 15,000 | 内存炸弹 |
| `maximum mapping size` | 150,000 | 同上 |
| `maximum string length` | 1,048,576 字节 | 同上 |

**踩过的坑**：这套沙盒**在 WebAssembly 版上是关掉的**。
原生版的 eval limit 实作依赖 POSIX 的 `SIGVTALRM` 计时器，
WASM 没有——官方文档把它列为已知限制的第一条：
「`while(1);` 在 LPC 里会卡死整个分页」。
（macOS 与 Windows 上其实也没有。）
本项目的 `boot-test.mjs` 因此**必须自己带逾时**：
不能假设 driver 会替你中止跑疯的 mudlib。

**优点 / 罩门**：优点是这套设计极度耐久——三十七年、四个分支、
从 32 比特 Unix 到 64 比特 WebAssembly，运行模型一行没改。
堆栈机的简单性是它活下来的原因：**它没有什么可以过时的东西**。
罩门是性能天花板：纯解译的堆栈机没有 JIT、没有寄存器分配，
所有性能都靠 efun（也就是把热点推回 C++）。
这正是为什么一个 MUD driver 会有 600 多个内置函数——
**efun 数量是解译器慢的直接后果**。

**效益**：对逆向工作而言，知道运行模型能回答一类具体问题：
「这段 LPC 运行时，`this_player()` 是谁？」——答案在控制堆栈；
「为什么这个参数是 0？」——因为 `F_FUNCTION` 补的；
「为什么这个错误没有让服务器死掉？」——因为某个上层有 `catch()`。

> 💡 君之一席话
> **活得最久的设计通常不是最先进的那个，是最没有东西可以过时的那个**——一台双堆栈机没有依赖任何一代硬件的特性，所以它可以从 1989 年的 Unix 搬到 2026 年的浏览器分页里，一行都不用改。

> 🔍 老手视角──真正的门道
> `F_FUNCTION` 自动补齐参数这件事，是一个非常值得警惕的设计。它让 mudlib 极度好写（加一个参数不会弄坏既有调用点），但它把**一整类错误变成了数据错误**：你不会拿到「参数个数不符」，你会拿到一个 0，然后那个 0 一路流进业务逻辑，最后在离现场很远的地方变成一个看不懂的行为。这种「宽容输入」的设计在协议层也有完全一样的形状——本书 §004 那个没有跳脱机制的分隔符、§009 那个「定义了 102 个 opcode 只用 16 个」的 header，都是同一种宽容。老手评估宽容设计时只问一个问题：**宽容的代价由谁承担？**如果是写的人省事、调试的人加倍痛苦，而这两者又不是同一批人（在 mudlib 的情境里，往往差了二十年），那这笔交易就是亏的。可落地的做法：在宽容的边界上加一条可关闭的严格模式——LPC 有 `#pragma strict_types`，用它；协议层则是「解析器有一个严格模式，CI 跑严格模式，正式环境跑宽容模式」。

---

## 038　对象的一生 — clone、apply 与心跳

**标签**：`#对象模型` `#clone` `#heart_beat` `#call_out` `#热重载` `#生命周期`
**证据等级**：🟡 FluffOS 官方文档（`docs/apply/object/*`、`docs/concepts/general/hot_reload.md`、`docs/driver/config.md`）

**起源**：LPC 的对象模型和主流 OOP 差一个关键概念，
不先讲清楚，读 mudlib 会一路困惑：**文件本身就是一个对象**。

**技术内核**：**blueprint（蓝本）与 clone（拷贝体）**。

```
/inherit/weapon/sword.c
   │
   ├─ load_object()  →  蓝本对象  /inherit/weapon/sword
   │                     （一份，第一次被碰到时编译并创建）
   │
   └─ clone_object() →  拷贝体    /inherit/weapon/sword#1234
                          #1235
                          #1236 ...（要几把有几把）
```

一个 `.c` 档就是一个类别**兼**一个实例。
`clone_object()` 产生的拷贝体共用同一份编译后的字节码（program），
但各自有一份变量。所以「一万把剑」的内存成本是一万份变量，不是一万份程序。

**对象的生命周期 apply**，按时间顺序：

| 时机 | apply | 说明 |
|------|-------|------|
| 编译后、变量初始化 | `__INIT` | 由编译器产生，初始化有初值的变量 |
| 创建时 | **`create()`** | 相当于建构子。**LPC 没有 `main()`，有的是 `create()`** |
| 每次有 living 对象进入 | **`init()`** | 典型用途是 `add_action()` 注册指令 |
| 周期性 | **`reset()`** | 重生怪物、补货。缺省 `time to reset : 900` 秒 |
| 周期性（需打开） | **`heart_beat()`** | 缺省 `heartbeat interval msec : 1000`。**要 `set_heart_beat(1)` 才会跑** |
| 长时间没人碰 | `clean_up()` | 缺省 `time to clean up : 600` 秒。对象可以自己决定要不要 `destruct` |
| 内存不足／闲置 | （swap） | `time to swap : 300`，0 为停用 |
| 被摧毁时 | `on_destruct()` | 收尾 |
| 环境被摧毁时 | `move_or_destruct()` | 决定跟着死还是搬家 |

**两种周期机制不要搞混**：

| | `heart_beat()` | `call_out()` |
|---|---------------|-------------|
| 形式 | apply（driver 定时来调用你） | efun（你排一个未来的调用） |
| 频率 | 固定间隔，缺省 1 秒 | 一次性，指定几秒后 |
| 开关 | `set_heart_beat(1/0)` | 排一次跑一次 |
| 典型用途 | 战斗回合、状态衰减、饥饿值 | 延迟效果、逾时、调度 |
| 成本 | **每个开了心跳的对象每秒都在跑** | 只在到期时跑 |

第五列是性能上的重点：一个 MUD 卡不卡，
通常直接取决于**有多少对象开着 `heart_beat`**。

**时间的粒度**由 `gametick msec` 决定（缺省 1000 毫秒）——
这是「游戏内时间的最短可见间隔」。
§040 会看到，WebAssembly 版把这个时钟的推进权从 libevent 交给了浏览器分页。

**热重载**：LPMud 的招牌能力，机制比想像中细致。
`update` 一个文件不能只重载它自己，因为：

- 每个 `#include` 的标头是**编译期并进去**的；
- 每个 `inherit` 的父程序是**编译期绑进子程序**的——
  子程序链接的是它编译当下的那一份父程序。

所以改了 `/std/base.c`，只重载自己文件改过的对象，
**每一个继承者都还在跑旧 code，而且完全没有征兆**。
FluffOS 给的解法是两个编译期 master apply：

```c
mixed inherit_program(string from, string path, int priv);
mixed include_file(string compiled, string from, string path);
```

driver 在**每一次编译**时，对每个 `inherit` 调用前者、每个 `#include` 调用后者。
mudlib 只要**原样回传 `path`**（保持原行为）**并记下这条边**，
就免费得到一张完整的相依图，可以据此做出正确的自动热重载。

这个设计的漂亮之处：**它没有添加任何机制**——
两个 apply 本来就是为了安全检查（决定准不准 include／inherit）而存在的，
热重载只是把它们的**参数**拿来用。

**踩过的坑**：`create()` 与 `reset()` 的关系在 MudOS 系与 LPMud 官方在线不一样，
这是跨血脉移植 mudlib 最常撞的一堵墙。
本项目没有踩到，因为 17 个 lib 全是 MudOS 系；
但 §035 的表提醒过：**知道血脉，才知道该预期哪一套语意**。

另一个真的踩到的是 `preload`：master 的 `preload()` apply 回传一张开机时要加载的清单。
本项目 `libs/lpmudname` 里有三个 daemon 因为 WASM 版没有 `package sockets`
而编译失败（`Undefined function socket_create` 等 13 处），
**driver 照常完成开机**——preload 清单里的项目加载失败，
只会让那个对象不存在，不会中止启动。
这正是本项目 `playable` / `limited` 分级的分界线：**看的是「进不进得了世界」**。

**优点 / 罩门**：优点是「蓝本 + 拷贝体」让一份 code 服务上万个实例，
而热重载让开发循环短到近乎即时。
罩门是**状态与程序的版本会分岔**：热重载换掉的是 program，
对象里既有的变量还在——如果新版程序对变量的假设变了，
你会得到一个「跑着新程序、带着旧数据」的对象。
这是 LPMud 家族最经典的一类幽灵 bug。

**效益**：对本项目的直接效益是**知道 boot-test 要等什么**。
`boot-test.mjs` 判断一个 mud 开得起来的依据不是「driver 没有退出」，
而是走完「注册 → 建角 → 进世界」——
因为 preload 失败、reset 没跑、心跳没开，driver 都不会抱怨。
**在一个「失败是静默的」系统上，唯一可信的验证是端到端跑一遍。**

> 💡 君之一席话
> **热重载真正的难点从来不是「重新编译」，是「谁依赖了我」**——没有相依图的热重载，只是把一个明显的错误换成一个安静的错误。

> 🔍 老手视角──真正的门道
> `inherit_program` / `include_file` 这一手很值得抄：**FluffOS 没有为了热重载添加任何 API**，它把两个既有的安全检查点的参数拿来当相依图的来源。这是「机制复用」的典范——同一个 hook 点，一个消费者用它的**回传值**（安全决策），另一个消费者用它的**参数**（可观测性）。老手在设计扩充点时会刻意留这个余裕：hook 的签名尽量带齐上下文，即使当下的用途只需要一个布尔值。因为扩充点的真正价值往往不在你设计它时想到的那个用途上。可落地的做法：任何 callback／middleware／拦截器，**签名里多带「这件事发生在什么情境下」的参数**（谁、为了谁、在编译／处理哪个东西时），成本几乎为零，而它会在两年后救你一次。

---

## 039　config — driver 的全部旋钮，以及本项目在上面栽的跟头

**标签**：`#设置档` `#config` `#external_port` `#幂等` `#工具设计`
**证据等级**：🟢 本项目实测（17 个 mudlib 的导入）＋ 🟡 FluffOS `docs/driver/config.md`

**起源**：driver 启动时只吃一个东西：一个设置档的路径。
`driver.exe config.ini`——之后的一切（mudlib 在哪、master 是谁、开哪些端口）
全部来自那个文件。所以**要把一份陌生的 mudlib 跑起来，
第一件事永远是把它的设置档弄对**。

本项目导入 17 个 mudlib 的过程，一半的时间花在这个文件上。

**技术内核**：本项目上游 `LPMud-Name/world/config.ini` 的关键字段，逐条说明——
这份文件刚好把最重要的几类旋钮都用到了：

```ini
name : 江湖论剑

external_port_1 : telnet 5001         # ★ UTF-8
external_port_2 : telnet 5003         # ★ GBK（由 mudlib 自己切换）
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

| 字段 | 作用 | 为什么重要 |
|------|------|-----------|
| `master file` | §036 那一整排 master apply 住在哪 | **弄错就完全开不了机** |
| `simulated efun file` | §036 的 ③ 从哪里来 | 弄错的话 mudlib 到处是「未定义函数」 |
| `include directories` | `#include <...>` 的搜索路径 | 冒号分隔多个 |
| `global include file` | **每个文件自动 include 这个** | 读 mudlib 时常见的「这个宏哪来的」，答案通常在这 |
| `external_port_N` | 开哪些端口、讲哪种协议（`telnet`／`websocket`／`ascii`／`binary`，可加 `_tls`） | ★ 见下方 |
| `websocket http dir` | WebSocket 端口同时当静态网页服务器 | driver 本身就能发网页 |
| `default fail message` | 指令不存在时的消息 | **值可以直接包含裸 ESC 字符**——本书所有 ANSI 方言的最外层来源之一 |

driver 的旋钮远不只这些。与本书关系最近的几类，附官方默认值：

| 类别 | 代表旋钮（默认值） |
|------|------------------|
| 时序 | `gametick msec`(1000)、`heartbeat interval msec`(1000)、`time to reset`(900)、`time to clean up`(600)、`time to swap`(300) |
| 限制 | `maximum evaluation cost`(30,000,000)、`maximum call depth`(150)、`evaluator stack size`(65,536)、`maximum array size`(15,000)、`maximum string length`(1 MiB) |
| 哈希表 | `object table size`(4096，建议是对象数的 1/4)、`hash table size`(65,536，建议是相异字符串数的 1/5，且应为质数)、`living hash table size`(256，**只能是 4/16/64/256/1024/4096**) |
| 语意开关 | `sane explode string`(1)、`sane sorting`(1)、`old range behavior`(0)、`this_player in call_out`(1) |
| reset 行为 | `no resets`(0)、`lazy resets`(0)、`randomized resets`(1) |

**「语意开关」那一列是兼容性层的真面目**：
`sane explode string`、`old range behavior` 这种旗标的存在本身就是历史证据——
**driver 知道自己以前的行为是错的，但不敢直接改，只能给一个开关**。
§035 说的「兼容性承诺会冻结错误」，在这里是可以逐行读出来的。

**踩过的坑（三个，全部是本项目实测）：**

**① `external_port_1` 决定 WebAssembly 版走哪个编码分支。**
本项目的 mudlib 在 master 里写着：

```c
object connect(int port)
{
    if (port == 5003) {
        set_encoding("GBK");
    }
    ...
}
```

而 WASM 版的 ICU 数据被裁掉了表格式字集（§041），
真的走进 GBK 分支会直接 raise error。
它之所以没炸，是因为 driver 的 `wasm_console_connect()`
把每条连接都标成「来自**第一个** external_port」
（`src/wasm/comm_wasm.cc:124-126`）——也就是 `telnet 5001`，UTF-8 分支。

> **本项目因此定下一条打包规则：第一个 `external_port` 必须是 UTF-8 的那一个。**
> 这条规则写在 `libs/lpmudname/NOTES.md` 的「没有踩到的坑」一节——
> **没踩到的坑也要记**，否则下一个人把两个端口的顺序调换，会得到一个非常难查的错误。

**② 这个设置档没有行尾注解。**
官方文档写得很清楚：*“Lines beginning with `#` are comments”*——
**只有整行注解**。值取的是冒号后面的整行。

本项目在自动修正脚本里把说明写成行尾注解：

```ini
external_port_1 : telnet 5001   # 改成 UTF-8 端口
                              └──────┬──────┘
                                 这整段都是值的一部分
```

三个原本已经修好的 lib 因此变回 `noboot`。

**③ 而且那支修正脚本不是幂等的。**
它每重跑一次就再叠一层注解，于是「修一次坏一点」。
这是本项目整批导入过程中**最贵的一次错误**，教训写在 §032：

> **自动修正工具本身必须是幂等的，否则它会变成另一种损坏来源。**

第三个坑比前两个严重得多，因为前两个是**知识**问题（不知道规则），
第三个是**工程**问题（工具的性质不对）。知识可以查，
而一个非幂等的修正工具会在你查数据的时候继续破坏现场。

**优点 / 罩门**：优点是所有行为集中在一个纯文本档，
没有环境变量、没有命令行旗标的组合爆炸，`diff` 两份 config 就能比较两台 mud。
罩门是**没有 schema、没有验证**：打错字段名不会有人告诉你，
只会得到默认值或一个很远的错误（本项目看过的极端例子是设置里写着
原营运者机器上的绝对路径 `c:\hell`，症状是 `Bad mudlib directory`）。

**效益**：本项目把「设置档对不对」变成建置管线的一部分——
`boot-test.mjs` 真的用那份 config 开一次机、走一次注册建角。
**没有 schema，就用端到端测试当 schema。**

> 💡 君之一席话
> **修东西的工具，本身必须能安全地重跑**——不能重跑的修正工具，第二次运行时就从解决方案变成故障源。

> 🔍 老手视角──真正的门道
> 那条「第一个 external_port 必须是 UTF-8」的规则，形状值得注意：它是一条**跨层的隐性契约**——`config.ini`（设置层）的**顺序**，决定了 `master.c`（mudlib 层）走哪个分支，而中间的因果藏在 driver 的 C++ 原代码里（`comm_wasm.cc` 的三行）。这种契约最危险的地方在于它在任何一层看起来都不像契约：config 看起来只是列了两个端口，master.c 看起来只是判断了一个端口号。老手处理这类契约的方法只有一个——**把它写在会被读到的地方，并附上「凭什么」**。本项目的做法是写进该 lib 的 `NOTES.md` 并附上 `comm_wasm.cc:124-126` 的行号；更强的做法是让打包脚本直接**检查**这件事并在违反时拒绝打包。可落地的判准：一条规则如果只存在于某个人的脑袋里，它的半衰期大约是三个月；写进注解大约两年；写进会失败的检查，才真的活得比你久。
