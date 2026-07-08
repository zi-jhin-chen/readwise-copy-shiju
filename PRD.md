# 拾句 (Shiju) — Readwise Clone 產品需求文件 (PRD)

> 給 Claude Code 的實作規格。目標是複製 Readwise 的核心體驗:把各來源的閱讀劃線集中,用間隔重複每天回顧,讓讀過的東西真的記得住。
>
> **既有基礎**:目前已有一個部署在 GitHub Pages + Firebase (Auth + Firestore) 的最小可用版本,含 Google 登入、手動新增劃線、每日回顧(簡易版)、書庫、標籤搜尋、Markdown/JSON 匯出、iOS 分享捷徑擷取。本 PRD 定義如何把它升級成接近真實 Readwise 的產品。
>
> **技術棧限制(不要改)**:純前端 SPA(vanilla JS 或輕量框架皆可)+ Firebase Firestore + Firebase Auth,部署於 GitHub Pages,以 PWA 形式安裝到 iOS/Android。**不引入需要付費方案的 Firebase 服務(如 Cloud Functions 排程),除非該功能明確標示為 P2 且說明替代方案。**

---

## 1. 產品定位

Readwise 的本質不是「稍後讀」,而是**記憶系統**。它解決的問題是:讀完一本書兩週後就忘光了。核心機制是間隔重複 (spaced repetition) 與主動回想 (active recall)——在正確的時間把正確的劃線重新浮現到眼前。

本專案(拾句)複製「Readwise 記憶系統」這一半,**不複製 Reader(稍後讀 app)**。Reader 是一整間公司做了兩年、跨 web/iOS/Android 的獨立產品,含去廣告排版、EPUB 解析、YouTube 逐字稿、RSS,自建不現實。拾句的擷取端改用「iOS 分享捷徑 + 手動輸入 + 批次匯入」達成。

### 一句話定位
> 一個免費、資料自持、跨裝置的個人劃線回顧系統,把 Kindle / 文章 / 實體書的重點集中起來,每天用間隔重複回顧,避免讀完就忘。

---

## 2. 目標使用者

單一使用者(產品擁有者本人),延伸到少數重度閱讀者。特徵:
- 大量閱讀 Kindle、Medium、網頁文章、實體書
- 想把重點沉澱成長期記憶,而非收藏後遺忘
- 願意每天花 5 分鐘做回顧
- 重視資料主權(資料存在自己的 Firebase,可隨時匯出)

非目標使用者:只想收藏文章、從不回顧的人。

---

## 3. 功能需求(依優先級)

### P0 — 核心記憶迴圈(必做,定義產品存在的理由)

#### 3.1 劃線資料模型 (Highlight)
每一則劃線是一個 Firestore document,欄位:
| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | string | 唯一 ID |
| `text` | string | 劃線內容(必填) |
| `note` | string | 使用者個人筆記/註解 |
| `bookId` | string | 所屬來源文件 ID(外鍵至 Book) |
| `tags` | string[] | 標籤 |
| `location` | string \| null | 頁碼/位置(Kindle 匯入時帶入) |
| `url` | string \| null | 原文連結(文章擷取時帶入) |
| `favorited` | boolean | 是否收藏 |
| `discarded` | boolean | 是否移出回顧輪替(仍保留在書庫) |
| `createdAt` | timestamp | 建立時間 |
| **間隔重複狀態(見 §4)** | | |
| `srs.halfLife` | number | 目前回想機率半衰期(天) |
| `srs.lastReviewed` | timestamp \| null | 上次回顧時間 |
| `srs.reviewCount` | number | 累計回顧次數 |

#### 3.2 來源文件模型 (Book / Document)
劃線依「來源」分組。一個 Book 代表一本書或一篇文章:
| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | string | 唯一 ID |
| `title` | string | 書名/文章標題(必填) |
| `author` | string | 作者 |
| `category` | enum | `book` \| `article` \| `tweet` \| `podcast` \| `supplemental` |
| `coverUrl` | string \| null | 封面圖 |
| `sourceUrl` | string \| null | 原始網址 |
| `frequency` | number | 頻率調整倍率,預設 1.0(見 §4.4) |
| `highlightCount` | number | 劃線數(冗餘欄位,方便顯示) |

#### 3.3 每日回顧 (Daily Review) — **產品的心臟**
- 每天產生一組固定的回顧卡片(預設 N=15,可在設定調整,Readwise 預設範圍約 3–50)。
- 選卡邏輯**必須**依 §4 的間隔重複演算法,而非目前的簡易「回顧次數少者優先」。
- 卡片正面顯示劃線內容 + 來源(書名/作者)。
- 每張卡片可執行的動作:
  - **標記已讀(繼續下一張)**:更新該劃線的 SRS 狀態(半衰期依演算法拉長)。
  - **收藏 (favorite)**:加入收藏,不影響 SRS。
  - **移出輪替 (discard)**:`discarded=true`,之後不再出現在回顧。
  - **編輯**:修改劃線內容或筆記。
  - **調整此來源頻率**:tune up / tune down(見 §4.4)。
  - **前往原文**:若有 `url`。
