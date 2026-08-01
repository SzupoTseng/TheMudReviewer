# 附录 B　文件地图与接手指南

> 给下一个要动这份代码的人。**先读这页，再读代码。**

---

## B.1　文件地图

```
zjmud-rust/
├── START_SERVER.bat      只启动 MUD 服务器
├── START_DESKTOP.bat     服务器 + 桌面版（Tauri）
├── START_WEB.bat         浏览器版桥接（手机也能连）
├── STOP_ZJMUD.bat        停止全部
├── BUILD_ZJMUD.bat       建置桌面版 release
├── LPMud-Name/world/     MUD 服务器（driver.exe）＝ 第八篇 §043/§044 的原代码来源
├── libs/                 ★ 17 份可在浏览器里跑的 mudlib（第九篇 §050、附录 F）
│   └── <slug>/  mud.json ＋ mudlib.data ＋ mudlib.json ＋ NOTES.md
├── docs/                 三份规格书 ＋ ZJMUD 迁移 SOP
├── designbook/           ← 你正在读的这本
└── webclient/
    ├── src/                        前端（桌面／浏览器／WASM **完全共用**）
    │   ├── index.html              205 行
    │   ├── css/tokens.css          121 行　设计 token + 三套主题
    │   ├── css/app.css           1,044 行　版面与组件
    │   └── js/
    │       ├── ansi.js       233 行　样式方言解析　**纯函数**
    │       ├── protocol.js   602 行　opcode 解析　　**纯函数**
    │       ├── dialects.js   262 行　方言 profile 表
    │       ├── store.js      243 行　状态容器 + pub/sub
    │       ├── net.js        325 行　Transport 抽象（三后端）
    │       ├── telnet.js     138 行　IAC 剥除＋分行（**浏览器与 node 共用**）
    │       ├── ui.js         735 行　DOM 组件
    │       ├── prefs.js       61 行　本地偏好
    │       ├── main.js     1,165 行　组装 + reducer + 登录状态机
    │       ├── wasmdriver.js 263 行　★ WASM driver 绑定（§042 的三个陷阱都在这）
    │       ├── wasmboot.js   149 行　★ 选 mud → 下载映像 → 开机
    │       ├── mudlibimage.js 112 行　★ 映像格式（format 1）的加载端
    │       └── telnetlogin.js 309 行　★ 会话层代打，玩非 zjmud 的 lib（§045）
    ├── src-tauri/
    │   ├── capabilities/default.json   ★ 缺这个 event.listen 会被挡
    │   └── src/
    │       ├── main.rs        55 行　指令注册
    │       ├── mud.rs       238 行　TCP（字节层分行）
    │       └── telnet.rs    162 行　IAC 剥除
    ├── bridge/
    │   ├── server.mjs       165 行　WebSocket ↔ TCP + 静态档
    │   └── telnet.mjs        26 行　Buffer 转接（退化成 telnet.js 的薄壳）
    ├── test/              2,756 行　**202 条 / 12 档**
    ├── wasm/driver/       FluffOS v2026.0729.0（由 fetch-driver.mjs 取得，不进版控）
    └── tools/
        ├── fetch-driver.mjs      取官方 wasm release
        ├── build-site.mjs        打包 libs/ 下每个 mud + 跑开机测试
        ├── boot-test.mjs         ★ 实跑注册→建角→进世界，决定分级（附录 F）
        ├── verify-fullstack.mjs  ★ 全链路：真 DOM + 真 driver + 真 HTTP（§046）
        ├── import-lib.mjs / import-all.mjs / graft-lib.mjs   导入与补件
        ├── pack-lib.mjs / unpack-lib.mjs / fix-image.mjs     映像工具
        └── win/                  截屏与键盘驱动脚本
```

**产品代码 6,613 行，测试 2,756 行（42%，202 条）。**

> 这份地图与行数是**撰写当下数出来的**。重数的方法：
> `wc -l webclient/src/js/*.js webclient/src-tauri/src/*.rs …`；
> 测试数 `cd webclient && npm test`（末三行是 `tests / pass / fail`）。

---

## B.2　从哪里开始读

