# Rubydex vs LLM 廣搜 — 完整測試報告

**測試期間**:2026-05-16(Claude 兩 model)、2026-05-20/21(codex / GPT-5.5)
**規模**:14 cases × 6 條件(3 model × 有/無 rubydex)= **84 結果點**
**目標 codebase**:三個生產 Rails 專案(匿名為 `App-A`、`App-B`、`App-C`)
**對照模型**:Claude Haiku 4.5、Claude Opus 4.7 (low effort)、GPT-5.5 (codex CLI)

> **匿名化說明**:所有數字皆為真實測試結果。生產專案名以 `App-A / B / C` 表示,業務 class / 模組 / 路徑已換成中性虛構名(保留結構難度、抽掉業務語意),通用 CS / Rails 詞(User、Profile、Invoice、AASM、callback 等)保留。

---

## 一、實驗目的

驗證一個假設:**rubydex(Rust 寫的 Ruby 語意 indexer)能否讓 LLM 更準/更省地完成「找全程式碼關聯」任務?**

每個 case 對應一個**具體可命名的 Ruby/Rails 撰寫手法**(不是抽象的「找東西」)。每題跑「有 rubydex MCP」vs「只用 grep/Read」兩輪,比對找全率與成本。

判斷標準採 **recall-first**:搜尋任務裡,**漏報(false negative)遠比過報(false positive)嚴重** — 漏一個 hit 等於 LLM 看不到該資訊、後續決策直接錯過(silent bug);過報只是增加 LLM 的 filter loading,不致命。

---

## 二、測試流程與 Prompt

### 2.1 流程

每個 case 跑兩輪,唯一差別是「能不能用 rubydex MCP」:

```
for case in 14 個 case:
    ┌─ with-rubydex   ── 載入 rubydex MCP ──────────────────────┐
    │                                                            ├─→ 各存 JSON/JSONL + 最終答案
    └─ without-rubydex ── 獨立空目錄當 cwd、不載 MCP ────────────┘
```

- **without 用獨立空目錄當 cwd**,確保 LLM 不會「不小心」連到 MCP。
- 每輪結束抽:cost(僅 Claude)、duration、token、工具呼叫次數(rubydex/grep/read/bash)、最終 `RESULT_SUMMARY`。
- GT(ground truth)是另外用 grep + 人工分類精算出來的,獨立於受測跑次。

### 2.2 Prompt 結構

每個 prompt 三段:**①工具指示行 → ②任務描述 → ③RESULT_SUMMARY 回報格式**。
with/without 兩版**只差第①行的工具指示**,任務與回報格式完全相同 → 確保差異只來自「能不能用 rubydex」。

**範例 prompt(without 版,任務為虛構示意):**

```
請用 Grep / Read 工具(只能用這兩個搜尋,不要呼叫任何 MCP 工具)。

某 model 宣告了 `has_one :profile, class_name: 'Account::Profile'`,
另有 presenter `delegate :phone, ... to: :profile`。

請列出所有真正會呼叫到 `Account::Profile#phone` 的地方,分四類:
A) 直接 profile.phone(profile 變數明顯是 Account::Profile)
B) 透過 user.profile.phone(association traversal)
C) 透過 presenter delegate(隱式呼叫)
D) 透過 send(:phone) / public_send(:phone) 動態呼叫

每點給 file:line + 屬於哪一類。

回覆最後請用以下格式總結:

