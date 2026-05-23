---
name: setup
description: 初始化 en-llm-wiki 專案結構。檢查 Node.js 是否安裝、巢狀建立 dictionary/{prefix,root,suffix,words} 與 review/ 與其下檔案、詢問使用者是否讓 LLM 生成約 150 個最簡單的英文功能字寫入 stopwords.md 的 ## default section。觸發時機:使用者輸入「/setup」、「初始化專案」、「第一次設定 en-llm-wiki」、或在空資料夾要求設定 en-llm-wiki。注意:不要與 Claude Code 內建的 /init(生成 CLAUDE.md)混淆。
---

# Setup Skill

當使用者請求 `/setup`(或「初始化專案」、「第一次設定」、「幫我把資料夾結構建好」)時,依以下步驟執行。本 skill 把 en-llm-wiki 的目錄結構與必要檔案一次到位。

---

## Step 1 — 環境檢查:Node.js

用 Bash tool 跑:

```
node --version
```

- 若回傳 `v18.x.x` 或更高 → 通過,進 Step 2
- 若回傳低於 v18(例如 v16) → 提示使用者升級,**但仍可繼續**(只有 `/review-word` 會用到 node)
- 若 `node` 找不到 / 指令錯誤 → 提示安裝建議,**並讓使用者選**「先繼續(之後再裝)」或「中止」:
  - Windows:`winget install OpenJS.NodeJS.LTS` 或下載 https://nodejs.org
  - macOS:`brew install node`
  - Linux:用各自包管理(`apt install nodejs npm` / `dnf install nodejs` …)
- 同時跑 `npm --version` 確認 npm 可用

**回報格式**:單行寫出偵測到的版本(或缺失)。

---

## Step 2 — 巢狀建立資料夾

用 Bash tool 建立(用 `mkdir -p` 等效,Windows PowerShell 用 `New-Item -ItemType Directory -Force`)。**已存在的不要動**:

```
dictionary/
dictionary/words/
dictionary/prefix/
dictionary/root/
dictionary/suffix/
review/
review/web/
.claude/skills/
```

對每個目錄回報「created / already exists」。

---

## Step 3 — 建立必要檔案(若不存在)

| 檔案 | 初始內容 |
|------|----------|
| `dictionary/stopwords.md` | `# StopWords\n\n## default\n\n## custom\n` |
| `dictionary/index.md` | `# Dictionary Index\n\n_自動維護的目錄。新單字、字首、字根、字尾會被 ingest skill 加進對應 section。_\n\n## Words\n\n## Prefix\n\n## Root\n\n## Suffix\n` |
| `dictionary/log.md` | `# Log\n\n_Append-only 操作紀錄,格式: - YYYY-MM-DD &lt;action&gt; &lt;target&gt;_\n` |
| `review/flashcards.json` | `{\n  "version": 1,\n  "cards": []\n}\n` |
| `review/package.json` | 若已存在則跳過。若無,寫入最小版:`{ "name": "en-llm-wiki-review", "version": "0.1.0", "private": true, "scripts": { "start": "node server.js" }, "dependencies": { "express": "^4.19.2" } }` |

**注意**:
- UTF-8 無 BOM、LF 換行
- 用 Read tool 先確認檔案存在性。若檔案已存在但內容不同,**不要覆寫** — 提示使用者並跳過
- `server.js`、`web/index.html` 等實作檔由 review-word skill 在啟動時負責;init 不要建這些

---

## Step 4 — 詢問:是否生成 default stopwords?

對使用者出示**單一問題**:

> 要不要讓我生成約 150 個最常見、最簡單的英文功能字(冠詞、代名詞、常用介系詞、be 動詞、助動詞等)寫入 `dictionary/stopwords.md` 的 `## default` section?之後 `/ingest` 遇到這些字會自動跳過,避免你複習到 the / is / of 這種太基礎的字。

選項:
1. **生成(推薦)** — 寫入 default section
2. **跳過,我之後自己加** — 不動 default section
3. **少一點(只要 ~50)** — 只寫核心功能字

**使用 AskUserQuestion tool 問,不要用純文字提問**。

---

## Step 5 — 若使用者同意 → 生成 stopwords

把以下英文功能字寫入 `dictionary/stopwords.md` 的 `## default` section,每行 `- <word>`,**全小寫、字母排序、去重**。

「全集 ~150」應包含(由 Claude 在執行時整理,以下為**指引清單,不必逐字照抄**):