| 你的目的 | 读这几个档，照顺序 |
|----------|-------------------|
| 理解协议 | `dialects.js` → `protocol.js` → `ansi.js` |
| 改 UI | `index.html` → `ui.js` → `app.css` |
| 改连接行为 | `net.js` → `src-tauri/src/mud.rs` 或 `bridge/server.mjs` |
| 加新 mudlib 支持 | `dialects.js`（加一张表就好） |
| 查为什么这样写 | 每个文件开头的区块注解，都写了「为什么」而不只是「做什么」 |

---

## B.3　绝对不要做的事

| 别做 | 为什么 |
|------|--------|
| ❌ 在 `protocol.js` / `ansi.js` 里碰 DOM | 它们是纯函数，一碰就失去 100 条免环境测试 |
| ❌ 把所有方言合并成一张 opcode 表 | 602–605 会撞号且**静默覆盖**，见 §010 |
| ❌ 用 `element.hidden` 而不确认 CSS | 作者样式的 `display` 会盖掉它，见 §015 |
| ❌ 从 WSL 跑 `npm install` | 会装到 Linux 版原生 binary，见 §017 |
| ❌ 直接双击 debug 版 exe | 内嵌的是 dev server URL，见 §017 |
| ❌ 在 `.bat` 的 `REM` 注解里写中文 | cmd 预读会误拆成指令，见 §017 |
| ❌ 静默丢弃未知 opcode | 新 mudlib 的内容会凭空消失且无从察觉 |
| ❌ 直接 `innerHTML` 渲染服务器内容 | 指游方言的 HTML 标记可带任意标签，见 §007 |

---

## B.4　改动后的验证流程

```bash
# 1. 全套测试（约 14 秒）
npm test

# 2. 若改了 UI，看一眼实机画面
powershell -File tools\win\shot.ps1

# 3. 若改了协议，对真服务器跑探针
node tools/live-login-probe.mjs <主机IP> 5001 24
```

**第 2 步不可省。** jsdom 不做完整 CSS 级联，所有涉及显示／隐藏或版面的改动，纯 DOM 断言都可能通过而画面是错的。

---

## B.5　已知未完成

| 项目 | 状态 | 难点 |
|------|------|------|
| HTML 式行内标记（指游 web 版） | 未实作 | 需白名单标签与属性，不可直接 `innerHTML` |
| 语音录制／播放 | 留接口未实作 | 依赖 AMR 与外部端点，原版 URL 是占位值 |
| 帐号 HTTP API | 未实作 | 原版端点是 `127.0.0.1` 占位值；且签章 salt 明文，需服务器端重做 |
| 属性条（`ESC012`）实机验证 | 未验证 | 测试角色尚未投胎，服务器不送这个封包 |
| 官方完整验证 | 无法实作 | `ZJKEY` 只在服务器端，第三方客户端拿不到 |

---

## B.6　三个最容易再犯的错

**① 把「编译成功」当成「能用」。**
本项目连续四次交付不能用的东西，每一次都通过了编译与当时的全部测试。

**② 把「已知限制」当成免责声明。**
逐条问：「用户能不能在没有这个功能的情况下，完成一次完整的内核任务？」——不能的话它就是 bug，不是限制。

**③ 错误不可见就开始猜。**
先花二十分钟把错误变得可见，再开始找 bug。本项目的 Tauri ACL 问题，是加了错误显示栏之后**它自己把答案印出来的**。

---

> 💡 君之一席话
> **接手一份逆向项目时，最该先读的不是代码，而是前人踩过的坑——因为代码只告诉你「现在长这样」，坑才告诉你「为什么不能改成那样」。**

> 🔍 老手视角──真正的门道
> 这份接手指南里最有价值的其实是 B.3 那张「绝对不要做的事」——它把八个曾经真实发生的错误，压缩成八条可以在 code review 时直接引用的规则。老手交接项目时会刻意产出这种**负面清单**，因为正面文档（架构图、API 说明）新人自己读代码也能推出来，而负面知识**只存在于踩过的人脑里**，不写下来就永久遗失。可落地的做法：每修好一个花掉你半天以上的 bug，就在负面清单加一行，格式固定为「别做什么 → 为什么 → 详见哪一节」。一年后这张表会比任何架构文档都值钱。