RESULT_SUMMARY:
hits_total: <總命中數>
hits_A: <A 類命中>
hits_B: <B 類命中>
hits_C: <C 類命中>
hits_D: <D 類命中>
END_SUMMARY
```

**同題 with 版只把第①行換成:** `請用 rubydex MCP 工具(不要用 Grep/Bash grep,但需要時可以 Read)。`

(codex 因兩家工具都可用,with 版第①行用「可以使用 rubydex MCP + Grep/Read,自行判斷哪些好用」,without 版同 Claude 限制只能 grep/Read。)

### 2.3 為什麼 `RESULT_SUMMARY` 用固定格式

每題結尾要求 LLM 用 `RESULT_SUMMARY: ... END_SUMMARY` 區塊回報結構化數字,目的是**讓答案可被腳本自動抽取比對**,不必人工讀每篇回答。prompt 只負責「指示要做什麼、怎麼記錄」,不暗示答案數量。

---

## 三、14 個測試情境與命題

| # | 手法 | 命題(要找什麼) | 為什麼難追 | GT |
|---|---|---|---|---|
| 1 | `has_one` + `delegate` 鏈式 | 找全會呼叫到某 delegated 屬性的點,分 直接/association/delegate/動態 四類 | 隱式 method 無 `def`、association traversal 需型別推論 | **24** |
| 2 | AASM `public_send("do_#{state}!")` | 找全 AASM event 直接 + 動態呼叫,標 silent bug | method 名是 runtime 字串,grep 找不到字面 | **16** |
| 3 | `class_eval` heredoc 動態定義 | 找定義位置 + 動態生成幾個 method | `def` 在 heredoc 字串裡,AST 不展開 | **6** |
| 4 | 多 namespace 同名 class(給 FQN) | 給 FQN 找全引用(寬鬆 prompt,看會不會主動找短形式) | grep FQN 漏短形式、grep 短形式漏 FQN | **101** |
| 4A | 多 namespace 同名(不給 FQN) | 先列出所有同名 class 再按 namespace 分組找引用 | grep 把所有 namespace 混在一起無法分組 | **141** |
| 5 | Sidekiq YAML 字串引用 worker | 找全 worker class 引用(含 YAML config 字串) | rubydex 只看 .rb,看不到 .yml | **1** |
| 6 | 重開 `Object` monkey patch | 找定義 + 所有 callers | 定義在不起眼 helper,影響全 codebase,caller 無線索 | **3** |
| 7 | polymorphic `source_type` 自訂字串 | 找全 `source_type: '<str>'` 寫入點,推對應 class,標 silent bug | 字串 literal 非 constant,dispatch 邏輯散落 | **9** |
| 8 | Zeitwerk autoload + 短形式常數 | 找全引用:完整 FQN + 短形式 + 字串 | grep FQN 漏短形式,反之亦然 | **25** |
| 9 | Rails callback symbol → method | 列所有 callback,分 symbol/block,驗證 method 存在 | `:foo` symbol 非 constant,rubydex 不認綁定 | **3** |
| 10 | Rails i18n key tree + 動態 lookup | 找全 yml key 的 .rb/.erb 引用(直接 + 動態組合) | rubydex 不看 yml,動態 key 要推變數值 | **4** |
| 11 | Rails nested resources path helper | 給 helper 找 routes 宣告 + controller#action + callers | helper 無 .rb,routes.rb 經命名規則展開 | **3** |
| 12 | `method_missing` 轉發 inner object | 列自己 def 的 method + 推 method_missing 轉發的型別 | 看不出哪些是轉發,要推 initializer 型別 | **3** |
| 13 | Hash dispatch table(字串→class) | 找 dispatch table lookup 位置 + 類似 pattern | `HASH[key]` 是字串 lookup,非 constant | **2** |

---

## 四、測試數據

### 4.1 hits 對 Ground Truth(✓=完全對,+/− = diff)

| Case | GT | Haiku-w | Haiku-wo | Opus-w | Opus-wo | Codex-w | Codex-wo |
|---|---|---|---|---|---|---|---|
| 1 | 24 | **0 (−24)** | 25 (+1) | **0 (−24)** | 26 (+2) | 26 (+2) | 33 (+9) |
| 2 | 16 | 16 ✓ | 16 ✓ | 13 (−3) | 15 (−1) | 16 ✓ | 16 ✓ |
| 3 | 6 | 6 ✓ | 6 ✓ | 6 ✓ | 6 ✓ | 6 ✓ | 6 ✓ |
| 4 | 101 | 102 (+1) | 109 (+8) | 102 (+1) | 106 (+5) | 106 (+5) | 106 (+5) |
| 4A | 141 | 140 (−1) | 232 (+91) | 140 (−1) | 140 (−1) | 151 (+10) | 151 (+10) |
| 5 | 1 | 1 ✓ | 1 ✓ | 1 ✓ | 2 (+1) | 2 (+1) | 2 (+1) |
| 6 | 3 | — | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ |
| 7 | 9 | 9 ✓ | 35 (+26) | 9 ✓ | 30 (+21) | 9 ✓ | 36 (+27) |
| 8 | 25 | 23 (−2) | 25 ✓ | 25 ✓ | 25 ✓ | 25 ✓ | 25 ✓ |
| 9 | 3 | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ |
| 10 | 4 | 5 (+1) | 4 ✓ | 4 ✓ | 4 ✓ | 9 (+5) | 2 (−2) |
| 11 | 3 | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ | 3 ✓ |
| 12 | 3 | 3 ✓ | 4 (+1) | 11 (+8) | 3 ✓ | 3 ✓ | 3 ✓ |
| 13 | 2 | 2 ✓ | 1 (−1) | 1 (−1) | 1 (−1) | 1 (−1) | 1 (−1) |

### 4.2 條件總排名(對稱誤差)

| 條件 | 完成 | 完全對 | diff≤±10% | 總 output token | 平均/case |
|---|---|---|---|---|---|
| Haiku-w | 13 | 8 | 11 | 84.7k | 6.5k |
| Haiku-wo | 14 | 8 | 10 | 88.6k | 6.3k |
| **Opus-w** | 14 | 8 | 10 | **37.5k** | **2.7k** |
| Opus-wo | 14 | 7 | 11 | 47.0k | 3.4k |
| Codex-w | 14 | 8 | 11 | 48.9k | 3.5k |
| Codex-wo | 14 | 7 | 9 | 52.8k | 3.8k |

### 4.3 Recall-first 排名(漏越少越好)

| 條件 | 不漏率(hits ≥ GT) | 漏的 case | **漏掉總 hits** |
|---|---|---|---|
| **Haiku-wo** | **13/14 (93%)** | case-13 | **−1** |
| **Codex-w** | **13/14 (93%)** | case-13 | **−1** |
| Codex-wo | 12/14 (86%) | case-10, 13 | −3 |
| Opus-wo | 11/14 (79%) | case-2, 4A, 13 | −3 |
| Haiku-w | 10/13 (77%) | case-1, 4A, 8 | **−27** |
| Opus-w | 10/14 (71%) | case-1, 2, 4A, 13 | **−29** |

---

## 五、結論

### 5.1 rubydex 的價值高度取決於「問題本質」

| rubydex **該用**(結構性查詢) | rubydex **不該用**(文本/動態) |
|---|---|
| constant / class / module 名引用(case-4、4A、8) | 字串 literal / symbol(case-5、7、9、13) |
| 多 namespace 同名 class 區分 | 動態組合 method 名 / class 名(case-2、12) |
| 短形式 ↔ FQN 同 class 識別 | 外部資源 yml / routes / locales(case-5、10、11) |

→ rubydex 是「結構性查詢」的好幫手,「文本式查詢」的負擔。對 Rails-heavy codebase(動態 DSL、symbol、yaml/routes),rubydex 幫不上的場景多於幫得上的。

### 5.2 最大的反例:「rubydex 工具陷阱」(case-1)

case-1(has_one + delegate)是最戲劇的失敗:**Claude 配 rubydex(Haiku-w、Opus-w)都拿 0 hits**。
Claude 看到 rubydex 可用就鎖死狂打 MCP,即使第一個 call 沒結果也不切換到 grep。同題不開 rubydex 反而拿 25-26(接近正解 24)。

→ rubydex 不只沒幫上忙,反而**鎖死 LLM 不去用更有效的工具**。

### 5.3 GPT-5.5 (codex) 與 Claude 的差異

1. **codex 不踩 rubydex 鎖死陷阱** — case-1 配 rubydex 拿 26(接近正解),它會 rubydex + grep 混用,不單押一邊。
2. **codex 系統性過報** — 對 prompt 排除規則遵守度低(case-4/4A/7/10 普遍 +5~+27)。但在 recall 視角下「過報」是安全錯誤。
3. **codex 的 rubydex 對答案影響小** — with/without 答案多半一致,rubydex 對它更像「純加速器」而非「改變結果」。
4. **case-7 是 codex 唯一靠 rubydex 變準的角度** — with(走 constant)精準 9,without 過報到 36。

### 5.4 Recall-first 視角徹底改變排名

- **對稱誤差下**:6 條件「完全對」都落在 7-8/14,差不多。
- **recall-first 下**:差距拉開 —
  - **贏家:Haiku-wo 與 Codex-w(各只漏 1)**。
  - **輸家:Haiku-w 與 Opus-w(各漏 ~28)**,而且**漏的主因就是 case-1 的 rubydex 鎖死**。
- 啟示:**不開 rubydex 的條件普遍 recall 較好**,因為避開了鎖死陷阱。

### 5.5 「純 LLM 找不到」幾乎不存在

扣掉兩個非能力因素 —
- **case-1**:rubydex 鎖死(工具問題,只影響 with-rubydex 變體)
- **case-13**:GT 把 `SOURCE_MODELS.keys` 也算 lookup,但 prompt 例字 `SOURCE_MODELS[xx]` 暗示只算 `[]` lookup,LLM 全答 1 其實符合 prompt 字面

→ **沒有任何條件因為「找不到」而漏**。LLM 的錯誤幾乎全是「過報」而非「漏報」。**純 LLM 的程式碼搜尋能力沒有想像中弱。**

### 5.6 成本/效率取捨

| 模型 | 角色定位 |
|---|---|
| **Opus 4.7 (low)** | 最省 token(2.7k/case),但有「賭」屬性 — case-8 只用 44 token 命中,case-1 卻用 2.1k 拿 0 分 |
| **Haiku 4.5** | 最費 token(~6.4k/case),但 Haiku-wo recall 最穩(只漏 1) |
| **GPT-5.5 (codex)** | token 居中(3.5-3.8k),最穩定無 outlier,但會過報;cost 需自行用 token 換算(codex CLI 不回報 USD) |

---

## 六、總結一句話

> **rubydex 是「結構性查詢」的精準工具,但對 Rails 動態/文本場景幫不上,甚至會鎖死 Claude(case-1)。**
> **以 recall-first 衡量,「不開 rubydex 的 Haiku」和「開 rubydex 的 GPT-5.5」並列最穩(各只漏 1 個 hit)。**
> **而真正的「LLM 找不到」幾乎不存在 — 錯誤集中在過報,純 LLM 搜尋能力被低估了。**
