# 拾句 (Shiju) — Reader 產品需求文件 (PRD)

> 給 Claude Code 的實作規格。
>
> **產品核心**：一個個人用的稍後讀 app。存下網路文章 → 乾淨排版閱讀 → 在文章裡直接劃線 → 劃線自動進入間隔重複的每日回顧。
>
> **既有基礎**：GitHub Pages + Firebase (Auth + Firestore) 的 PWA，已有 Google 登入、手動新增劃線、簡易每日回顧、書庫、搜尋、匯出、iOS 分享捷徑擷取。
>
> **技術棧約束（不要改）**：靜態前端部署於 GitHub Pages；Firebase Auth + Firestore；PWA 安裝到 iOS/Android。唯一例外是**內容擷取服務**，見 §5，必須是獨立的擷取端點（Jina Reader 或自建 Cloud Run proxy）。

---

## 1. 定位

Readwise 是兩個產品共用一個資料庫：
- **Reader**：稍後讀 app，存文章 / PDF / 電子報 / EPUB / YouTube，在裡面劃線。
- **Readwise**：記憶系統，把劃線用間隔重複每天浮現。

本專案做的是**這兩者串起來的那條主線**：Reader 的「存 → 讀 → 劃」加上 Readwise 的「回顧」。不做 Reader 的周邊格式支援（PDF / EPUB / RSS / YouTube）於第一階段。

### 一句話定位
> 存下你想讀的文章，用舒服的排版讀完，把重點劃下來，然後在往後的日子裡一句句想起它。

---

## 2. 範圍界定（誠實版）

| 功能 | 可行性 | 說明 |
|---|---|---|
| 存網頁文章、去廣告排版閱讀 | ✅ 做 | 靠擷取服務把 HTML 轉乾淨內文 |
| 文章內選字劃線、加註解 | ✅ 做 | 核心，見 §4.3 錨定機制 |
| 劃線進入間隔重複回顧 | ✅ 做 | 沿用既有 SRS 設計 |
| 收件匣 / 稍後 / 封存 分類 | ✅ 做 | 單一 state 欄位 |
| 離線閱讀 | ✅ 做 | Firestore 持久化快取 + Service Worker |
| 閱讀進度、跨裝置續讀 | ✅ 做 | 存 scroll 位置 |
| 全文搜尋（跨文章內文） | ⚠️ 有限 | Firestore 無全文索引，見 §6.3 |
| RSS 訂閱 | ⚠️ P2 | 需輪詢，前端可在開啟 app 時透過 proxy 抓 |
| YouTube 逐字稿 | ⚠️ P2 | 需 proxy 抓字幕，CORS 擋住直連 |
| PDF 閱讀與劃線 | ⚠️ P3 | PDF.js 可渲染，但劃線錨定難度高一階 |
| EPUB | ⚠️ P3 | epub.js，同上 |
| Twitter / X 貼文擷取 | ⚠️ 不保證 | oEmbed 端點仍在但**不穩定**，且非 CORS 友善；可試 proxy 抓，失敗就退回手動貼上 |
| Facebook 貼文擷取 | ❌ 不做 | 登入牆 + 積極反爬。Graph API 需審核且已大幅收緊，公開貼文內容基本拿不到。**規劃時直接放棄，用手動貼上** |
| 瀏覽器擴充 web clipper | ❌ 不做 | 改用 iOS 分享捷徑 |
| Ghostreader / AI 對話 | ❌ 不做 | 另立專案 |

---

## 3. 資料模型

### 3.1 Document（一篇文章 / 一本書 / 一個來源）
Firestore: `users/{uid}/documents/{docId}`

| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | string | |
| `title` | string | 文章標題 |
| `author` | string \| null | |
| `sourceUrl` | string \| null | 原始網址 |
| `siteName` | string \| null | 來源站名（Medium、天下…） |
| `coverUrl` | string \| null | 封面圖 |
| `category` | enum | `article` \| `book` \| `tweet` \| `note` \| `manual` |
| `state` | enum | `inbox` \| `later` \| `archive` |
| `wordCount` | number | 估算字數（中文以字元數計） |
| `readingProgress` | number | 0–1，閱讀進度 |
| `lastPosition` | number | 上次讀到的 blockIndex，跨裝置續讀 |
| `frequency` | number | 回顧頻率倍率，預設 1.0 |
| `highlightCount` | number | 冗餘欄位 |
| `savedAt` | timestamp | |
| `extractStatus` | enum | `pending` \| `ok` \| `failed` \| `manual` |