- 冠詞:a, an, the
- 人稱代名詞主格:i, you, he, she, it, we, they
- 人稱代名詞受格:me, you, him, her, it, us, them
- 所有格代名詞:my, your, his, her, its, our, their
- 所有格代名詞獨立式:mine, yours, hers, ours, theirs
- 反身代名詞:myself, yourself, himself, herself, itself, ourselves, yourselves, themselves
- 指示詞:this, that, these, those
- 疑問詞:what, which, who, whom, whose, where, when, why, how
- 連接詞:and, but, or, nor, so, yet, if, because, although, while, when, since, until, unless
- be 動詞:be, am, is, are, was, were, been, being
- have 助動詞:have, has, had, having
- do 助動詞:do, does, did, doing, done
- 情態:can, could, will, would, shall, should, may, might, must, ought
- 否定:not, no, never
- 量詞 / 限定詞:some, any, all, every, each, both, few, many, much, most, more, less, several, other, another, none, one, two, three
- 常用介系詞:of, in, on, at, to, from, by, with, for, about, into, onto, upon, over, under, between, among, through, across, against, before, after, during, without, within, around, near
- 副詞:very, too, also, just, only, even, still, already, yet, again, here, there, now, then, well, just
- there / it 句型相關:there, it
- 其他高頻:as, than, like, such, so, up, down, out, off, away, back

把上述去重、字母排序後寫入 `## default` section。**不要超過 160 個**,寧可精簡也不要塞滿。

寫完後在 `dictionary/log.md` append:`- 2026-05-12 setup seeded default stopwords (N words)`(N 為實際寫入字數)。

---

## Step 6 — 預先安裝 review 相依套件

**目的**:把首次 `/review-word` 的等待時間挪到 setup 階段。Step 1 偵測 Node.js 失敗時跳過本步驟。

1. 檢查 `review/node_modules/` 是否已存在。
   - 已存在 → 跳過(回報「review deps already installed」)
2. 用 Bash tool 從**專案根目錄**跑(**不要 cd**,避免 cwd 黏滯造成下次命令失敗):
   ```
   npm install --prefix review
   ```
   - `--prefix` 讓 npm 在指定目錄安裝,不依賴 cwd,可重複呼叫不會踩到「`cd review` 已在 review/」之類錯誤
   - **不要** 用 `run_in_background: true`(要等它完成)
3. 失敗 → 把 npm 錯誤完整貼給使用者,但**不中止 setup**(已建好的目錄/檔案仍然有效,使用者可手動補裝);收尾報告標明 deps 未裝
4. 成功 → 在 `dictionary/log.md` append `- <YYYY-MM-DD> setup npm install (review)`

---

## Step 7 — 收尾報告

對使用者回報:

1. ✅ 已建目錄 / 跳過已存在的目錄(列出)
2. ✅ 已建檔案 / 跳過已存在的檔案(列出)
3. Node.js 偵測結果
4. default stopwords 是否寫入,字數
5. review 相依套件:已安裝 / 已存在 / 失敗(若失敗附原因)
6. **下一步建議**:
   - `/ingest <英文單字>` 收第一個字
   - `/flashcard <word>` 加入複習
   - `/review-word` 打開複習網頁(deps 已預裝,直接啟動)

---

## 邊界 / 禁令

- 不要動 `llm-wiki-prompt.md`、`README.md`、`manual.md`、`CLAUDE.md`(它們不是 init 該管的)
- 不要建立 `.claude/commands/`
- 若使用者明確說「重做」或「reset」 → 仍然不要刪檔,只追加 / 修補;真要清空請使用者自己刪
- 若任何步驟失敗,**停下來回報**,不要靜默跳過

---

## cwd 黏滯陷阱(必讀)

Bash tool 的工作目錄會在多個 tool call 之間延續(shell state 不會,但 cwd 會),所以**不要**用 `cd <dir> && <cmd>` 來執行需要在 `<dir>` 下跑的指令 —— 第一次成功但第二次就會踩到「`cd: <dir>: No such file or directory」(因為已經身在 <dir> 裡)。

正確做法:
- npm:`npm install --prefix review`(從專案根)
- node 啟動 server:`node review/server.js`(server.js 內用 `__dirname` 解析路徑,不依賴 cwd)
- 其他 npm 子指令:`npm --prefix review run <script>`

任何新增的相依檢查 / 安裝指令,都要遵守這條規則。
