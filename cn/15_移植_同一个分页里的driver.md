## 040　三个没有浏览器对应物的东西 — Transport、反转的循环、VFS

**标签**：`#WebAssembly` `#移植` `#接口抽取` `#事件循环` `#链接期分派`
**证据等级**：🟡 上游原代码与文档（`fluffos/src/wasm/README.md`、`src/net/transport.h`）

**起源**：§032 从**客户端**这一侧写了「服务器搬进分页」这件事——
症状、三次误判、真因。这一节换到**driver**那一侧：
一个 1989 年设计、假设自己独占一个 Unix 行程的东西，
要怎么搬进一个「不准阻塞、没有 socket、没有文件系统」的分页里？

上游的答案很干净：**只有三样东西在浏览器里没有对应物，就只抽三个接缝。**

**技术内核**：

### 接缝一：socket → `Transport`

原本 `comm.cc` 直接对 fd 读写。移植的做法是抽出一个抽象字节管道：

```cpp
// net/transport.h — 每个 interactive_t 拥有一个
write() / flush() / schedule_command() / close()
```

三个实作，**链接期择一**：

| 实作 | 文件 | 用在哪 |
|------|------|--------|
| `SocketTransport` | `net/transport_libevent.cc` | 原生 telnet（bufferevent／TLS） |
| `WebsocketTransport` | 同上 | 原生 WebSocket（libwebsockets） |
| `WasmConsoleTransport` | `wasm/comm_wasm.cc` | **浏览器分页** |

关键是它上面的东西**一行都没改**：`comm.cc`（用户、指令队列、`input_to`、
提示、snoop）与 `net/telnet.cc`（完整的 telnet 协商，走 libtelnet）
在每个平台上编译的是同一份原代码。

`WasmConsoleTransport` 的两个方向：
- **出**：字节交给 JS 的 `Module.fluffos.onOutput(id, bytes)`；
- **入**：`fluffos_input(id, bytes)` → `comm_telnet_received()`——
  **和 socket 读取走的是同一条路径**。

而 `fluffos_connect()` 则是原生 accept 流程的镜像：
`user_add()` → telnet 初始化与初始协商 → `master::connect(port)` → `logon()`。
没有监听 socket（`init_user_conn` 是空实作）。

> **这件事对客户端的直接后果**：driver 送出的是**真正的 telnet 串流**。
> 没有中间进程可以代劳剥 IAC，浏览器得自己来——
> 这就是本项目 `src/js/telnet.js` 存在的理由（§附录 B）。

### 接缝二：事件循环被反转

原生 driver 永远阻塞在 libevent 的 `event_base_loop()` 里。
**分页不能阻塞——事件循环是页面的，不是你的。**

上游的处理方式很值得学：**只把「推进时间的那个循环」换掉，调度内核留在原地**。
gametick 队列、`add_gametick_event()`、各种维护事件全都还在共用的 `backend.cc`；
`wasm/backend_wasm.cc` 只换掉循环本身，用一个纯粹的 wall-time 优先队列，
**完全不用 libevent**。页面在计时器里调用 `fluffos_tick(now_ms)`，
它排掉所有到期的事件，并把时间推进「已经过了几个 gametick」
（分页被暂停时会补跑，但**有上限**——最多 100 拍）。

心跳、`call_out`、reset、回收——driver 的所有调度本来就走这个 API，
**所以它们上面的一切都不知道自己被移植了**。

### 接缝三：文件系统 → MEMFS

driver 的文件 I/O（编译、`save_object`、`read_file`、`ed`、log）
是普通的 POSIX 加 `ghc::filesystem`，Emscripten 直接把它映到内存文件系统上——
**driver 端零修改**。

mudlib 用 Emscripten 的 `file_packager` 打包成单一 `mudlib.data` 映像
加一个 `mudlib.js` 加载器，在 runtime 起来之前挂载好。
写入落在 MEMFS，**只活到分页关掉为止**。

