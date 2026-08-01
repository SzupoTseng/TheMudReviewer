# 附录 E　FluffOS / LPC 技术速查

> 第八、九篇把 driver 的构造分散在各节解释。这份附录把**可查表的部分**集中起来：
> efun 分类、apply 全表、config 全旋钮、telnet 选项、套件矩阵、WASM 导出 API。
>
> 证据等级：🟡 上游 `fluffos` 仓库（`docs/`、`src/net/telnet.cc`、`src/CMakeLists.txt`）
> ＋ 🟢 本项目对 `v2026.0729.0` 的实测。
> **数量全部是撰写当下数出来的，不是估计。**

---

## E.1　三个调用方向（§036 的速查版）

| 方向 | 机制 | 谁定义 | 找不到时 |
|------|------|--------|---------|
| mudlib → driver | **efun** | driver（C++） | 编译错误 |
| driver → mudlib | **apply** | mudlib（LPC） | **静默略过** |
| mudlib → mudlib | **simul_efun** | mudlib 的单一对象 | 编译错误 |

**记住这一条就够了：apply 打错字不会有人告诉你。**

---

## E.2　efun 目录（451 篇文档，26 类）

| 类别 | 篇数 | 代表 efun |
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
| **`jsbridge`** | 3 | **`js_eval` `js_call` `js_export`**（**仅 WASM**，§042） |
| `crypto` | 1 | `hash` |
| `external` | 1 | `external_start` |

> ⚠️ **`crypt()` 在 `strings` 而不是 `crypto`。** 这件事救了本项目 17 份 mudlib——
> 关掉 crypto 套件不会让登录流程消失（§041、§044）。
> **efun 住在哪个套件，比它叫什么名字重要。**

---

## E.3　apply 全表（69 个）

### E.3.1　object applies（11 个）— 每个对象的生命周期

| apply | 何时被调用 |
|-------|-----------|
| `__INIT` | 编译器产生，初始化有初值的变量 |
| **`create`** | 对象创建时（相当于建构子；**LPC 没有 `main()`**） |
| **`init`** | 每次有 living 对象进入时（典型用途：`add_action`） |
| **`reset`** | 周期性（缺省 900 秒）——重生怪物、补货 |
| **`heart_beat`** | 周期性（缺省 1000 ms），**要 `set_heart_beat(1)` 才会跑** |
| `clean_up` | 长时间没人碰（缺省 600 秒） |
| `id` | 「这个名字指的是我吗」 |
| `is_living` | 是不是生物 |
| `move_or_destruct` | 环境被摧毁时：跟着死还是搬家 |
| `on_destruct` | 被摧毁时的收尾 |
| `virtual_start` | virtual object 的起始 |

### E.3.2　master applies（37 个）— 全域事件、安全与编译决策

| 分组 | apply |
|------|-------|
| **连接** | **`connect`**（§044）、`epilog`、`preload`、`crash`、`flag` |
| **编译** | `compile_object`（virtual object）、**`include_file`**、**`inherit_program`**（热重载相依图，§038）、`get_include_path`、`make_path_absolute` |
| **错误** | `error_handler`、`log_error`、`parser_error_message` |
| **身分** | `creator_file`、`author_file`、`domain_file`、`privs_file`、`get_root_uid`、`get_bb_uid`、`object_name`、`get_save_file_name` |
| **安全（`valid_*`，14 个）** | `valid_read` `valid_write` `valid_object` `valid_shadow` `valid_override` `valid_seteuid` `valid_socket` `valid_bind` `valid_link` `valid_hide` `valid_database` `valid_ffi` `valid_save_binary` |
| **其他** | `get_mud_stats`、`retrieve_ed_setup`、`save_ed_setup` |

> 🟢 §032 那个「建角时 `Denied write permission`」的坑就在 `valid_write`：
> 找不到 SECURITY_D 就一律拒绝（fail-closed），
> 而**同一个文件里的 `valid_read` 是 fail-open**——两个相反的缺省。