- 回顧進度需**跨裝置與跨 session 保留**:做到第 7 張關掉 app,重開仍從第 7 張繼續(存於當日 review session 文件)。
- 全部回顧完 → 顯示完成畫面 + 連續天數 (streak)。

#### 3.4 連續天數 (Streak)
- 記錄每天是否完成當日回顧,計算連續完成天數。
- 提供回顧歷史(哪些天完成過),類似 Readwise 的火焰圖示點開後的日曆。
- 資料存於 `meta/streak` 文件:`{ history: { 'YYYY-MM-DD': true }, ... }`,只保留最近 365 天。

#### 3.5 書庫 (Library)
- 依來源 (Book) 分組列出所有劃線。
- 每個 Book 顯示:封面、標題、作者、劃線數、分類。
- 點入 Book → 看該來源所有劃線。
- 全文搜尋:跨劃線內容、筆記、書名、作者、標籤。
- 劃線層級操作:編輯、刪除、收藏、加標籤、移出輪替。

#### 3.6 手動新增 + iOS 捷徑擷取(已有,保留)
- 表單新增單則劃線(內容 / 來源 / 作者 / 筆記 / 標籤)。
- 透過 URL 參數 (`?quote=&source=&url=`) 由 iOS 分享捷徑帶入預填。

---

### P1 — 匯入與匯出(讓產品真正可用,而非空殼)

#### 3.7 Kindle 匯入 ★ 高價值
- Readwise 最多人用的功能就是把多年 Kindle 劃線倒進來。
- **方案 A(必做)**:解析 Kindle 的 `My Clippings.txt`。使用者用 USB 或郵件把該檔從 Kindle 取出,在拾句上傳,批次解析成 Highlights + Books。格式為固定的分隔區塊(標題行、位置/時間行、空行、內文、`==========` 分隔),寫一個 parser 即可,純前端可完成。
- 解析時自動去重(同內容同位置不重複匯入)。
- **方案 B(P2)**:串接 Amazon Kindle Notebook 網頁 (read.amazon.com/notebook) 自動同步——需要登入 Amazon 抓取,前端做不到,需後端爬蟲,列為未來。

#### 3.8 CSV 匯入 / 匯出
- 匯入:標準欄位 (Highlight, Title, Author, Note, Location, Tags) 的 CSV 批次匯入。
- 匯出:全庫匯出成 CSV。
- 保留現有的 Markdown(依來源分組)與 JSON 全備份匯出。

#### 3.9 Apple Books 匯入(P1 或 P2,視難度)
- Apple Books 劃線存在本機 SQLite,無官方匯出。實務作法是使用者用第三方腳本匯出成 CSV/Markdown 再匯入拾句 → 併入 §3.8 CSV 匯入即可,不需另做。

#### 3.10 實體書 OCR 拍照擷取(P2)
- Readwise 招牌功能:拍書頁 → 手指劃選 → 存成劃線。
- 前端可用瀏覽器相機 + 一個 OCR 服務(如 Tesseract.js 純前端,或呼叫雲端 OCR)。
- 純前端 Tesseract.js 對中文辨識品質有限,列 P2,先確認核心迴圈跑順再說。

---

### P2 — 進階與加分(接近真實 Readwise 但非必要)

#### 3.11 主動回想 (Active Recall) — Mastery
Readwise 的 Mastery 讓回顧不只是「看過」,而是「考自己」:
- **Q&A 模式**:把劃線變成問答卡,正面問題、背面答案。
- **Cloze deletion(克漏字)**:遮住劃線中的關鍵字,讓使用者回想填空。
- 回想後由使用者評分「多久後再看」,回饋進 SRS。
- 這是把拾句從「被動重讀」升級成「主動記憶」的關鍵,但實作複雜(要標記答案/挖空),列 P2。

#### 3.12 主題回顧 (Themed Review)
- 除了每日回顧,可自訂主題:選定某些書/標籤組成一個獨立的回顧集,依自訂頻率(每週某幾天/每月一次)浮現。
- 例:一個「設計哲學」主題只抽特定標籤的劃線。

#### 3.13 每日 email 摘要(P2,有技術限制)
- Readwise 核心體驗之一是每天早上一封 email 把當日劃線寄來。
- **限制**:需要伺服器端排程,Firebase 免費方案 (Spark) 不支援 Cloud Functions 排程。
- **替代方案**:(a) 用 iOS「捷徑」的個人自動化,每天早上定時自動開啟拾句回顧頁;(b) PWA 推播通知(需 service worker + Firebase Cloud Messaging,設定較繁);(c) 若願升級 Firebase Blaze 方案(用量小仍近乎免費)可用 Cloud Scheduler + Functions 寄信。預設走 (a)。