**解决的痛点**：三个接缝换来一件事——**共用逻辑里没有任何 `#ifdef __EMSCRIPTEN__`**。
上游文档明讲：平台条件编译只剩两处（`net/net_compat.h` 的类型声明，
以及 `tracing.cc` 里一个线程能力判断）。
其他 per-target 的单例全部用同一招——**链接期选实作**：

| 接口 | 原生 | WASM |
|------|------|------|
| 事件循环（`backend.h`） | `backend_libevent.cc` | `wasm/backend_wasm.cc` |
| 连接传输 | `net/transport_libevent.cc` | `wasm/comm_wasm.cc` |
| TLS（`net/tls.h`） | `net/tls.cc` | **完全不链接**（唯一的共用调用点 `sys_reload_tls` 从该平台排除） |
| DNS 解析 | `dns_libevent.cc` | `dns_stub.cc` |
| 当机处理 | `crash_handler.cc`（backward-cpp） | `crash_handler_wasm.cc` |

**踩过的坑（上游的，不是本项目的）**：TLS 那一列的处理方式特别诚实——
**不是给一个假的实作，是连调用它的那个 efun 都从这个平台移除**。
所以 WASM 版的 mudlib 调用 `sys_reload_tls()` 会得到「这个函数不存在」的编译错误，
而不是一个安静地什么都不做的桩。
**在移植层，「明确地不存在」比「存在但无效」好**——
后者会让 mudlib 以为自己开了 TLS。

**优点 / 罩门**：优点是这个移植没有制造第二份 driver：
telnet 协商、用户管理、`input_to`、提示、snoop、编译器、VM 全部是同一份 code，
所以「原生会动、WASM 不会动」这种分岔几乎不可能出现在上层。
罩门是**下层分岔全部集中在三个文件里**，而这三个文件的行为差异
（没有网络延迟、没有 eval limit、没有真 DNS）会以非常间接的方式冒出来——
§042 的三个陷阱全部属于这一类。

**效益**：对本项目而言，这三个接缝直接决定了客户端要负责什么：

| 接缝 | 客户端因此必须做的事 | 本项目的哪个文件 |
|------|-------------------|-----------------|
| Transport 是字节管道 | 自己剥 telnet IAC、自己分行 | `src/js/telnet.js` |
| 事件循环由页面驱动 | 自己用 `setInterval` 泵 `fluffos_tick` | `src/js/wasmdriver.js` |
| 文件系统是 MEMFS | 自己下载映像、挂载、清楚告知「重整即失」 | `src/js/mudlibimage.js`、`wasmboot.js` |

> 💡 君之一席话
> **移植一个东西时，先数清楚「目标环境真的没有的东西」有几样**——通常比想像的少；抽那几个接缝，其余一行都别碰，否则你得到的不是移植，是分岔。

> 🔍 老手视角──真正的门道
> 「链接期分派」这一招在今天有点被遗忘，值得重新提一下：同一个标头、多个 `.cc`、建置系统选一个连进去——没有虚函数表的成本、没有 `#ifdef` 把共用文件切得七零八落、也没有运行期的分支。它的隐形好处是**编译器会逼你把接口补完**：少实作一个函数就是链接错误，而 `#ifdef` 少写一段只会安静地少一段行为。老手在做跨平台时的顺序通常是：先用 `#ifdef` 快速跑通（探索期），再把 `#ifdef` 收敛成接口 + 多实作（稳定期）——**跳过第一步会过度设计，停在第一步会腐烂**。可落地的判准：当同一个 `#ifdef` 在三个以上的文件里出现，就该把它换成一个接口了。

---

## 041　被拿掉的东西 — 套件矩阵，与 mudlib 上看得见的形状

**标签**：`#套件` `#编译期开关` `#ICU` `#编码` `#落差揭露`
**证据等级**：🟡 上游 `src/wasm/README.md` 的套件矩阵 ＋ 🟢 本项目实测（17 个 lib 的 `NOTES.md`）

**起源**：§036 说过，efun 是按套件（package）打包的，套件是**编译期开关**。
WASM 版关掉了一批。这一节要回答的是：
**关掉的那些，在 mudlib 上长什么样子？**