### E.3.3　interactive applies（21 个）— 一条连接的网络事件

| 分组 | apply |
|------|-------|
| **登录／断线** | **`logon`**（§044）、`net_dead` |
| **输入** | `process_input`、`write_prompt`、`receive_ed` |
| **输出** | `catch_tell`、`receive_message`、`receive_snoop`、`terminal_colour_replace` |
| **telnet 协商** | `telnet_suboption`、`terminal_type`、`window_size`、`receive_environ` |
| **扩充协议（§056）** | `gmcp` `gmcp_enable` `msdp` `msdp_enable` `msp_enable` `mxp_enable` `mxp_tag` `zmp` |

> **最后一列就是「driver 层协议」与「mudlib 层协议」的分界线**（§043）：
> GMCP／MSDP／MXP／ZMP 有 apply，driver 知情；
> **ZJMUD 没有——driver 从头到尾不知道它存在。**

---

## E.4　config 旋钮（§039 的完整版）

### E.4.1　必填与内核

| 设置 | 说明 |
|------|------|
| `name` | **必填**。这台 mud 的名字 |
| `mudlib directory` | **必填**。mudlib 根目录（**唯一不相对于 mudlib 的路径**） |
| `log directory` | **必填**。debug.log 与统计档的位置 |
| `include directories` | **必填**。`#include <...>` 的搜索路径，冒号分隔 |
| `master file` | **必填**。master 对象（弄错就完全开不了机） |
| `simulated efun file` | simul_efun 对象 |
| `global include file` | **每个文件自动 include** ——「这个宏哪来的」通常在这 |
| `debug log file` | log 文件名 |
| `mud ip` | 多 IP 主机上要绑哪一个 |
| `websocket http dir` | WebSocket 端口兼静态网页根目录 |

### E.4.2　时序与生命周期

| 设置 | 缺省 |
|------|-----:|
| `gametick msec` | 1000 |
| `heartbeat interval msec` | 1000 |
| `time to reset` | 900 |
| `time to clean up` | 600 |
| `time to swap` | 300（0 = 停用） |

### E.4.3　限制（沙盒，§037）

| 设置 | 缺省 |
|------|-----:|
| `maximum evaluation cost` | 30,000,000 |
| `maximum call depth` | 150 |
| `evaluator stack size` | 65,536 |
| `inherit chain size` | 30 |
| `maximum local variables` | 64（范围 64–255） |
| `maximum array size` | 15,000 |
| `maximum mapping size` | 150,000 |
| `maximum string length` | 1,048,576 字节 |
| `maximum buffer size` | 1,048,576 |
| `maximum bits in a bitfield` | 12,000 |
| `maximum byte transfer` | 262,144 |
| `maximum read file size` | 262,144 |

### E.4.4　哈希表（调校用）

| 设置 | 缺省 | 建议值 |
|------|-----:|--------|
| `hash table size` | 65,536 | 相异字符串数的 **1/5**，且应为质数 |
| `object table size` | 4,096 | 对象数的 **1/4**（最小 1024） |
| `living hash table size` | 256 | **只能是 4/16/64/256/1024/4096** |

### E.4.5　语意开关（兼容性层的真面目，§039）

| 设置 | 缺省 | 意义 |
|------|-----:|------|
| `sane explode string` | 1 | `explode()` 最多剥一个前导分隔符 |
| `reversible explode string` | 0 | 让 `implode(explode(x,y),y) == x`；覆盖上一项 |
| `sane sorting` | 1 | 排序有明确且稳定的顺序 |
| `old range behavior` | 0 | 负索引从尾端算（**旧行为**） |
| `warn old range behavior` | 1 | 用到旧行为就警告 |
| `this_player in call_out` | 1 | `call_out` 回呼里 `this_player()` 可用 |
| `enable_commands call init` | 1 | `enable_commands()` 时调用 `init()` |
| `reverse defer` | 0 | `defer()` 反序运行 |
| `sprintf add_justified ignore ANSI colors` | 1 | 对齐时忽略 ANSI 色码的宽度 |
| `call_out(0) nest level` | 1000 | `call_out(0)` 连锁的嵌套上限 |
| `call other type check` / `call other warn` / `old type behavior` | 0 / 0 / 0 | `->` 的类型检查强度 |