### 3.2 Content（文章內文，獨立文件）
Firestore: `users/{uid}/documents/{docId}/content/body`

| 欄位 | 型別 | 說明 |
|---|---|---|
| `markdown` | string | 擷取後的 markdown 全文 |
| `blocks` | array | 見下 |

**為什麼獨立一個文件**：Firestore 單一文件上限 1 MiB。把內文抽出來，`documents` 的列表查詢才不會每次都拉整篇文章。長文超過 1 MiB 的情況極少見（中文長文約 20–50 KB），若真的超過，切成 `content/body_0`、`content/body_1` 分段。

**`blocks` 的用途與定義**：這是劃線錨定的地基。存檔時把 markdown 解析成一個有序的區塊陣列並**一併存下來**，讓渲染與錨定永遠對得上：

```js
blocks: [
  { i: 0, type: 'h1',   text: '文章標題' },
  { i: 1, type: 'p',    text: '第一段的純文字內容…' },
  { i: 2, type: 'quote',text: '引用段落' },
  { i: 3, type: 'p',    text: '…' },
]
```
- `text` 一律存**純文字**（粗體斜體等行內格式在渲染時另外處理，不進 offset 計算）。
- 一旦存檔，`blocks` **視為不可變 (immutable)**。重新擷取同一篇文章要建新的 doc 或明確走 re-anchor 流程。

### 3.3 Highlight（劃線）
Firestore: `users/{uid}/highlights/{highlightId}`

| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | string | |
| `docId` | string | 所屬文章 |
| `text` | string | 劃線文字（同時作為錨定失效時的比對依據） |
| `note` | string | 個人註解 |
| `tags` | string[] | |
| `color` | enum | `yellow` \| `blue` \| `pink`（預設單色亦可） |
| **錨定（見 §4.3）** | | |
| `blockIndex` | number | 位於第幾個 block |
| `startOffset` | number | block 純文字內的起始字元位移 |
| `endOffset` | number | 結束位移 |
| **間隔重複狀態（見 §7）** | | |
| `srs.halfLife` | number | 回想機率半衰期（天） |
| `srs.lastReviewed` | timestamp \| null | |
| `srs.reviewCount` | number | |
| `favorited` | boolean | |
| `discarded` | boolean | 移出回顧輪替，但保留在文章裡 |
| `createdAt` | timestamp | |

跨 block 的劃線：第一版**限制在單一 block 內**。使用者拖曳跨段時，切成多筆 highlight，或直接截斷到第一個 block 結束。這個限制大幅簡化錨定，且實務上絕大多數劃線不跨段。

---

## 4. 核心流程

### 4.1 儲存文章 (Save)
入口三種，都走同一條 pipeline：
1. **iOS 分享捷徑**：在 Safari 分享 → 「存到拾句」→ 開啟 `?save=<encoded-url>`。
2. **App 內貼上網址**：收件匣頁面的輸入框。
3. **手動輸入**：無網址，直接打字或貼文字（給實體書、Facebook 貼文等抓不到的來源）。

Pipeline：
```
URL
 → 建立 document（state=inbox, extractStatus=pending，立刻寫入 Firestore，UI 馬上看得到）
 → 呼叫擷取服務（§5）
 → 成功：寫入 content/body（markdown + blocks），extractStatus=ok
 → 失敗：extractStatus=failed，UI 顯示「無法擷取，改為手動貼上內文？」
```
**先寫 document 再擷取**很重要：使用者按下存檔後應立刻看到文章出現在收件匣（顯示「擷取中…」），而不是等 3 秒轉圈。

### 4.2 閱讀 (Reading View)
- 排版是這個 app 的門面。襯線字（Noto Serif TC）、行高 1.9–2.0、單欄限寬約 34–38 中文字、留白充足。
- 頂部極簡：返回、標題、進度條。捲動時隱藏。
- 底部工具列：字級調整、封存、加入稍後、劃線列表。
- 進入文章時捲動到 `lastPosition` 的 block，並提示「從上次讀到的地方繼續」。
- 捲動時 throttle 更新 `readingProgress` 與 `lastPosition`（每 2–3 秒或每離開一個 block 寫一次，避免 Firestore 寫入過頻）。
- 讀到底部 → 提示「封存這篇？」

### 4.3 劃線與錨定 (Highlighting) ★ 最核心、最容易做壞

**渲染規則（決定錨定是否穩定）**
- 每個 block 渲染成一個 DOM 元素，帶 `data-block-index` 屬性。
- **block 內的文字節點結構必須是純文字的確定性映射**：也就是說，`blocks[i].text` 的第 N 個字元，必須能唯一對應到畫面上的第 N 個字元。行內粗體/連結會切出多個 text node，因此不能天真地用 `Range.startOffset`。