#### 3.14 與筆記工具整合匯出(P2)
- Readwise 賣點是自動把劃線同步到 Notion / Obsidian / Roam。
- 拾句可做:一鍵匯出成 Obsidian 友善的 Markdown 檔(每本書一個 .md,含 callout 格式);或透過 Notion API 寫入指定 database(使用者已有 Notion 使用習慣)。

#### 3.15 首頁小工具 / streak 火焰圖示(P2)
- iOS 主畫面 widget 顯示今日回顧待辦數(需 Scriptable,非 PWA 能力)。
- App 內顯示 streak 火焰數字 + 點開日曆。

---

## 4. 間隔重複演算法規格 (Spaced Repetition — 核心 IP)

> **這是產品的核心,不能用「回顧次數少者優先」草草帶過。** 以下規格參考 Readwise 公開說明的機制:採用**以回想機率半衰期為基礎的衰減演算法**,而非固定日期式排程 (如 Anki 的 SM-2)。

### 4.1 基本模型
每則劃線有一個「回想機率」`p`,隨時間衰減:

```
p(t) = 2 ^ ( -elapsedDays / halfLife )
```

- `elapsedDays` = 距上次回顧的天數。
- `halfLife` = 該劃線目前的半衰期(天)。半衰期越長,代表越熟、衰減越慢。
- 新劃線給一個初始 `halfLife`(例如 0.5–1 天),使其很快進入回顧。

### 4.2 每日選卡邏輯
1. 計算每則未 `discarded` 劃線的當前回想機率 `p(t)`。
2. **`p` 越低者越該複習**(快忘了)→ 以 `(1 - p)` 作為抽選權重。
3. 依權重加權隨機抽出 N 張(N = 使用者設定的每日回顧量),避免每天完全相同也避免純隨機。
4. 加入 §4.4 的來源頻率倍率 `book.frequency` 作為權重乘數。
5. (選)每日固定亂數種子,確保同一天多裝置看到同一組卡片。

### 4.3 回顧後更新半衰期
使用者「標記已讀」該劃線時:
- 視為一次成功回想 → **拉長半衰期**:`halfLife = halfLife * growthFactor`(例如 growthFactor ≈ 2.0，即每複習一次熟悉度加倍)。
- 更新 `lastReviewed = now`、`reviewCount += 1`。
- (P2, 搭配 Active Recall)若使用者答錯/選「很快再看」→ **縮短半衰期**:`halfLife = max(minHalfLife, halfLife * 0.5)`。
- 每次顯示後,該劃線短期內再次出現的機率應顯著下降(因半衰期剛被拉長),但不保證看完全部才輪第二次——符合 Readwise 描述。

### 4.4 頻率調整 (Frequency Tuning)
- 劃線多的來源會過度洗版:若總庫 500 則、某書 100 則,該書每日出現機率約 20%。
- 每個 Book 有 `frequency` 倍率(預設 1.0)。使用者可在回顧卡片或書庫 tune up(如 ×2)/ tune down(如 ×0.5)。
- 選卡權重 = `(1 - p) * book.frequency`。

### 4.5 劃線品質過濾 (Highlight Quality Filter)(P2)
- 過濾「低品質」劃線(過短的句子片段)不進每日回顧。
- 簡單規則:字數過少 (如 < 10 字) 或無句尾標點者,預設排除,可由使用者覆寫。

---

## 5. 資訊架構與畫面

```
拾句 (PWA)
├── 登入頁 (Google Sign-In)
├── 每日回顧 (Daily Review)        ← 預設著陸頁,產品心臟
│   ├── 回顧卡片(逐張)
│   │   └── 動作:已讀 / 收藏 / 移出 / 編輯 / 調頻 / 前往原文
│   └── 完成畫面 + streak
├── 書庫 (Library)
│   ├── 依來源分組列表
│   ├── 搜尋
│   └── Book 詳情 → 該來源劃線
├── 新增 / 擷取 (Add)
│   ├── 手動表單
│   └── iOS 捷徑帶入 (URL 參數)
├── 匯入 (Import)
│   ├── Kindle My Clippings.txt
│   └── CSV
├── 匯出 (Export)
│   ├── Markdown / CSV / JSON
│   └── (P2) Obsidian / Notion
└── 設定 (Settings)
    ├── 每日回顧數量 N
    ├── SRS 參數(進階)
    ├── 品質過濾開關
    └── 帳號 / 登出
```

---

## 6. 技術架構