这件事本项目有第一手数据——17 个 mudlib 各自的 `NOTES.md` 就是逐个量出来的落差表。

**技术内核**：先看第三方函数库这一层。

| 组件 | WASM 版 | 为什么 |
|------|--------|--------|
| libevent | **移除** | 换成 host 驱动的 tick 队列（§040） |
| libwebsockets、`net/websocket.cc` | **移除** | 页面**就是**客户端，没有监听 socket |
| OpenSSL、`net/tls.cc` | **移除** | TLS 由浏览器负责，不需要在这里终结 |
| **libtelnet、`net/telnet.cc`** | **保留** | 纯 C、可携；**页面讲的是真 telnet** |
| **ICU** | **保留**（交叉编译） | 字素迭代、字集转换、`sprintf` 宽度计算 |
| libpcre（8.x） | **保留**（JIT 关闭——wasm 没有可运行页） | `pcre_*` efun |
| zlib | **移除** | MCCP 与 compress 套件都关了 |
| jemalloc | **移除** | 换成 emscripten 的 dlmalloc |
| backward-cpp | **移除** | wasm 没有原生 unwinder |
| POSIX eval-limit 计时器 | **自动停用** | 只支持 `__linux__` |

再看套件矩阵：

| 开着 | 关掉 | 只有 WASM 才有 |
|------|------|--------------|
| core、ops、math、matrix、trim、uids、sha1、parser、contrib、develop、mudlib_stats、**pcre** | **sockets**、compress、external、async、**db**、crypto、ffi | **jsbridge**（`js_eval()` / `js_call()` / `js_export()`：LPC ↔ 页面 JavaScript 双向） |

**踩过的坑（本项目实测）：关掉的套件不会在开机时告诉你。**

`libs/lpmudname/NOTES.md` 记的是这样：

> WASM build 没有 `package sockets`，以下 preload daemon 在加载时编译失败
> （`Undefined function socket_create` 等共 13 处）：
>
> | 对象 | 原本做什么 | 在 WASM 上的后果 |
> |------|-----------|-----------------|
> | `/adm/daemons/kuafu` | 跨服连接（socket 监听） | 跨服功能不存在 |
> | `/adm/daemons/qqd` | QQ 相关对外服务 | 不存在 |
> | `/adm/daemons/miraid` | 对外通报 | 不存在 |
>
> driver **照常完成开机**——这三个是 `preload` 清单里的项目，
> 加载失败只会让那个对象不存在，不会中止启动。

这正是 §038 讲的：**preload 失败是静默的**。
所以「开得起来」与「功能完整」是两件事，而 driver 不会替你区分。
本项目的分级（`playable` / `limited` / `noboot`）
之所以要靠**实际跑一遍注册建角**来决定，原因就在这里。

**★ 最值得注意的一项落差：ICU 被裁掉了表格式字集。**

上游为了体积把 ICU 的数据档（原本约 30 MB）用 `icupkg` 裁到约 780 KB，
只留下断词规则（brkitr）与转换器别名表。理由是 driver 只从数据档里拿断词数据——
字符属性与 NFC 已编进 `libicuuc`，而 UTF-8／UTF-16／Latin-1／ASCII 的转换是**算法式**的，
不需要数据表。

**代价是：GBK、Big5、Shift-JIS 这些表格式字集消失了。**
`set_encoding()`／`string_encode()`／`buffer_transcode()` 一旦指向它们就会 raise LPC error。
（LPC 可以用 `__WASM__` 判断并自行调整。）

对一本讲**中文 MUD 协议**的书来说，这一条不是细节，是主线：
本项目的 mudlib 在 `master.c` 第 14 行写着

```c
if (port == 5003) { set_encoding("GBK"); }
```

而 §039 已经说过它为什么没炸——`wasm_console_connect()`
把连接标成来自第一个 `external_port`，也就是 UTF-8 的 5001。
**两个独立的限制刚好对消了**，而这种「刚好没事」必须被写下来，
否则有一天有人调换两个端口的顺序，会得到一个非常难查的错误。