**建立劃線的流程**
1. 使用者選字，取得 `window.getSelection()` 的 Range。
2. 從 `range.startContainer` 往上找到最近的 `[data-block-index]` 元素，得到 `blockIndex`。
3. 計算「該 block 元素內、從開頭到 range 起點之間的純文字長度」→ 這就是 `startOffset`。實作方式：用 `document.createRange()` 建一個從 block 元素起點到 selection 起點的 range，取 `.toString().length`。同理得 `endOffset`。
4. 驗證：`blocks[blockIndex].text.slice(startOffset, endOffset) === selection.toString()`。**不相等就中止並回報 bug**，不要靜默存錯的錨點。
5. 存入 Firestore。

**渲染既有劃線**
1. 載入該文章的所有 highlights，依 `blockIndex` 分組。
2. 對每個 block，把該 block 的所有劃線區間依 `startOffset` 排序，切出「未劃線 / 已劃線」交錯的片段，重建該 block 的 innerHTML，劃線片段包在 `<mark data-hl-id="...">`。
3. 區間重疊時：合併成一個顯示區間（但底層仍是兩筆 highlight），或後者覆蓋前者。第一版取「合併顯示」。

**為什麼這樣做得通**：因為 `blocks` 在存檔時就固定了，內容永不變動。這比 Readwise 面對「網頁隨時改版」的情境簡單得多——我們存的是快照。

**劃線互動**
- 選字後跳出小浮層：劃線 / 劃線並加註 / 複製 / 取消。
- 點既有的 `<mark>`：跳出 編輯註解 / 加標籤 / 改顏色 / 刪除。
- 文章底部或側邊有「本篇劃線」列表，點擊跳到位置。

### 4.4 回顧 (Daily Review)
劃線一旦建立就自動進入回顧池，走 §7 的間隔重複演算法。回顧卡片顯示：
- 劃線文字（襯線大字）
- 來源：`title` · `author` · `siteName`
- 「回到原文」→ 開啟文章並捲動到該劃線位置（highlight 閃爍一下）
- 動作：已讀下一張 / 收藏 / 移出輪替 / 編輯註解 / 調整此來源頻率

### 4.5 收件匣管理
- 三個 tab：**收件匣 (Inbox)** / **稍後 (Later)** / **封存 (Archive)**。
- 每篇顯示：封面縮圖、標題、來源站名、預估閱讀時間（中文以 ~400 字/分估算）、劃線數、進度條。
- 滑動手勢：右滑封存、左滑移到稍後。

---

## 5. 內容擷取架構（唯一的後端）

純前端 `fetch()` 抓不到別人的網頁（CORS）。必須經過一個擷取端點。

### 5.1 方案 A：Jina Reader（優先，零維護）
在網址前面加 `https://r.jina.ai/` 即可把任意 URL 轉成 markdown，無需 API key（有速率限制，帶免費 key 可提高）。加上 `Accept: application/json` 標頭會回傳含 title、content 的 JSON。

```js
const r = await fetch('https://r.jina.ai/' + encodeURIComponent(url), {
  headers: { 'Accept': 'application/json' }
});
const { data } = await r.json();   // data.title, data.content (markdown)
```

**⚠️ 實作前必須先驗證的一件事**：Jina Reader 是否回傳允許瀏覽器直連的 CORS 標頭。用瀏覽器 console 直接跑一次上面的程式碼測試。
- 若可以 → 純前端完成，不需要任何後端。
- 若被 CORS 擋 → 走方案 B。

### 5.2 方案 B：自建 Cloud Run proxy（備援，你已經有這個 pattern）
Design Vibe Sensing 已經蓋過一模一樣的東西：Cloud Run + Playwright，Jina 作為 fallback，API key gating。直接復刻：

```
POST https://<your-cloud-run>/extract
Body: { url }
Header: X-Api-Key: <shared secret>
→ { title, author, siteName, coverUrl, markdown }
```
- 這個 service 負責回傳 `Access-Control-Allow-Origin: https://zi-jhin-chen.github.io`。
- 內部策略：先試 Jina，失敗再用 Playwright + Readability.js 抓，再失敗回傳 `failed`。
- Cloud Run 免費額度對個人用量綽綽有餘（每月幾百次擷取）。