> **`old range behavior` 与 `old type behavior` 这种旗标的存在本身就是史料**——
> driver 知道自己以前的行为是错的，但不敢直接改，只能给一个开关（§035）。

### E.4.6　reset 行为／诊断／玩家 I/O

| 设置 | 缺省 |
|------|-----:|
| `no resets` / `lazy resets` / `randomized resets` | 0 / 0 / **1** |
| `no ansi`（把输入里的 ESC 换成空白） | **1** |
| `strip before process input` | 1 |
| `interactive catch tell` | 0 |
| `receive snoop` / `snoop shadowed` | 1 / 0 |
| `trace` / `trace code` | 1 / 0 |
| `display preload progress` | 1 |
| `mudlib error handler` / `trap crashes` | 1 / 1 |

### E.4.7　★ 协议开关 — 直接决定线路上的协商字节

| 设置 | 缺省 | 对应 telnet 选项 |
|------|-----:|----------------|
| `enable mssp` | **1** | 70 |
| `enable msp` | **1** | 90 |
| `enable mxp` | 0 | 91 |
| `enable gmcp` | 0 | 201 |
| `enable zmp` | 0 | 93 |
| `enable msdp` | 0 | 69 |

> 🟢 **本项目实测验证了这张表**：WASM driver 的首包里出现了
> `IAC WILL 70`（MSSP）与 `IAC WILL 90`（MSP）——**两个缺省为 1 的**；
> 而 MXP／GMCP／ZMP／MSDP **一个都没出现**——**四个缺省为 0 的**（§055）。
> **设置档的默认值，在连接的第 10 个字节上就看得见。**

### E.4.8　端口与安全

```ini
external_port_1 : telnet 5001          # 类型：telnet / websocket / ascii / binary
external_port_2 : telnet 5003
external_port_3 : websocket 5004
# external_port_2_tls : cert=cert.crt key=cert.key
```

| 安全设置 | 说明 |
|---------|------|
| `ffi allowed libraries` | `ffi_load()` 可打开的 .so 白名单（空 = 全交给 `valid_ffi` apply） |
| `allowed os environment variables` | `get_os_env()` 可读的环境变量白名单（**空 = 全部拒绝**） |
| `writable os environment variables` | `set_os_env()` 可写的白名单（**空 = 全部拒绝**） |

> ⚠️ **格式陷阱（§039，本项目最贵的一次错误）**：
> **只有整行 `#` 注解，没有行尾注解**——值取的是冒号后**整行**。

---

## E.5　telnet 选项速查（§055）

