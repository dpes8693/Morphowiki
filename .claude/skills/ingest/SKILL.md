---
name: ingest
description: 分析英文單字並建立雙向關聯的字典頁。觸發時機:使用者輸入「/ingest <word>」、要求 ingest 某個英文單字、或請求把某個英文單字加入字典 / 字根字首字尾庫。對單字進行語素拆解、產生 zh-TW 釋義並維護 wiki-link 雙向反向連結。
---

# Ingest Skill

當使用者請求 `/ingest <word>`(或要求分析、收錄某個英文單字)時,依照以下步驟執行。**所有檔案路徑都使用相對於專案根的 `dictionary/` 子樹**。

## 適用情境
- 使用者輸入 `/ingest <word>` 或 `/ingest <word1> <word2> …`
- 使用者說「幫我 ingest <word>」、「把 <word> 加入字典」、「分析這個單字」
- 使用者貼出一個英文單字並要求收錄

## 不適用情境
- 使用者要查既有的單字 → 直接讀 `dictionary/words/<word>.md`
- 使用者要把字加入 stopwords → 改用 `stopword` skill
- 使用者要 ingest 中文詞或片語 → 不處理,詢問使用者

---

## Step 1 — 正規化輸入

1. 把 `<word>` 轉成小寫、去除前後空白。
2. 若包含空白(片語)或非英文字母字元,告知使用者目前只支援單一英文單字並停止。

## Step 2 — 檢查 stopwords(**必做,不可略過**)

1. 讀 `dictionary/stopwords.md`。
2. 解析 `## default` 與 `## custom` 兩個 section 內的所有單字(每行一個,可能是 `- word` 列表或裸字)。
3. **若 `<word>` 出現在任一 section**:
   - **拒絕 ingest**,輸出:`「<word>」已在 stopwords (<section>),不會建立字典頁。如要強制 ingest,請先用 /stopword 將其移除(目前未支援)。`
   - 停止後續步驟。
4. 若不在 stopwords,繼續。

## Step 3 — 檢查是否已存在

1. 若 `dictionary/words/<word>.md` 已存在,告知使用者該字已收錄,並詢問是否要覆蓋。若使用者未要求覆蓋就停止。

## Step 4 — 語素拆解

從詞源學角度把單字拆成 **prefix + root + suffix**(任何一段可缺)。例:
- `unbelievable` → `un` (prefix) + `believe` (root) + `able` (suffix)
- `rebuild` → `re` (prefix) + `build` (root)
- `happy` → 無 prefix + `happy` (root,本身為自由詞素) + 無 suffix

對每個語素準備:
- morpheme 拼寫(全小寫,prefix 不帶連字號,例如 `un` 而非 `un-`)
- type(prefix / root / suffix)
- 簡短 zh-TW 意義說明

## Step 5 — 寫 `dictionary/words/<word>.md`

格式(嚴格依此模板):

```markdown
---
word: <word>
pos: <part of speech>
ipa: <IPA>
added: <YYYY-MM-DD>
---

# <word>

**詞性**: <part of speech>
**IPA**: <IPA>
**中文釋義**: <zh-TW gloss,可多義以分號分隔>

## 拆解
- 字首: [[../prefix/<prefix>]] — <zh-TW meaning>
- 字根: [[../root/<root>]] — <zh-TW meaning>
- 字尾: [[../suffix/<suffix>]] — <zh-TW meaning>

## 詞源
<2–4 句 etymology,可提及來源語言、原意演變>

## 記憶法
<1–3 句中文記憶法,結合語素或意象>
```

規則:
- 若某語素不存在,**整行省略**(不要寫「無」)。
- IPA 用標準 IPA(Cambridge/Merriam-Webster 風格皆可),用 `/…/` 包起來。
- 中文用繁體中文(zh-TW)。
- `added` 用當天日期,格式 `YYYY-MM-DD`。
- wiki-link 一律使用相對路徑 `[[../prefix/<x>]]`、`[[../root/<x>]]`、`[[../suffix/<x>]]`,不加 `.md`。

## Step 6 — 建立 / 更新語素頁(**雙向 link 的反向端**)

對 Step 4 拆出的每個語素 `<m>`,操作對應檔案 `dictionary/<type>/<m>.md`:

### 若檔案不存在 — 建立

```markdown
---
morpheme: <m>
type: <prefix|root|suffix>
meaning: <zh-TW meaning>
---

# <m>-   (prefix)   或   # <m>   (root)   或   # -<m>   (suffix)

**類型**: <prefix|root|suffix>
**意義**: <zh-TW meaning>

## Words containing this
- [[../words/<word>]]
```

標題符號規則:
- prefix → `# <m>-`(後綴連字號)
- suffix → `# -<m>`(前綴連字號)
- root → `# <m>`(無連字號)

### 若檔案已存在 — 更新

1. 讀檔。
2. 找 `## Words containing this` section。若不存在,在檔尾追加。
3. 加入新的一行 `- [[../words/<word>]]`,**維持字母排序、去重**。
4. 不要動 frontmatter 或其他既有內容。

**雙向同步原則**:words MD 內的每個 morpheme link 必須在對應的 prefix/root/suffix MD 內有反向 link。**兩端必須同次操作中一起更新。**

## Step 7 — 更新 `dictionary/index.md`

1. 若檔案不存在,建立骨架:

   ```markdown
   # Dictionary Index

   字典所有頁面總覽。每行一個 page + 一句說明。

   ## Words

   ## Prefix

   ## Root

   ## Suffix
   ```

2. 在 `## Words` section 加入 `- [[words/<word>]] — <一句 zh-TW 說明>`,維持字母排序、去重。
3. 對 Step 6 中**新建**的語素頁,也分別在 `## Prefix` / `## Root` / `## Suffix` 加入 `- [[<type>/<m>]] — <一句 zh-TW 說明>`。**僅新建時加,既有頁面不重複加。**

## Step 8 — Append `dictionary/log.md`

1. 若檔案不存在,建立:

   ```markdown
   # Log

   Append-only 操作紀錄。
   ```

2. 追加一行:

   ```
   - <YYYY-MM-DD> ingest <word> (prefix=<m1>, root=<m2>, suffix=<m3>)
   ```

   缺少的語素省略對應欄位。

## Step 9 — 回報

告訴使用者:
- 建立了哪些檔案
- 更新了哪些既有檔案(尤其是反向 link)
- 拆解結果

---

## 重要約束

1. **先查 stopwords,任何字在 stopwords 內一律拒絕 ingest**。
2. **雙向 link 必須同時更新兩端** — words 內 link 到 morpheme,morpheme 必須 link 回 words。
3. 中文一律 zh-TW。
4. 不要建立 `.claude/commands/` 任何檔案 — 全部走 skill 路線。
5. 不要更動 `dictionary/stopwords.md`(那是 stopword skill 的職責)。
6. 不要刪除既有檔案的內容,只能追加或在指定 section 內插入。