### 5.3 擷取後處理
1. markdown → 解析成 `blocks[]`（用同一份 parser，前後端行為必須一致；建議統一在**前端**解析，後端只回 markdown）。
2. 過濾雜訊 block：導覽列殘留、「訂閱電子報」、版權宣告、分享按鈕文字。
3. 估算 `wordCount`（中文計字元，英文計 whitespace 分詞）。
4. 抓 `coverUrl`：markdown 中第一張圖，或後端回傳的 og:image。

### 5.4 特殊來源
- **Twitter / X**：先試 proxy 抓；抓不到就退回手動貼上。**不要為它寫專門的 oEmbed 整合**，該端點近年不穩定且非 CORS 友善，投報率低。
- **Facebook**：不支援。UI 上偵測到 `facebook.com` 網址時，直接提示「Facebook 貼文無法自動擷取，請手動貼上內文」，並把網址填進 `sourceUrl`。這比讓使用者按下存檔然後失敗要好。
- **付費牆文章**：擷取會拿到前幾段。維持現狀，不做繞過。

---

## 6. 技術要點

### 6.1 離線
- Firestore `persistentLocalCache` + `persistentMultipleTabManager`（已有）→ 讀過的文章離線可讀，離線劃線連網後自動同步。
- Service Worker precache app shell（HTML/CSS/JS/字型）→ 離線可開 app。
- **未讀文章的離線可用性**：Firestore 快取只涵蓋「曾經讀取過」的文件。若要「存了就能離線讀」，需在存檔完成後主動 `getDoc(content/body)` 一次把內文拉進快取。實作時在擷取成功後補一次讀取即可。

### 6.2 Firestore 寫入節流
閱讀進度會頻繁變動。規則：
- `lastPosition` / `readingProgress`：throttle 至少 3 秒一次，且離開文章時強制寫一次。
- 不要在 scroll handler 裡直接 `updateDoc`。

### 6.3 搜尋
Firestore 沒有全文索引。分兩層：
- **劃線搜尋**（主要需求）：劃線數量級小（數千筆），一次載入 `text` + `note` + `tags` 到前端做過濾即可。
- **文章內文搜尋**：不做。或退而求其次只搜 `title` / `author` / `siteName`。
- 若日後真的要全文搜尋，唯一乾淨的路是接 Algolia / Typesense，超出本專案範圍。

### 6.4 Firestore 結構總覽
```
users/{uid}/documents/{docId}                 // 文章 metadata
users/{uid}/documents/{docId}/content/body    // 內文 markdown + blocks
users/{uid}/highlights/{hlId}                 // 劃線（含錨定 + SRS 狀態）
users/{uid}/reviewSessions/{YYYY-MM-DD}       // 當日回顧進度
users/{uid}/meta/state                        // streak、設定
```
安全規則沿用：每人只能讀寫 `users/{uid}` 底下。

### 6.5 效能
- 收件匣列表分頁載入（`limit(20)` + 捲動載入更多）。
- 每日選卡不要把全部 highlights 拉進前端。在 highlight 上維護 `srs.nextReviewAt`，用 `where('srs.nextReviewAt', '<=', now).orderBy(...).limit(N*3)` 取候選池，再於前端加權抽樣。

---

## 7. 間隔重複演算法（沿用，簡述）

Readwise 的做法是**以回想機率半衰期為基礎的衰減演算法**，而非日期式排程。

```
p(t) = 2 ^ ( -elapsedDays / halfLife )
```
- 新劃線給小的初始 `halfLife`（0.5–1 天），讓它很快進入回顧。
- 每日選卡：以 `(1 - p) * document.frequency` 為權重，加權隨機抽 N 張（N 由設定決定，預設 15）。
- 標記已讀 → `halfLife *= 2.0`、`lastReviewed = now`、`reviewCount += 1`、重算 `nextReviewAt`。
- 使用者 tune down 某來源 → `document.frequency` 下調（例如 ×0.5），避免劃線多的長文洗版。
- 回顧進度存 `reviewSessions/{date}`，跨裝置續讀。
- 完成當日回顧 → streak +1。

---

## 8. 資訊架構

```
拾句 (PWA)
├── 登入 (Google)
├── 收件匣 Inbox      ← 預設著陸頁
│   ├── Inbox / Later / Archive 三個 tab
│   ├── 貼上網址存文章
│   └── 文章卡片（封面、標題、來源、閱讀時間、劃線數、進度）
├── 閱讀頁 Reader
│   ├── 乾淨排版內文
│   ├── 選字劃線 + 註解 + 標籤
│   ├── 本篇劃線列表
│   └── 續讀位置、進度
├── 每日回顧 Daily Review
│   ├── 卡片（劃線 + 來源 + 回到原文）
│   └── 完成畫面 + streak
├── 劃線庫 Highlights
│   ├── 依文章分組 / 依標籤篩選
│   └── 搜尋
├── 匯出 Export
│   └── Markdown / CSV / JSON
└── 設定 Settings
    ├── 每日回顧數量
    ├── 閱讀字級
    └── 擷取服務端點（若走方案 B）
```