| 编号 | 名称 | 用途 | FluffOS 立场 | 🟢 出现在实测首包？ |
|-----:|------|------|-------------|------------------|
| 0 | BINARY | 8-bit 透传 | WILL / DO | — |
| 1 | ECHO | **回显控制（密码屏蔽）** | WILL / DO | — |
| 3 | SGA | Suppress Go Ahead——**字符模式信号** | WILL / DO | — |
| 6 | TM | Timing Mark | WILL / DO | — |
| **24** | **TTYPE** | 终端机类型 | WONT / **DO** | ✅ `ff fd 18` |
| **31** | **NAWS** | 窗口大小（改变时重报） | WILL / DO | ✅ `ff fd 1f` |
| 34 | LINEMODE | 行模式 | WONT / DO | — |
| **39** | **NEW-ENVIRON** | 环境变量 | WONT / **DO** | ✅ `ff fd 27` |
| **42** | **CHARSET** | 字集协商 | WILL / DO | ✅ `ff fb 2a` |
| 69 | MSDP | 频外类型化 KV | WILL / DO | ❌（缺省关） |
| **70** | **MSSP** | MUD 服务器状态 | WILL / DO | ✅ `ff fb 46` |
| 85 | COMPRESS | MCCP v1 | — | — |
| **86** | **COMPRESS2** | **MCCP2 压缩** | WILL / DO | ⚠️ **只有原生版**（`ff fb 56`）——WASM 版 zlib 被移除（§041） |
| **90** | **MSP** | 音效 | WILL / DO | ✅ `ff fb 5a` |
| 91 | MXP | 行内标记 | WILL / DO | ❌（缺省关） |
| 93 | ZMP | Zenith MUD Protocol | WILL / DO | ❌（缺省关） |
| 200 | ATCP | Achaea Telnet Client Protocol | — | — |
| 201 | GMCP | 频外 JSON | WILL / DO | ❌（缺省关） |

**协商框架**：

```
IAC WILL <opt>   我想激活（我这边做）      IAC WONT <opt>   我不做
IAC DO   <opt>   请你激活（你那边做）      IAC DONT <opt>   请你别做
IAC SB <opt> …数据… IAC SE                子协商数据信道
IAC IAC                                   跳脱的 0xFF
```

**四个性质**：对称、双向独立（`WILL` 与 `DO` 各谈各的）、
缺省关闭、**拒绝是合法且无副作用的**——最后一条就是本项目「一律拒绝」策略成立的理由。

---

## E.6　WebAssembly 平台（§040–§042）

### E.6.1　导出 API

| 导出 | 签名 | 对应原生的什么 |
|------|------|--------------|
| `fluffos_boot` | `(string config) → int` | `main()` 读设置、加载 master、preload |
| `fluffos_tick` | `(number now_ms) → int` | libevent 的一圈循环 |
| `fluffos_connect` | `() → int`（连接 id，负值 = 失败） | accept 一条连接 |
| `fluffos_input` | `(int id, array bytes, int n)` | 一次 socket read |
| `fluffos_disconnect` | `(int id)` | 关一条连接 |
| `fluffos_shutdown` | `()` | 关机 |
| `fluffos_flag` | `(string)` | master 的 `flag()` apply |
| **回呼** `Module.fluffos.onOutput` | `(id, bytes)` | 一次 socket write |
| **回呼** `Module.fluffos.onDisconnect` | `(id)` | 对端消失 |

### E.6.2　三条必守规则（🟢 本项目各踩过一次）

| # | 规则 | 违反的症状 |
|---|------|-----------|
| 1 | **绝不在 `onOutput` 里调用 `fluffos_input`**——一律排队、只在 tick 的堆栈上送 | 服务器只回 2 行就卡住 |
| 2 | **不能用 `connId === null` 当「尚未连接」**——`fluffos_connect()` 会在回传前就同步回呼 | 第一次回复被静默丢弃（`rawSend` 调用次数 0） |
| 3 | **`fluffos_tick` 要喂「经过多久」不是「现在几点」**——开机当下记原点、之后送差值 | `call_out(…, 30)` 第一拍就到期（「您花在连线进入手续的时间太久了」） |

### E.6.3　套件矩阵

| 开着 | 关掉 | 仅 WASM |
|------|------|--------|
| core、ops、math、matrix、trim、uids、sha1、parser、contrib、develop、mudlib_stats、**pcre** | **sockets**、compress、external、async、**db**、crypto、ffi | **jsbridge** |

**第三方函数库**：移除 libevent／libwebsockets／OpenSSL／zlib／jemalloc／backward-cpp；
**保留** libtelnet（页面讲真 telnet）、ICU、libpcre、musl crypt。

### E.6.4　已知落差

