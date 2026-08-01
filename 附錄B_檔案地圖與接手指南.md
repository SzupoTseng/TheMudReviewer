# 附錄 B　檔案地圖與接手指南

> 給下一個要動這份程式碼的人。**先讀這頁，再讀程式碼。**

---

## B.1　檔案地圖

```
zjmud-rust/
├── START_SERVER.bat      只啟動 MUD 伺服器
├── START_DESKTOP.bat     伺服器 + 桌面版（Tauri）
├── START_WEB.bat         瀏覽器版橋接（手機也能連）
├── STOP_ZJMUD.bat        停止全部
├── BUILD_ZJMUD.bat       建置桌面版 release
├── LPMud-Name/world/     MUD 伺服器（driver.exe）＝ 第八篇 §043/§044 的原始碼來源
├── libs/                 ★ 17 份可在瀏覽器裡跑的 mudlib（第九篇 §050、附錄 F）
│   └── <slug>/  mud.json ＋ mudlib.data ＋ mudlib.json ＋ NOTES.md
├── docs/                 三份規格書 ＋ ZJMUD 遷移 SOP
├── designbook/           ← 你正在讀的這本
└── webclient/
    ├── src/                        前端（桌面／瀏覽器／WASM **完全共用**）
    │   ├── index.html              205 行
    │   ├── css/tokens.css          121 行　設計 token + 三套主題
    │   ├── css/app.css           1,044 行　版面與元件
    │   └── js/
    │       ├── ansi.js       233 行　樣式方言解析　**純函式**
    │       ├── protocol.js   602 行　opcode 解析　　**純函式**
    │       ├── dialects.js   262 行　方言 profile 表
    │       ├── store.js      243 行　狀態容器 + pub/sub
    │       ├── net.js        325 行　Transport 抽象（三後端）
    │       ├── telnet.js     138 行　IAC 剝除＋分行（**瀏覽器與 node 共用**）
    │       ├── ui.js         735 行　DOM 元件
    │       ├── prefs.js       61 行　本地偏好
    │       ├── main.js     1,165 行　組裝 + reducer + 登入狀態機
    │       ├── wasmdriver.js 263 行　★ WASM driver 綁定（§042 的三個陷阱都在這）
    │       ├── wasmboot.js   149 行　★ 選 mud → 下載映像 → 開機
    │       ├── mudlibimage.js 112 行　★ 映像格式（format 1）的載入端
    │       └── telnetlogin.js 309 行　★ 會話層代打，玩非 zjmud 的 lib（§045）
    ├── src-tauri/
    │   ├── capabilities/default.json   ★ 缺這個 event.listen 會被擋
    │   └── src/
    │       ├── main.rs        55 行　指令註冊
    │       ├── mud.rs       238 行　TCP（位元組層分行）
    │       └── telnet.rs    162 行　IAC 剝除
    ├── bridge/
    │   ├── server.mjs       165 行　WebSocket ↔ TCP + 靜態檔
    │   └── telnet.mjs        26 行　Buffer 轉接（退化成 telnet.js 的薄殼）
    ├── test/              2,756 行　**202 條 / 12 檔**
    ├── wasm/driver/       FluffOS v2026.0729.0（由 fetch-driver.mjs 取得，不進版控）
    └── tools/
        ├── fetch-driver.mjs      取官方 wasm release
        ├── build-site.mjs        打包 libs/ 下每個 mud + 跑開機測試
        ├── boot-test.mjs         ★ 實跑註冊→建角→進世界，決定分級（附錄 F）
        ├── verify-fullstack.mjs  ★ 全鏈路：真 DOM + 真 driver + 真 HTTP（§046）
        ├── import-lib.mjs / import-all.mjs / graft-lib.mjs   匯入與補件
        ├── pack-lib.mjs / unpack-lib.mjs / fix-image.mjs     映像工具
        └── win/                  截圖與鍵盤驅動腳本
```

**產品程式碼 6,613 行，測試 2,756 行（42%，202 條）。**

> 這份地圖與行數是**撰寫當下數出來的**。重數的方法：
> `wc -l webclient/src/js/*.js webclient/src-tauri/src/*.rs …`；
> 測試數 `cd webclient && npm test`（末三行是 `tests / pass / fail`）。

---

## B.2　從哪裡開始讀