---

## 9. 路線圖

**Phase 1 — Reader 主線（先讓「存→讀→劃」跑通）**
1. 資料模型重構：`documents` + `documents/{id}/content/body` + `highlights`。既有資料寫 migration。
2. 驗證擷取服務（§5.1 CORS 測試），必要時建 Cloud Run proxy。
3. 存文章 pipeline：貼網址 / iOS 捷徑 `?save=` / 手動輸入。
4. 閱讀頁：markdown → blocks → 渲染 + 排版。
5. **劃線錨定**（§4.3）：建立、驗證、渲染、編輯、刪除。
6. 收件匣三態 (inbox/later/archive) + 閱讀進度續讀。

**Phase 2 — 把回顧接回來**
7. 劃線自動進入 SRS 回顧池，實作 §7 演算法取代現有簡易選卡。
8. 回顧卡片「回到原文」跳轉並定位。
9. reviewSessions 跨裝置續讀、streak。
10. 頻率調整 (tune up / tune down)。

**Phase 3 — 加厚**
11. Service Worker 離線殼 + 存檔後預抓內文。
12. 標籤系統、劃線庫篩選、匯出（Markdown 依文章分組 / Obsidian 友善格式）。
13. 劃線顏色、跨 block 劃線。
14. RSS 訂閱、YouTube 逐字稿、PDF（各自獨立評估）。

---

## 10. 驗收標準

**Phase 1**
- [ ] 在 iPhone Safari 分享一篇 Medium 文章 → 拾句開啟 → 文章出現在收件匣，數秒內狀態變成可讀。
- [ ] 開啟文章，排版乾淨（無導覽列、無廣告、無「訂閱電子報」殘留）。
- [ ] 選取一段文字劃線 → 重新整理頁面 → 劃線仍在原位、範圍完全正確。
- [ ] 在段落中間有粗體字、超連結的地方劃線 → 錨定仍然正確（這是最容易壞的 case，必須測）。
- [ ] iPhone 讀到一半 → iPad 開啟同一篇 → 從同一個位置繼續。
- [ ] 飛航模式下開啟讀過的文章 → 可讀、可劃線；恢復連線後劃線同步。
- [ ] 貼上一個 Facebook 網址 → UI 明確提示無法擷取，並提供手動貼上路徑（不是靜默失敗）。

**Phase 2**
- [ ] 新劃線隔天出現在每日回顧。
- [ ] 回顧卡片點「回到原文」→ 開啟文章、捲到該劃線、highlight 閃爍。
- [ ] 對某篇長文 tune down → 該文劃線在往後回顧的出現頻率明顯下降。

---

## 附錄 A — blocks parser 契約

markdown → blocks 的解析必須是**確定性**的：同一份 markdown 每次解析出完全相同的 `blocks` 陣列。這是錨定穩定的前提。

- 使用固定版本的 markdown parser（建議 `marked` 或 `markdown-it`，鎖版本號）。
- `type` 取值：`h1`–`h3`、`p`、`quote`、`li`、`code`、`img`。
- `text` 為該 block 去除所有行內標記後的純文字。
- 圖片 block 的 `text` 為 alt 文字（不可劃線，渲染時不加 `data-block-index`）。
- 空 block 直接捨棄，捨棄後**重新編號 `i`**，並以重新編號後的結果存檔。

## 附錄 B — 錨定正確性測試案例

實作 §4.3 後，至少用以下情境驗證：
1. 純文字段落中間劃一句話。
2. 段落開頭第一個字開始劃。
3. 段落最後一個字結束。
4. 劃線範圍**跨越一個粗體字**（`這是**重點**內容` → 劃「是重點內」）。
5. 劃線範圍**跨越一個超連結**。
6. 同一個 block 內有兩筆不重疊的劃線。
7. 同一個 block 內有兩筆**重疊**的劃線。
8. 含 emoji 或中日韓字元的段落（注意 JS 字串長度以 UTF-16 code unit 計，emoji 佔 2；`slice` 與 `Range.toString().length` 的行為一致，故可通用，但仍需實測）。
