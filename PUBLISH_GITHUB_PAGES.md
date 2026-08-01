# 發佈到 GitHub Pages

本目錄本身就是一個可直接發佈的靜態網站，**零外部資源**（CSS/JS/圖片全部內嵌或同目錄），
不需要任何 build step、不需要 Jekyll。

## 檔案角色

| 檔案 | 角色 |
|------|------|
| `index.html` | 多語落地頁 = 網站首頁 |
| `泥巴考古學.html` | 繁體全書（側欄目錄，單檔 555 KB） |
| `cn/泥巴考古学.html` | 简体全书 |
| `TheMudReviewer.png` | 封面海報（落地頁引用；書內為 base64 內嵌） |
| `.nojekyll` | **必要**。沒有它 GitHub Pages 會跑 Jekyll，`_build.py` 這類底線開頭的檔會被特殊處理 |

## 發佈步驟

```bash
git remote add origin https://github.com/SzupoTseng/TheMudReviewer.git
git push -u origin main
```

到 repo 的 **Settings → Pages**，Source 選 `Deploy from a branch`，
branch 選 `main`、資料夾選 **`/ (root)`**。

本書是倉庫的根目錄，所以網址是：

| 網址 | 內容 |
|------|------|
| `https://szupotseng.github.io/zjmud-rust/` | 倉庫根的 `index.html`，會自動轉址到下一列 |
| `https://szupotseng.github.io/TheMudReviewer/` | 本書多語落地頁 |

> **`.nojekyll` 要放在「發佈來源的根目錄」，不是放在書的目錄。**
> 本倉庫從 `/ (root)` 發佈，所以根目錄也有一份 `.nojekyll`；
> 少了它，Jekyll 會去解析倉庫裡上千個 `.md` 與 `.c`，建置很可能直接失敗，
> 而失敗的表徵是「Pages 一直沒更新」而不是明確的錯誤訊息。

## 重新建置

改了任何 `.md` 之後：

```bash
python _build.py           # 繁體 → 泥巴考古學.html + _全書.md
python _convert_cn.py      # 繁 → 简，並產生 cn/_build.py
cd cn && python _build.py  # 简体 → 泥巴考古学.html + _全书.md
```

Windows 主控台若是 cp950，跑简体那步要先 `chcp 65001` 且 `set PYTHONIOENCODING=utf-8`，
否則 `print()` 會丟 `UnicodeEncodeError`（**只是印不出來，檔案其實已寫好**）。

工具鏈與版面規範見 `BUILD_STANDARD.md`——**鐵律是只改常數與內容，不要自己另立一套**。
本書相對參考書的差異只有兩處在地化：`title_for()`／`part_for()`，
因為本書一個「篇」跨多個檔（例如第六篇有四檔），H1 是篇名而非章名。