**其他已知落差**（上游明列，本项目逐条确认过）：

| 落差 | 具体症状 |
|------|---------|
| **没有 eval limit** | LPC 里 `while(1);` 卡死整个分页（§037） |
| 没有真 DNS | `query_ip_number()` 一律回 `127.0.0.1`；`resolve()` 下一拍合成回传 |
| 没有 zlib | 压缩的 `write_file`（flag 2）报错；压缩的 `save_object` 退化成纯文本存盘；`.gz` 不会被自动解压 |
| MEMFS 不落地 | 重整即失（持久化是上游的下一阶段：IDBFS／OPFS 叠在 `/data` 上） |
| 分页在背景时计时器被暂停 | 醒来时 gametick 补跑，**上限 100 拍** |
| `crypt()` | 属 core，**不受 crypto 套件关闭影响**；只会看到 `old crypt() password detected` 警告 |

最后一列是本项目特地去确认的（`libs/lpmudname/NOTES.md` 的「没有踩到的坑」）：
`crypt()` 是登录流程的关键（§044 会看到握手行就是它算出来的），
如果它跟着 crypto 套件一起消失，17 个 mud 一个都登不进去。
**它没有消失，因为它住在 core 而不是 crypto**（`src/packages/core/core.spec:191`）。
这种「凭什么没事」的确认，和「这里坏了」一样值得记录。

**优点 / 罩门**：优点是**体积**——`fluffos.wasm` 约 3.6 MB 原始大小，
过线约 0.8 MB（brotli）／1.1 MB（gzip），加约 110 KB 的 JS glue。
一台完整的 MUD driver 用不到一 MB 就送到浏览器里。
罩门是**落差清单会随版本改变**，而 mudlib 不会知道。
本项目的缓解方式是把落差变成建置产物：
每个 lib 的 `NOTES.md` 都由 `boot-test.mjs` 实测后写回，
driver 升版就重跑一次——**落差表不是文档，是测试结果**。

**效益**：这张矩阵把「这个 mud 为什么有些功能不见了」
从一个要现场调试的问题，变成一个查表就能回答的问题。
17 个 lib 导入时，绝大多数「这个 daemon 为什么载不起来」
都可以在十秒内对应到矩阵上的某一格。

> 💡 君之一席话
> **移植的成品清单有两份：能跑的东西，以及不见了的东西**——只交前者的人，会在半年后被人问「为什么跨服聊天没有反应」，而那时已经没有人记得答案。

> 🔍 老手视角──真正的门道
> 这一节最该学的是**落差的记录方式**。多数项目把「已知限制」写成 README 里的一段话，三个月后它就过期了——因为没有东西会在它变错的时候告诉你。本项目的做法是把它变成**建置产物**：`boot-test.mjs` 实跑一次注册建角，把加载失败的对象、收到的 opcode、最终分级写回每个 lib 的 `NOTES.md`，并在文件里标明「由建置产生，勿手改」。这个转换的价值不在自动化省了多少时间，在于**它让落差清单拥有了一个过期机制**：driver 升版、mudlib 改坏，下一次建置就自己降级。老手看一份「已知限制」文档，第一个问的是「它上次被验证是什么时候」；答不出来的，就当它不存在。可落地的做法：任何长期文档里的事实性断言，想办法让它由一支会失败的脚本产生——**能自己出错的文档，才是活的**。

---

## 042　五个导出、两个回呼，以及三个只有这里才有的陷阱

**标签**：`#导出接口` `#重入` `#时钟` `#静默失败` `#闭环验证`
**证据等级**：🟢 本项目实测（三个陷阱各有 before/after）＋ 🟡 上游 `src/wasm/README.md` §4

**起源**：接缝抽好了、套件选好了，剩下的就是**页面怎么开这台 driver**。
接口小得惊人：五个导出函数、两个回呼。

**技术内核**：上游文档给的完整形状——

