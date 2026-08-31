# CLAUDE.md — scamper-thinking-generator

「SCAMPER 創新思考法產生器」——單檔前端工具，把 SCAMPER 創新思考法（Substitute／Combine／Adapt／Modify／Put to Another Use／Eliminate／Reverse，共 7 個構面）做成互動網頁：輸入一個現有產品、服務或問題，依七個角度逐一發想 2-3 個創新點子，最後綜合評估選出最具商業價值或突破性的前三個方案並附落地建議。核心互動模型是 **manual-first**（比照 `mandala-thinking` 的做法，本專案即是依使用者指示「參考曼陀羅思考法產生器」建立）：所有欄位平常就是可直接編輯的 `<textarea>`，AI 只是加速發想的選用功能，不是必經流程。

## 架構

單一 `index.html`：CSS/JS 全內嵌、無外部資源、無建置步驟。視覺主題是珊瑚玫瑰色系「打破常規、大膽重組」風格（`--bg #180f13` + 玫瑰紅 `--accent #fb7185`），與姊妹專案（薄荷綠/藍/琥珀/翠綠/紫/靛藍/橘/天藍/洋紅/深石墨藍+teal/咖啡棕）區隔。

- **資料結構**（`state`，存 `localStorage` key `scamperState`）：`{topic, scamper:{substitute:[],combine:[],adapt:[],modify:[],other_use:[],eliminate:[],reverse:[]}, evaluation:{top3:[{facet,idea,reason,action}×3], summary}}`。`scamper[key]` 是字串陣列，每構面可存 2-3+ 筆點子，各自可新增/刪除。
- **渲染**：`renderSignatureStrip()` 畫 S-C-A-M-P-E-R 七字母徽章列（視覺簽名元素，點擊捲動聚焦對應卡片，兼作跳轉導覽，比照 `six-thinking-hats-generator` 的帽架手法但改用圓角方塊徽章）；`renderFacetGrid()`/`renderIdeaList(key)` 畫 7 張構面卡片；`renderEvaluation()` 畫綜合評估區塊。所有 `<textarea>` 的 `input` 事件直接寫回 `state` 對應欄位並 `persistState()`。
- **綜合評估「選前三」機制**（mandala 沒有的新功能）：`evaluation.top3` 用**複製文字（denormalized）**而非索引參照——按點子旁「⭐選入前三」把文字複製進空卡槽，之後刪除原始點子不影響已選入的評估內容。雙軌並行：手動選入，或按「🧠 AI 綜合評估」讓 AI 讀取全部已填點子回傳 JSON 覆蓋整個 `evaluation` 物件（因為排名本身是跨構面比較的結果，無法只覆蓋單構面）。
- **PRESETS**：5 組內建範例（`index.html` 內 `PRESETS` 陣列），第一組「巷口手沖咖啡吧」是手打（非 AI）完整示範（含七構面點子與完整綜合評估），其餘 4 組跨產業虛構主題（共享滑板車／線上讀書會／校園置物櫃／寵物旅宿，皆不含真實個資/機構）。
- **BYOK AI**：與 `mandala-thinking`/`ai-prompt-generator`/`Prompt` 同一套 `callLLM()`（逐字複製，未修改）——四家服務商（Claude/OpenAI/Gemini/OpenRouter）、逕自瀏覽器直連、逾時 180 秒、429/500/503/529 自動重試。設定存 `localStorage`（key `scamperApiConfig`）。
  - 「🧠 AI 一鍵展開全部七構面」：`buildFullPrompt()` 要求 AI 以 JSON 回覆七構面各 2-3 筆點子，`extractJson()` 用正則抓出回應中第一個 `{...}` 區塊再 `JSON.parse`。
  - 每張構面卡片旁「🧠 AI 重新生成此構面」：`regenerateFacet(key)` 只呼叫 `buildFacetPrompt()`（帶入其餘構面現有內容當上下文避免重複），只覆蓋該構面。
  - 「🧠 AI 綜合評估：選出前三方案」：`buildEvaluationPrompt()` 統整全部已填點子（帶構面標籤）要求 AI 選前 3 名＋理由＋落地建議＋總評。
- **匯出**：`buildFullText()` 把主題＋七構面點子（含中英文構面名）＋綜合評估前三＋理由＋建議＋總評組成純文字，供「複製為文字」「下載 .txt」使用；「🖨 列印 / 存 PDF」走 `window.print()` + `@media print` 樣式。
- **列印/PDF 浮水印**：`#printWatermark`，與 `mandala-thinking`/`six-thinking-hats-generator`/`new-product-strategy-studio` 等共用同一張「馬克老師」品牌 base64 PNG（未經對話視窗，用 Bash `sed` 直接從原始檔抽出該行字串複製過來）。

## 與姊妹專案的差異

- **不加序號授權機制**——比照 `mandala-thinking`/`coffee-ig-planner`/`social-post-grader` 的先例，這是「思考框架練習工具」類別而非「商業文件產生器」類別，使用者本次未要求鎖工具。
- **manual-first 設計**：延續 mandala 的核心互動——欄位本身就是最終產出，AI 只是灌入內容的一種方式，沒有「先組成提示詞→送給AI」的中間步驟。
- **綜合評估選前三**是本專案獨有、mandala 沒有的新機制（見上方架構說明）。
- **視覺簽名元素**改用「S-C-A-M-P-E-R 七字母徽章列」，而非 mandala 的九宮格本身或 six-hats 的帽架換色。

## 共用模組（直接複製既有程式碼）

- **跑馬燈**：`#marqueeBar` 逐字複製 `mandala-thinking/index.html` 的 IIFE，共用同一個 Google Apps Script 端點與 Sheet（`localStorage` key 改為 `scamperMarquee`）。
- **創作者資訊／使用警語**：`manual.html`／`index.html` footer 逐字複製既有共用內容，第一點警語措辭改成貼合 SCAMPER 輸出情境。
- **PWA**：`manifest.json`／`service-worker.js`／`#installBtn` 邏輯比照既有專案（network-first + 同源快取備援）。圖示（`icons/`）用 Python PIL 現畫，深色珊瑚玫瑰底＋7 個圓角方塊環狀排列意象（不沿用九宮格/帽子造型）。
- **訪客計數器**：`visitor-badge.laobi.icu`，`page_id=m255525.scamper-thinking-generator`。

## Port

**8806**（工作區 8765-8805 已全數被其他專案占用，8805 是最新的 `amazon-listing-generator`）。已在 `.claude/launch.json` 新增對應設定，供 Preview MCP 使用。

## 指令

無建置/測試指令。修改 `index.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8806` 測完關閉。修改內嵌 `<script>` 後可用以下方式快速檢查語法：

```bash
python -c "
import re
html = open('index.html', encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write('\n\n'.join(re.findall(r'<script>(.*?)</script>', html, re.S)))
"
node --check _check.js
```

## 部署

已推公開 GitHub repo `M255525/scamper-thinking-generator`，用 `.github/workflows/deploy-pages.yml`（比照 `mandala-thinking` 逐字複製）以 Actions workflow 部署 GitHub Pages（非 legacy branch-source），已上線：<https://m255525.github.io/scamper-thinking-generator/>。

## 本次未做（後續視需要再處理）

- 桌面版 exe 打包
- 序號授權（使用者本次未要求；若之後要鎖工具，比照 `ai-prompt-generator` 的「鎖整個工具 12 個月」模式加回）
