# 📘 使用 MkDocs 建立文件網站教學筆記

以下為 MkDocs 搭建網站的完整步驟與指令，適用於 Windows / macOS / Linux。

---

## 📌 步驟一：安裝 MkDocs

### ✅ 安裝 Python（若尚未安裝）

MkDocs 是 Python 撰寫的工具，請先安裝 Python 3.7+  
可以到官方下載：https://www.python.org/downloads/

安裝後請確認版本：

```bash
python --version
```

或

```bash
python3 --version
```

### ✅ 安裝 MkDocs 本體

```bash
pip install mkdocs
```

### ✅ 確認安裝成功

```bash
mkdocs --version
```

---

## 📌 步驟二：建立新專案

### ✅ 建立一個新的 MkDocs 專案

```bash
mkdocs new my-docs
cd my-docs
```

這會產生一個專案結構如下：

```
my-docs/
├── docs/
│   └── index.md        ← 預設首頁內容
└── mkdocs.yml          ← 設定檔
```

---

## 📌 步驟三：啟動本地預覽伺服器

```bash
mkdocs serve
```

接著打開瀏覽器輸入網址：

```
http://127.0.0.1:8000/
```

就能即時預覽你撰寫的 Markdown 文件。

---

## 📌 步驟四：編輯內容

你可以在 `docs/` 目錄下新增你的 Markdown 文件，例如：

```
docs/
├── index.md
├── about/
│   └── license.md
├── user-guide/
│   ├── writing-your-docs.md
│   └── styling-your-docs.md
```

---

## 📌 步驟五：設定導覽列（`mkdocs.yml`）

打開 `mkdocs.yml`，設定導覽列如下：

```yaml
site_name: 我的文件網站

nav:
  - Home: index.md
  - User Guide:
      - Writing your docs: user-guide/writing-your-docs.md
      - Styling your docs: user-guide/styling-your-docs.md
  - About:
      - License: about/license.md
```

---

## 📌 步驟六：建置靜態網站

當你撰寫完成後，使用下列指令生成靜態 HTML 網站：

```bash
mkdocs build
```

這會在目錄中產生 `site/` 資料夾，裡面就是可上傳到 Web 主機的 HTML 靜態網站。

---

## 📌 步驟七：部署到 GitHub Pages（可選）

如果你想把網站部署到 GitHub Pages：

### 步驟 1：初始化 Git 倉庫（如果尚未 init）

```bash
git init
git remote add origin https://github.com/你的帳號/你的repo.git
```

### 步驟 2：部署至 GitHub Pages

```bash
mkdocs gh-deploy
```

這會自動將 `site/` 資料夾內容部署到 GitHub Pages 的 `gh-pages` 分支。

---

## ✅ 完成！

現在你已經完成 MkDocs 網站的搭建與部署，可以將這份筆記加入到你的文件中！
