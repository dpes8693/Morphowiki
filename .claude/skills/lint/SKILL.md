---
name: lint
description: 檢查 morphowiki 字典結構與單字品質。/lint 跑全庫純結構檢查(雙向 wiki-link、section/frontmatter 格式、拆解格式、stopword 衝突、flashcards 一致性、morpheme 引用完整性、## 筆記區 存在性);/lint <word> 在結構檢查之外,額外對該單字做 LLM 語意檢查(拆解詞源合理性、釋義準確性、詞源段與拆解 consistency、詞性、IPA),有問題列建議並詢問 user 套用哪幾條。觸發時機:使用者輸入「/lint」或「/lint <word>」、要求檢查字典結構、驗證最近 ingest/new-word 結果。
---

# Lint Skill

兩種模式:

- `/lint` — **全庫結構檢查**。純解析,無 LLM,只報不修,結果 chat 輸出。
- `/lint <word>` — **結構檢查 + 單字語意檢查**。語意部分用 LLM(對該單字一次,token 成本相當於小型 ingest);語意問題列出建議後**問 user 套用哪幾條**,user 同意才用 Edit tool 改檔。結構問題仍只報不修。

兩種模式都**不寫任何 lint report 檔、不更新 log.md、不動 stopwords / flashcards**。

## 適用情境
- 使用者輸入 `/lint`(全庫掃)或 `/lint <word>`(單字檢查 + 語意 + 互動修正)
- 使用者懷疑某次 ingest / new-word 結果有問題,想快速 sanity check
- 使用者修了某個 morpheme 頁,想確認雙向 link 沒斷

## 不適用情境
- 想跑 build / test → 本專案無此概念
- 想用 lint 重新 ingest 一個字 → 不會;lint 只審視既有檔,不會做拆解 / 雙向 link 新增等 ingest 行為(它連 morpheme 頁都不會主動修,雖然修補單一字的拆解可能會連帶要改 morpheme 頁,那也是套用語意建議時的副作用,而非 lint 主動做的事)
- 想用 lint `--fix` 一鍵修所有問題 → 不支援,**結構問題故意只報不修**(雙向 link 等問題需 user 決定哪邊是 source of truth,自動套建議反而危險)

---

## Step 0 — 解析參數

- 無參數 → 模式 = `full`(全庫結構檢查)。
- 一個英文單字參數 → 模式 = `single`(對該字結構+語意檢查 + 互動修正)。
  - 先檢查 `dictionary/words/<word>.md` 是否存在,不存在 → 告知「`<word>` 不在 dictionary/words/,可能 typo?或要先 `/ingest <word>`?」並停止。
- 多參數或非英文字 → 告知格式錯誤並停止。

## Step 1 — 蒐集檔案清單

`full` 模式讀全庫;`single` 模式只讀與目標 word 直接相關的檔案,但 stopwords / index / flashcards 還是要讀(以便對該字做 stopword 衝突、index 缺漏、flashcards 一致性檢查)。

讀以下路徑,建立內部資料結構:

- `dictionary/words/*.md` → words set(`single` 模式只需讀目標 word)
- `dictionary/prefix/*.md` / `dictionary/root/*.md` / `dictionary/suffix/*.md` → morpheme files(記下 type 與檔名 stem;`single` 模式只需要目標 word 拆解引用到的那些)
- `dictionary/stopwords.md` → 解析 `## default` 與 `## custom` 兩 section,組 stopword set
- `dictionary/index.md` → 解析 `## Words` 內的 `[[words/<w>]]`
- `dictionary/log.md` → 純行格式檢查用(`single` 模式可略過)
- `review/flashcards.json`(若存在)→ 解析 `cards[].word` 集合

## Step 2 — 對每個 `dictionary/words/<w>.md` 跑檢查

對每個 word file,讀檔並執行下列檢查。**用累積方式收集問題**,不要遇到一個錯就停。

### error(系統性)
1. **frontmatter 必要欄缺漏**:`word` / `pos` / `ipa` / `added` 任一缺失或值為空。
2. **`added` 非 ISO 日期**:不符 `^\d{4}-\d{2}-\d{2}$`。
3. **缺 section**:`## 拆解` / `## 詞源` / `## 記憶法` / `## 筆記區` 任一不存在(用 `splitSections` 風格的 `^##\s+(.+?)\s*$` 解析,與 `review/server.js:129` 對齊)。`## 筆記區` 內容空是合法的(user 還沒填),只要 section 標題行存在即通過;內容自由不檢查(繁簡、wiki-link 等都不管)。
4. **拆解行格式錯**:`## 拆解` 內以 `-` 開頭的行,**至少有一行**必須能匹配 `server.js:159` 的 `parseBreakdownLine`(label ∈ {`字首`, `字根`, `字尾`}、有 `[[…]]` link、可選 inline code、可選 em/en/hyphen dash + meaning)。若整個 section 一個合法行都沒有 → error。
5. **拆解缺 root**:解析後 root 為 null → error(ingest 規則要求至少 1 個 root)。
6. **拆解引用的 morpheme 檔不存在**:每條拆解行的 `[[../<type>/<key>]]`,對應 `dictionary/<type>/<key>.md` 必須存在。**這條專門抓 new-word 嚴格策略違規**(例如池外的 `-ation`)。
7. **反向 link 缺失**:對拆解引用的每個 morpheme 頁,該頁 `## Words containing this` section 必須包含 `[[../words/<w>]]`。缺則 error。
8. **flashcards 指向不存在的 word**:`review/flashcards.json` 內 `cards[].word` 在 `dictionary/words/` 找不到 → error。

### warn(可疑但不致命)
9. **stopword 衝突**:`<w>` 同時出現在 `stopwords.md`(default 或 custom)→ warn(可能先 ingest 後補 stopword,不算 bug)。
10. **詞性縮寫不合法**:frontmatter `pos` 或正文 `**詞性**:` 的值(分號分隔後)有任一不在白名單 `{n., v., vt., vi., adj., adv., prep., conj., pron., det., art., interj., aux., num.}` → warn。
11. **檔名 / frontmatter word 不一致**:`<w>` ≠ frontmatter `word:` 值 → warn。
12. **檔名非全小寫**:`<w>` 含大寫字母 → warn。
13. **index.md 未列**:`dictionary/index.md` 的 `## Words` section 找不到 `[[words/<w>]]` → warn。

## Step 3 — 對每個 morpheme 頁跑檢查

對 `dictionary/{prefix,root,suffix}/*.md` 的每個檔:

### error
14. **morpheme 頁列了不存在的 word**:`## Words containing this` 內的 `[[../words/<w>]]` 對應 `dictionary/words/<w>.md` 不存在 → error。
15. **morpheme 頁列了 word,但該 word 拆解不引用本 morpheme**:反向 link 不對稱 → error。

### warn
16. **frontmatter `morpheme:` 與檔名 stem 不一致**:解新規則檔名(`-al.md` / `inter-.md` / `nat.md`)的 stem(去前後連字號)應等於 frontmatter `morpheme:` → 不等則 warn。**舊檔(無連字號)也用同邏輯比對**。
17. **type 與所在目錄不一致**:frontmatter `type:` 應為 `prefix` / `root` / `suffix`,且與檔案所在目錄一致 → 不一致則 warn。

## Step 4 — index.md 反向檢查

18. **(warn)** index.md `## Words` 列了 `[[words/<w>]]` 但 `dictionary/words/<w>.md` 不存在。

## Step 5 — log.md 格式檢查

19. **(warn)** `dictionary/log.md` 內非空、非標題、非 `_…_` 說明的行,若不符 `^- \d{4}-\d{2}-\d{2} \S+ \S+` → warn。

## Step 5.5 — `single` 模式:LLM 語意檢查

僅 `/lint <word>` 走此步驟。讀目標 word md 後,以 LLM 針對該字進行四項語意檢查。**這是本 skill 唯一呼叫 LLM 的地方**,token 成本相當於一次小型 ingest。

對每項檢查,若發現問題,記錄成「問題 + 建議修正內容」格式(供 Step 7 修正互動使用)。

### 檢查 A:拆解詞源合理性 + 中文釋義準確性(core,有問題即 error)
- 拆解列出的每個 morpheme 是否真的是這個字歷史上的詞源拆解?例如 `dictation` 真實拆解是 `dict + -ation`,若 word md 寫成 `dict + -ate + -ion + …` 並不符合該字的詞源歷程(雖然 `-ation` 結構上 = `-ate + -ion`),仍要指出。
- 中文釋義是否準確、涵蓋常見義?有沒有漏掉重要 sense 或誤譯?

### 檢查 B:詞源段與拆解 consistency + 詞性合理性(有問題即 warn,因有解釋空間)
- `## 詞源` 段提到的來源語言、原意演變、衍生路徑,是否與 `## 拆解` 列出的 morpheme 一致?例如拆解寫 `nat`「出生」但詞源段主述其他來源 → 不一致。
- frontmatter `pos:` 與該字實際用法是否合理?多義字 pos 是否漏掉常見詞性(例如 `national` 只寫 `adj.` 但實際也常作 `n.`)?

### 檢查 C:IPA 合理性(warn,有不確定性)
- LLM 對 IPA 不全可信,但能抓明顯錯(例如音節數明顯不對、重音記號完全缺、用了非 IPA 字元)。
- 標記 warn,不標 error,避免 false positive 強迫使用者修。

### 不做的檢查
- 記憶法品質(主觀,不檢)
- 繁簡中文混用(false positive 風險高,不檢)
- `## 筆記區` 內容(user 領域,不檢)

### 重要:語意檢查使用既有檔案資訊
語意檢查**只讀 word md 的內容**,不主動重新查字典資料庫(本 skill 沒有外部資源)。LLM 用自身語言知識判斷即可。若 LLM 對某項拿不準,**寧可不報也不亂建議** — false positive 比漏報更糟,因為會誤導 user。

## Step 6 — 輸出

按「檔案」分組輸出結構問題。每個檔下面列出該檔的所有問題,前面標 `error` / `warn`。檔案分組之間空一行。若該檔無問題,**不要列**(避免噪音)。最後給一行摘要。

`single` 模式額外列出**編號的語意問題清單**(供 Step 7 互動修正參照),放在結構問題之後、摘要之前。每條一個編號,前面標 `error` / `warn`,後面跟一句建議修正。

### `full` 模式範例輸出

```
✗ dictionary/words/dictation.md
  error  拆解引用 [[../root/dict]] 但 dictionary/root/dict.md 不存在
  error  拆解引用 [[../suffix/-ation]] 但 dictionary/suffix/-ation.md 不存在

✗ dictionary/words/national.md
  warn   index.md 的 ## Words 未列 [[words/national]]

摘要: 2 errors, 1 warn, 6 word files / 4 morpheme files / 1 flashcards.json checked
```

若全部通過:

```
✓ 全部通過(6 word files / 4 morpheme files / 1 flashcards.json)
```

### `single` 模式範例輸出(`/lint dictation`)

```
=== 結構檢查 ===
✗ dictionary/words/dictation.md
  error  拆解引用 [[../suffix/-ation]] 但 dictionary/suffix/-ation.md 不存在
  warn   詞性 `n.` 合法

=== 語意檢查 ===
[1] error  拆解詞源不合理:`-ation` 雖可拆 `-ate + -ion`,但 dictation 歷史上的字尾就是 `-ation`(< 拉丁 dictationem)。建議拆為 root `dict` + suffix `-ation`,且若 pool 沒這兩個 morpheme,應該移除這個字而非硬留。
[2] warn   IPA `/dɪkˈteɪʃ.ən/` 重音記號位置可能偏左,建議改為 `/dɪkˈteɪ.ʃən/`(主音節在 teɪ)
[3] warn   ## 詞源 段未提到拉丁 dictare 來源,與拆解 root `dict` 預期一致性偏弱

要套用哪幾條? 回 "1,3" / "all" / "none" / "details <n>":
```

`details <n>` 是讓 user 看某條建議的完整 diff(顯示原文 → 建議改寫的具體文字),再決定要不要套用。

## Step 7 — 修正互動(僅 `single` 模式 + 有語意問題)

`full` 模式跳過本步。`single` 模式若 Step 5.5 列出至少一個語意問題:

1. **等待 user 回覆**。可能的回覆:
   - `none` 或空 → 不修任何條,前往 Step 8 收尾。
   - `all` → 套用所有語意條(error + warn 都套)。
   - `1,3` 或 `1 3` 等數字清單 → 只套這幾條。
   - `details <n>` → 顯示第 n 條的詳細 diff(原文段落 + 建議改寫後的段落,如 `## 詞源` 整段重寫),再回到等待。
   - 其他 → 視為不明確,重問一次。
2. **套用選中條目**:對每條,用 `Edit` tool 在目標 word md 內做精確替換。原則:
   - **改最小範圍**:只改該條目指涉的那行 / 那個 section,不順手重寫整檔。
   - **不改 frontmatter `added` 日期**(那是 ingest 日,改了會誤導歷史)。
   - **不動 `## 筆記區`**(user 領域)。
   - 若一條建議涉及拆解 morpheme 變動(例如 `dictation` 案例移除非法 `-ation` 引用),且該變動會牽連到對應 morpheme 頁的 `## Words containing this`,**附帶更新對應 morpheme 頁**(維持雙向 link 對稱);若該 morpheme 頁不存在,只改 word md 不創檔(尊重原本嚴格策略,讓使用者決定要不要 ingest 該 morpheme)。
3. **回報實際套用結果**:列出哪幾條被套了、改了哪些檔。
4. **結構問題仍不主動修**。若 user 對結構問題提出修法,告知「結構問題請手動修或重跑 `/ingest <word>`,本 skill 故意不做結構自動修」。

## Step 8 — 不要做的事

- **結構問題不主動修**。即使可推斷出「在 morpheme 頁補一行 `[[../words/<w>]]`」就能解決問題,也只報不修(語意修正除外,且只在 `single` 模式 + user 同意後做)。
- **`full` 模式不呼叫 LLM**。`full` 是 cheap structural check,只有 `single` 才會做 LLM 語意檢查。
- **不要寫 lint report 到任何檔案**。chat 輸出就好(維護報告檔有額外負擔,使用者要存自己會講)。
- **不要更新 log.md**(lint 是讀取操作,即便 `single` 模式做了語意修正也不留 log 痕 — 語意修正屬於 user 編輯範疇,不算 ingest 事件)。
- **不要動 stopwords.md / flashcards.json**。
- **不要動 `## 筆記區`**(user 領域,連 lint 都不准碰)。
- **不要創新 morpheme 頁**。若修拆解時牽連到「該 morpheme 不在池中」的情況,只改 word md,不建立 morpheme 檔(那是 ingest 的職責)。

---

## 重要約束

1. **只報不修(結構)、報後問再修(語意)**:結構問題只列出讓 user 處理;語意問題列出建議後問 user 套用哪幾條,user 同意才用 Edit tool 改。
2. **error vs warn**:能讓 `review/server.js` 抓不到資料、或破壞雙向 link 對稱性的,是 error;其他是 warn。
3. **解析規則必須與 `review/server.js` 對齊**:`splitSections`(`server.js:129`)與 `parseBreakdownLine`(`server.js:159`)若有變動,本 skill 的 Step 2.3 / 2.4 規則要同步調整。
4. **檔名以 `-` 開頭的 suffix**(`-al.md`):Bash 操作要加 `--` 或 `./` 前綴,Read / Glob tool 直接用無妨。
5. **語意檢查只用 LLM 自身知識**:不嘗試呼叫外部字典 API、不去 web search。若不確定就不報,避免 false positive。
6. **`## 筆記區` 存在性是 error,內容自由不檢查**:不檢繁簡、不檢 wiki-link、不檢任何內容。
7. 中文輸出用 zh-TW;檔案路徑保留相對於專案根的形式(`dictionary/words/…`)。
8. 不建立 `.claude/commands/`、不動其他 skill 的檔。