| 落差 | 症状 |
|------|------|
| **没有 eval limit** | LPC 的 `while(1);` **卡死整个分页**（POSIX 计时器只支持 Linux） |
| **ICU 只剩算法式字集** | 数据档由 ~30 MB 裁到 ~780 KB（只留 brkitr）→ **GBK／Big5／Shift-JIS 消失**，`set_encoding("GBK")` 会 raise error |
| 没有真 DNS | `query_ip_number()` 一律 `127.0.0.1` |
| 没有 zlib | 压缩 `write_file`(flag 2) 报错；压缩 `save_object` 退化为纯文本 |
| MEMFS 不落地 | 重整即失 |
| 背景分页 | 计时器被暂停，醒来补跑 gametick，**上限 100 拍** |

**体积**：`fluffos.wasm` ~3.6 MB 原始 / **~0.8 MB brotli**（~1.1 MB gzip）+ ~110 KB JS glue。

---

## E.7　VM 指令格式（§037）

| 指令 | 格式 | 语意 |
|------|------|------|
| `F_FUNCTION` | `b0, b1b2=函数编号, b3=参数个数` | 调用本对象函数；初始化 frame pointer；**自动调整参数个数**（不足补 0、过多弹掉） |
| `F_RETURN` | `b0` | 释放整个 frame，只留堆栈顶端一值 |
| `F_CALL_OTHER` | `b1, b2=参数个数` | 调用其他对象（`ob->fn()`） |
| `F_AGGREGATE` | `b1, b2b3=数组大小`（最大 0xffff） | 从堆栈顶端取 N 个组成数组 |
| `F_CATCH` | `F_CATCH, b1b2=之后的地址` | `setjmp()` + **递归调用求值器**；`F_THROW` 做 `longjmp()` |
| `F_SSCANF` | 参数个数为单一字节 | 前两参数传值，其余用 **`T_LVALUE`** 传参考 |

**两个堆栈**：值堆栈（`struct svalue`，参数即区域变量，靠 frame pointer 索引）
与控制堆栈（返回地址、前一个 fp、参数／区域变量个数、`extern_call` 旗标）。

**内存**：没有 `malloc`／`free`；`allocate(n)` 的单位是**元素**；回收靠**引用计数**
（没有 GC 停顿，但**有循环引用泄漏**）。

---

## E.8　LPC 与 C 的差异速查

| 面向 | C | LPC |
|------|---|-----|
| 进入点 | `main()` | **`create()`**（apply） |
| 内存 | `malloc`/`free` | **无**；`allocate(元素数)`；引用计数回收 |
| 字符串 | `char*` + `strcpy`/`strcat` | **内置类型**，`+` 直接串接（近 BASIC） |
| 结构 | `struct` / `union` | **`class`**（无 union）；或用 mapping |
| `->` | 结构成员 | **两种用途**：`call_other()` **与** class 成员 |
| `sscanf("%s %s")` | 两个「词」 | **第一个词 + 剩下全部**（⚠️ 最常见的移植地雷） |
| 指针 | 有 | 无（`ref`/`&` 传参考） |
| 编译 | 机器码 | **字节码 + 解译**，可热重载 |
| 额外类型 | — | `object` `mapping` `mixed` `function` `buffer` |
| 现代语法糖 | — | mapping 的 `m.key`、可选链 `m?.key?.deep`（读取用） |

---

## E.9　本项目用到的版本

| 项目 | 值 |
|------|---|
| FluffOS driver | **`v2026.0729.0`**（官方 wasm release，2.4 MB zip） |
| 映像格式 | 自订（`webclient/src/js/mudlibimage.js`，`format: 1`） |
| 收录 mudlib | **17 份**（另有 1 份导入中） |
| 客户端测试 | **202 条，12 个文件，全过** |

> 取得 driver：`node webclient/tools/fetch-driver.mjs`
> 打包全部 mud：`node webclient/tools/build-site.mjs`
> 全链路验证：`node webclient/tools/verify-fullstack.mjs`