```js
const M = await createFluffOS({ print, printErr, locateFile });
M.FS.chdir('/testsuite');                       // mudlib 挂载点
M.fluffos = {
  onOutput:     (id, bytes) => {...},           // server → client 的线路字节
  onDisconnect: (id)        => {...},
};
M.ccall('fluffos_boot',    'number', ['string'], ['etc/config.test']);
setInterval(() => M.ccall('fluffos_tick', 'number', ['number'],
                          [performance.now()]), 50);
const id = M.ccall('fluffos_connect', 'number', [], []);
M.ccall('fluffos_input', null, ['number','array','number'], [id, bytes, n]);
// 另外还有：fluffos_flag（对应 master::flag）、fluffos_disconnect、fluffos_shutdown
```

| 导出／回呼 | 对应原生的什么 |
|-----------|--------------|
| `fluffos_boot(config)` | `main()` 读设置、加载 master、preload |
| `fluffos_tick(now_ms)` | libevent 的一圈循环 |
| `fluffos_connect()` | accept 一条连接（`user_add` → 协商 → `master::connect` → `logon`） |
| `fluffos_input(id, bytes, n)` | 一次 socket read |
| `fluffos_disconnect(id)` / `fluffos_shutdown()` | 关连接／关机 |
| `onOutput(id, bytes)` | 一次 socket write |
| `onDisconnect(id)` | 对端消失 |

`fluffos_flag` 对应 master 的 `flag()` apply（上游用它跑 LPC 测试套件）。

**还有 `jsbridge`**：只有 WASM 平台才有的套件，让 LPC 反过来调用页面——
`js_eval("navigator.userAgent")`、`js_call("fetch_json", ({url}), (: cb :))`，
以及 `js_export("inventory", (: ui_inventory :))` 让页面 UI 直接触发 LPC。
**本项目没有用它**：ZJMUD 客户端的设计前提是「同一份前端同时服务桌面版、桥接版与 WASM 版」，
用了 `jsbridge` 就会多出一条只有 WASM 才有的路径，
而本书第四篇整篇都在说为什么不要那样做。**记在这里是为了说清楚那条没走的路。**

**踩过的坑：三个陷阱，全部只有「服务器在同一个分页里」才会出现。**

### 陷阱一：不能在 `onOutput` 里调用 `fluffos_input`

`fluffos_input()` **会在自己回传之前就产生输出**，
而 `onOutput` 是 driver **在自己的输出路径中途**回呼页面的。
此时再调用 `fluffos_input`，等于在 telnet 解析器还没吐完字节时重入 driver。

上游与官方参考前端都把这件事写在注解里：**“sends must be queued”**。
所以正解不是「延迟一点送」，是**一律排队、只在 tick 的堆栈上送**。

本项目的实测 before/after：直接在回呼里回复 → 服务器只回 2 行、卡在 `authing` 30 秒；
改成排队 → 一路走到 `ESC000 0008`（要求建角）。

### 陷阱二：`fluffos_connect()` 在回传之前就开始回呼

```js
connId = M.ccall('fluffos_connect', 'number', [], []);
//        └─ 这一行还没赋值完，logon 的输出已经同步回呼进 onOutput 了
```

如果 `send()` 用 `connId === null` 当作「还没连上」的判准，
那么**握手行触发的第一次回复会被静默丢弃**——
连队列都没进去，`rawSend` 一次都没被调用。

本项目在这里连续误判两次（先怪时序、再加静置窗），
真正找到它的方法只有一个：**在 `rawSend` 印一行，看它到底有没有被调用**。
修法是把「已关闭」变成一个独立的旗标，并且**在 `ccall` 之前**就设成 false：

```js
closed = false;                                   // ★ 必须在 ccall 之前
connId = M.ccall('fluffos_connect', 'number', [], []);
```

§032 已经把这个故事写过一次；这里补的是它的**通则**——
**同步回呼会在你以为的初始化完成之前抵达，
所以「初始化完成」不能用「赋值语句已运行」来表示。**

### 陷阱三：`fluffos_tick` 要的是「经过多久」，不是「现在几点」

