# 拾句 — 部署步驟

前端放 GitHub Pages,資料放 Firebase Firestore。全程免費,約 30–60 分鐘。

## 第一步:建立 Firebase 專案(約 15 分鐘)

1. 到 https://console.firebase.google.com → 「新增專案」→ 名稱隨意(例如 `shiju`)→ Google Analytics 可以關掉
2. 專案首頁點 **`</>`(Web)** 圖示註冊應用程式 → 暱稱隨意 → **不要**勾 Firebase Hosting → 註冊
3. 畫面會顯示 `firebaseConfig` 那段程式碼 → **複製起來**,貼進 `index.html` 最上方標示「① 換成你自己的 Firebase 專案設定」的位置

### 開啟 Google 登入
4. 左側選單 **Authentication** → 「開始使用」→ Sign-in method 分頁 → 啟用 **Google** → 填支援電子郵件 → 儲存

### 建立 Firestore 資料庫
5. 左側選單 **Firestore Database** → 「建立資料庫」→ 位置選 `asia-east1`(台灣)→ 選 **正式版模式(production mode)**
6. 建好後點 **規則(Rules)** 分頁 → 全部刪掉,貼上 `firestore.rules` 的內容 → 發布

## 第二步:部署到 GitHub Pages(約 10 分鐘)

1. GitHub 建一個新 repo(例如 `shiju`,Public)
2. 上傳這四個檔案:`index.html`、`manifest.json`、`icon-192.png`、`icon-512.png`
   (`firestore.rules` 和這個 README 不用上傳)
3. Repo → **Settings → Pages** → Source 選 `Deploy from a branch` → Branch 選 `main` / `(root)` → Save
4. 等 1–2 分鐘,網址會是 `https://你的帳號.github.io/shiju/`

### 授權網域(重要,不做會無法登入)
5. 回 Firebase Console → **Authentication → Settings → Authorized domains** → Add domain → 加入 `你的帳號.github.io`

## 第三步:裝到 iPhone / iPad

1. Safari 打開 `https://你的帳號.github.io/shiju/`
2. **先在 Safari 裡完成一次 Google 登入**(確認一切正常)
3. 分享 → 「加入主畫面」→ 桌面就有「拾句」icon,打開是全螢幕、像原生 app
4. iPad 重複同樣步驟,登入同一個 Google 帳號 → 資料即時同步

## 之後怎麼更新

改 `index.html` → push 到 GitHub → 1–2 分鐘後自動生效,兩台裝置重開 app 就是新版。

## 疑難排解

- **點登入沒反應/顯示 unauthorized-domain** → 第二步的第 5 點沒做,去加網域
- **主畫面版登入卡住** → 先回 Safari 分頁登入一次,再開主畫面版(iOS 全螢幕模式偶爾擋登入視窗,程式已內建 redirect 備援)
- **離線能用嗎** → 能。Firestore 有本機快取,沒網路時照樣看和新增,連上網自動同步