| 你的目的 | 讀這幾個檔，照順序 |
|----------|-------------------|
| 理解協議 | `dialects.js` → `protocol.js` → `ansi.js` |
| 改 UI | `index.html` → `ui.js` → `app.css` |
| 改連線行為 | `net.js` → `src-tauri/src/mud.rs` 或 `bridge/server.mjs` |
| 加新 mudlib 支援 | `dialects.js`（加一張表就好） |
| 查為什麼這樣寫 | 每個檔案開頭的區塊註解，都寫了「為什麼」而不只是「做什麼」 |

---

## B.3　絕對不要做的事

| 別做 | 為什麼 |
|------|--------|
| ❌ 在 `protocol.js` / `ansi.js` 裡碰 DOM | 它們是純函式，一碰就失去 100 條免環境測試 |
| ❌ 把所有方言合併成一張 opcode 表 | 602–605 會撞號且**靜默覆蓋**，見 §010 |
| ❌ 用 `element.hidden` 而不確認 CSS | 作者樣式的 `display` 會蓋掉它，見 §015 |
| ❌ 從 WSL 跑 `npm install` | 會裝到 Linux 版原生 binary，見 §017 |
| ❌ 直接雙擊 debug 版 exe | 內嵌的是 dev server URL，見 §017 |
| ❌ 在 `.bat` 的 `REM` 註解裡寫中文 | cmd 預讀會誤拆成指令，見 §017 |
| ❌ 靜默丟棄未知 opcode | 新 mudlib 的內容會憑空消失且無從察覺 |
| ❌ 直接 `innerHTML` 渲染伺服器內容 | 指游方言的 HTML 標記可帶任意標籤，見 §007 |

---

## B.4　改動後的驗證流程

```bash
# 1. 全套測試（約 14 秒）
npm test

# 2. 若改了 UI，看一眼實機畫面
powershell -File tools\win\shot.ps1

# 3. 若改了協議，對真伺服器跑探針
node tools/live-login-probe.mjs <主機IP> 5001 24
```

**第 2 步不可省。** jsdom 不做完整 CSS 級聯，所有涉及顯示／隱藏或版面的改動，純 DOM 斷言都可能通過而畫面是錯的。

---

## B.5　已知未完成

| 項目 | 狀態 | 難點 |
|------|------|------|
| HTML 式行內標記（指游 web 版） | 未實作 | 需白名單標籤與屬性，不可直接 `innerHTML` |
| 語音錄製／播放 | 留介面未實作 | 依賴 AMR 與外部端點，原版 URL 是佔位值 |
| 帳號 HTTP API | 未實作 | 原版端點是 `127.0.0.1` 佔位值；且簽章 salt 明文，需伺服器端重做 |
| 屬性條（`ESC012`）實機驗證 | 未驗證 | 測試角色尚未投胎，伺服器不送這個封包 |
| 官方完整驗證 | 無法實作 | `ZJKEY` 只在伺服器端，第三方客戶端拿不到 |

---

## B.6　三個最容易再犯的錯

**① 把「編譯成功」當成「能用」。**
本專案連續四次交付不能用的東西，每一次都通過了編譯與當時的全部測試。

**② 把「已知限制」當成免責聲明。**
逐條問：「使用者能不能在沒有這個功能的情況下，完成一次完整的核心任務？」——不能的話它就是 bug，不是限制。

**③ 錯誤不可見就開始猜。**
先花二十分鐘把錯誤變得可見，再開始找 bug。本專案的 Tauri ACL 問題，是加了錯誤顯示欄之後**它自己把答案印出來的**。

---

> 💡 君之一席話
> **接手一份逆向專案時，最該先讀的不是程式碼，而是前人踩過的坑——因為程式碼只告訴你「現在長這樣」，坑才告訴你「為什麼不能改成那樣」。**

> 🔍 老手視角──真正的門道
> 這份接手指南裡最有價值的其實是 B.3 那張「絕對不要做的事」——它把八個曾經真實發生的錯誤，壓縮成八條可以在 code review 時直接引用的規則。老手交接專案時會刻意產出這種**負面清單**，因為正面文件（架構圖、API 說明）新人自己讀程式碼也能推出來，而負面知識**只存在於踩過的人腦裡**，不寫下來就永久遺失。可落地的做法：每修好一個花掉你半天以上的 bug，就在負面清單加一行，格式固定為「別做什麼 → 為什麼 → 詳見哪一節」。一年後這張表會比任何架構文件都值錢。