这一个 §032 没写，因为它是后来才浮出来的，而且**只有在真浏览器里才会踩到**。

症状：`91书剑` 一连上就收到「您花在连线进入手续的时间太久了」。
那是 `clone/user/login.c` 里的 `call_out("time_check", 30)`——理应 30 秒后才跑。

| 版本 | 送进 `fluffos_tick` 的值 | 结果 |
|------|------------------------|------|
| 第一版 | `Date.now()`（epoch 毫秒） | driver 第一拍就认为过了 **1.7 兆毫秒** |
| 第二版 | `performance.now()` | 仍然错，只是小一点——那是**分页加载**起算的，而用户先看清单、再等 29 MB 下载完才开机，开机时它已经是几十万毫秒 |
| 第三版 | `monotonicNow() - clockOrigin`，`clockOrigin` 压在**开机那一刻** | ✅ |

**第二版最值得记**：它在 node 测试里完全正常——
因为那边开机就在行程起头，`performance.now()` 还很小。
**这是一个只有在真实使用情境（先看清单、再下载几十 MB）才会显现的差异**，
而本项目的验证回路一开始没有涵盖它。

验证的方式也很直接：把 `performance.now()` 加上 300000 的偏移量重跑，
**逐字重现了用户截屏里的画面**（只收到握手行 + `ESC015` 逾时消息）。
**能重现，才叫找到原因。**

**优点 / 罩门**：优点是接口小到可以完整背下来——
五个函数、两个回呼，没有状态机、没有握手、没有版本协商。
罩门是这个接口**把原本由操作系统与网络提供的保证全部拿掉了**，
而那些保证原本是隐形的：

| 原本谁提供 | 提供了什么保证 | WASM 版谁负责 |
|-----------|--------------|--------------|
| TCP／操作系统 | 送出与收到之间有 RTT，不会零延迟重入 | **页面**（排队 + 静置窗） |
| 内核的 socket 缓冲 | 写入不会同步触发对端的处理 | **页面**（只在 tick 里送） |
| 系统时钟／libevent | 单调、从行程起算的时间源 | **页面**（自己记原点送差值） |
| 操作系统的行程隔离 | mudlib 跑疯不会影响别人 | **没有人**（eval limit 停用，§041） |

第一列解释了为什么本项目在桥接版从来没踩到陷阱一与陷阱二：
**真实 TCP 的 RTT 天然盖掉了那个空窗**。
把服务器搬进分页，等于把这层天然的缓冲拿掉——
**原本被延迟掩盖的竞态，会一次全部现形。**

**效益**：本项目最终把这三个陷阱与所有兼容性修正，
一起固化成一条会自己跑完的验证链：
`node webclient/tools/verify-fullstack.mjs`——真 DOM、真 wasm driver、真 HTTP，
选 mud → 开机 → 连接 → 登录 → 建角 → 进世界 → 换另一台 mud。
**三个陷阱都是静默失败，而对付静默失败的唯一工具就是一条会自己走完全程的路。**

> 💡 君之一席话
> **当你把两个原本分开的东西塞进同一个行程，你拿掉的不只是网络——你拿掉的是「时间」这个天然的同步机制**；原本靠延迟侥幸成立的假设，会在那一刻全部到期。

> 🔍 老手视角──真正的门道
> 三个陷阱表面上是三种 bug（重入、状态判断、时钟来源），但它们有同一个形状：**一个原本由环境提供、从来没有被写下来的保证，被移植拿掉了**。这类问题最难的地方在于你没有东西可以查——没有人会在文档里写「本系统假设送出与收到之间有延迟」，因为在原本的环境里那不是假设，是物理事实。老手处理移植时会刻意做一件事：**把「环境替我做了什么」列出来**，一项一项问「新环境还做吗」。时间、顺序、隔离、原子性、失败模式——这五样是最常被无声拿掉的。可落地的做法：移植的第一份文档不要写「我要改什么」，先写「旧环境保证了什么」；那份清单通常就是你未来三周会踩的坑的目录，而且**写它的成本远低于一个一个踩**。