- **前端**:單頁應用,純 vanilla JS(延續現有)或可選 Vite + 輕量框架;但**必須維持可直接部署到 GitHub Pages 的靜態輸出**。
- **認證**:Firebase Auth,Google 登入。
- **資料庫**:Firestore,路徑結構:
  ```
  users/{uid}/books/{bookId}
  users/{uid}/highlights/{highlightId}
  users/{uid}/reviewSessions/{YYYY-MM-DD}   // 當日回顧進度與選卡
  users/{uid}/meta/state                     // streak、設定
  ```
- **安全規則**:每人只能讀寫自己 `users/{uid}` 底下資料(現有規則沿用)。
- **離線**:啟用 Firestore persistent cache,離線可讀可寫,連網自動同步(現有已具備)。
- **PWA**:manifest + apple-touch-icon,支援「加入主畫面」全螢幕。加 service worker 以支援離線殼與(P2)推播。
- **擷取**:iOS 捷徑透過 URL 參數帶入;不做瀏覽器擴充(那屬於 Reader 範疇)。

### 效能與資料量考量
- 一個重度使用者可能有數萬則劃線。**不要一次把全部 highlights 載進前端**做選卡。
- 每日選卡:Firestore 無法直接做加權隨機。策略:
  - 維護每則劃線的 `srs.dueScore`(可預先算的排序鍵,如下次應複習時間),用 Firestore query `orderBy(dueScore).limit(N * k)` 取出候選池,再在前端做加權抽樣。
  - 或在 highlight 上存 `nextReviewAt` 時間戳,query `where nextReviewAt <= today`。
- 書庫列表分頁載入 (pagination),避免一次拉全部。

---

## 7. 分階段路線圖(給 Claude Code 的實作順序)

**Phase 1 — 把心臟換掉(升級現有 MVP)**
1. 把資料模型從「單層 highlights」重構成 `books` + `highlights` 兩個 collection。
2. 實作 §4 的間隔重複演算法(§4.1–4.4),取代現有簡易選卡。
3. 每日回顧加入完整卡片動作(收藏/移出/編輯/調頻)。
4. 回顧進度存 `reviewSessions/{date}`,跨裝置續讀。
5. Streak 計算 + 歷史日曆。

**Phase 2 — 讓它裝得下真實資料**
6. Kindle `My Clippings.txt` parser + 匯入 UI + 去重。
7. CSV 匯入/匯出。
8. 書庫分頁 + Book 詳情頁 + 頻率調整入口。
9. 設定頁(每日數量、品質過濾、SRS 進階參數)。

**Phase 3 — 接近真實 Readwise**
10. Active Recall(Q&A / cloze) — Mastery。
11. 主題回顧 (Themed Review)。
12. Obsidian / Notion 匯出整合。
13. 每日提醒(iOS 捷徑自動化 or FCM 推播)。
14. 實體書 OCR 拍照擷取。

---

## 8. 明確排除範圍 (Out of Scope)

以下屬於 Readwise Reader 或需要大型後端,本專案**不做**:
- 稍後讀 (read-it-later):存文章、去廣告排版、閱讀器 UI。
- EPUB / PDF 解析與內建閱讀。
- YouTube 逐字稿擷取、RSS 訂閱、Twitter thread 編排。
- 瀏覽器擴充 web clipper。
- Ghostreader / Chat with Highlights 等 LLM 功能(除非另立專案接 API)。
- 自動從 Amazon/Instapaper/Pocket 帳號雲端同步(需後端爬蟲/OAuth)。

---

## 9. 驗收標準 (Definition of Done — Phase 1)

- [ ] 匯入 ≥ 200 則測試劃線後,每日回顧能依 SRS 演算法選出一組合理的卡片(快忘的優先、剛看過的短期不重複、劃線多的來源不洗版)。
- [ ] 回顧到一半關閉 app,換另一台裝置登入同帳號,能從同一張卡續讀。
- [ ] 完成當日回顧後 streak +1,連續兩天完成顯示 streak = 2。
- [ ] tune down 某來源後,該來源劃線在往後回顧出現頻率明顯下降。
- [ ] 全程可離線操作,連網後資料同步無衝突。
- [ ] 部署於 GitHub Pages,iOS Safari 加入主畫面後全螢幕可用。

---

## 附錄 A — Kindle My Clippings.txt 格式範例(供 parser 參考)

```
書名 (作者名)
- Your Highlight on page 12 | Location 176-177 | Added on ...

這裡是劃線的內文。
==========
```
每個區塊以 `==========` 分隔。第一行為「標題 (作者)」,第二行為 metadata(含 page / location / 時間),空行後為內文。註記類型可能是 Highlight / Note / Bookmark,parser 需分別處理(Bookmark 無內文可略過,Note 併入對應 highlight 的 note 或獨立存)。
