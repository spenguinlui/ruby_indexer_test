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

### 3.1 逐題詳解（要找什麼 / 為什麼難追）

**case-1 · has_one + delegate 鏈式** — 某 model 用 `has_one :profile` 建立關聯,presenter 又用 `delegate :phone, to: :profile` 把 phone 委派出去。要找出整個 codebase 裡所有最終會碰到 `Account::Profile#phone` 的點,分四類:(A) 直接 `profile.phone`、(B) 走關聯 `user.profile.phone`、(C) 透過 presenter 的 delegate 隱式呼叫、(D) 透過 `send(:phone)` 動態呼叫。
**難在**:`phone` 在 Profile 裡**根本沒有 `def phone`** — 是 `delegate` 巨集在執行期生成的,grep `def phone` 找不到;且要判斷 `user.profile` 的型別需要型別推論。

**case-2 · AASM public_send 動態 event** — AASM 是 Ruby 狀態機 gem,狀態轉換稱 event。要找全觸發 event 的呼叫,含直接(`order.approve!`)與動態(`order.public_send("do_#{state}!")`),並標出 silent bug(動態拼出的 event 名不存在、執行到才炸的)。
**難在**:動態方法名是執行期用字串拼出來的,原始碼裡**沒有那個字面字串**,grep 找不到。

**case-3 · class_eval heredoc 動態定義** — `class_eval` 把一段字串當 class 內容在執行期執行;heredoc 是多行字串寫法。要找方法定義位置 + 這段一共動態生成幾個方法。
**難在**:`def` 寫在「字串」(heredoc)裡再交給 class_eval,對解析 **AST** 的工具來說只是字串常數,不展開。

**case-4 · 多 namespace 同名 class(給 FQN)** — 給定 **FQN**(如 `AccountsReceivable::Models::Invoice`)找全引用;prompt 故意寬鬆,看模型會不會主動連短形式 `Invoice` 一起找。
**難在**:grep 完整 FQN **漏短形式**,grep 短形式又**誤抓其他 namespace 的同名 class**。

**case-4A · 多 namespace 同名(不給 FQN)** — 不給 FQN,要先自己找出有幾個同名 class 分散在不同 namespace,再按 namespace 分組統計引用。
**難在**:grep 一個短名會把所有 namespace 引用**混在一起**,無法判斷某次引用指哪一個。

**case-5 · Sidekiq YAML 字串引用 worker** — Sidekiq 是背景工作框架,worker 是執行背景工作的 class。要找全某 worker 的引用,含寫在 `config/sidekiq.yml` 排程裡的字串引用。
**難在**:rubydex 只解析 `.rb`,**看不到 `.yml`**;這個 worker 主要被 YAML 字串排程引用,語意 indexer 完全看不到。

**case-6 · 重開 Object monkey patch** — monkey patch 指執行期重開既有 class(連 `Object` 都能開)加方法。要找某個加在 `Object` 上的方法定義 + 所有 caller。
**難在**:加在 `Object` 上**所有物件都有**這方法,呼叫端看似普通呼叫、無線索;定義又藏在不起眼的 helper。

**case-7 · polymorphic source_type 自訂字串** — polymorphic association 用 `xxx_type` 字串欄記錄「對應哪個 model」。要找全寫入 `source_type: '<str>'` 的點,推對應 class,標出對不到 class 的 silent bug。
**難在**:`source_type` 存的是**字串字面值**非常數,不在 index 裡;字串→class 對應邏輯散落各處。

**case-8 · Zeitwerk autoload + 短形式常數** — Zeitwerk 是 Rails 自動載入器,依「檔名↔常數名」規則自動載入,所以很多地方寫短名。要找全某常數引用:FQN + 短形式 + 字串。
**難在**:短形式合法,同一常數**有的寫全名有的寫短名**,grep 一種就漏另一種(語意 indexer 該贏 grep 的場景)。

**case-9 · Rails callback symbol → method** — callback 是 model 生命週期掛鉤(`before_save :foo`)。要列所有 callback,分 symbol / block,並驗證 symbol 指到的方法存在。
**難在**:`:foo` 是 **symbol 不是常數**,語意 indexer 不認「symbol 綁到哪個方法」這條關係。

**case-10 · Rails i18n key tree + 動態 lookup** — i18n 把翻譯放 `.yml`,用 key 取用(`t('users.title')`)。要找全某段 key 在 `.rb` / `.erb` 的引用,含動態拼的 key。
**難在**:rubydex 不看 yml;動態 key(`t("users.#{section}.title")`)要先**推 `section` 的可能值**才知道指到哪。

**case-11 · Rails nested resources path helper** — path helper 是路由自動生成的網址方法(`edit_admin_facility_path`)。給 helper 名,反推 routes 宣告 + `controller#action` + 所有 caller。
**難在**:helper **沒有實際 `def`**(路由依命名規則生成);routes.rb 只寫 `resources :facilities`,helper 名經命名規則展開,中間無字面字串可 grep。

**case-12 · method_missing 轉發 inner object** — `method_missing` 是「呼叫不存在方法時」的攔截點,常用來轉發給內部物件。要列 class 自己 `def` 的方法,並推 method_missing 轉發給內部哪個物件(推型別)。
**難在**:被轉發的方法與自己定義的**呼叫起來一模一樣**,看不出哪些是轉發;要推 initializer 內部物件型別。

**case-13 · Hash dispatch table(字串→class)** — dispatch table 用 Hash 把字串 key 對應到 class / 邏輯,靠 `HASH[key]` 查表決定行為。要找查表位置(`SOURCE_MODELS[source_type]`)+ 類似 pattern。
**難在**:`HASH[key]` 是**字串查表**,key 非常數,語意 indexer 不當成「引用某 class」的關係。

### 3.2 名詞 / 縮寫速查

| 縮寫 / 術語 | 全名 | 一句話 |
|---|---|---|
| **FQN** | Fully Qualified Name | 完整限定名稱,帶完整命名空間的名字(`A::B::Invoice`);相對於只寫 `Invoice` 的短形式 |
| **AST** | Abstract Syntax Tree | 抽象語法樹;寫在字串裡的程式碼不會被展開成 AST,解析工具看不到 |
| **MCP** | Model Context Protocol | 讓 LLM 接外部工具的協定;rubydex 透過 MCP 提供語意查詢 |
| **GT** | Ground Truth | 正確答案基準,獨立用 grep + 人工分類精算 |
| **recall / precision** | 召回率 / 精確率 | recall = 該找的有多少被找到(漏報壓低);precision = 找到的有多少對(過報壓低)。本報告 recall-first = 寧過報別漏 |
| **AASM** | Acts As State Machine | Ruby 狀態機 gem,狀態轉換稱 event |
| **Zeitwerk** | Rails 自動載入器 | 依「檔名↔常數名」自動載入,讓很多地方可寫短名 |
| **i18n** | internationalization | 國際化;翻譯放 `.yml` 用 key 取用(中間 18 字母縮寫) |
| **silent bug** | 靜默錯誤 | 不報錯但行為已錯;如動態方法名不存在、執行到才炸 |
| **monkey patch** | 猴子補丁 | 執行期重開既有 class(連 Object)加方法 |
| **delegate** | 委派 | Rails 巨集,把方法呼叫轉交關聯物件;被委派方法無實際 `def` |
| **polymorphic** | 多型關聯 | 用 `xxx_type` 字串欄記錄對應哪個 model |
| **callback** | 生命週期掛鉤 | model 在存檔/刪除等時機自動觸發的鉤子 |
| **method_missing** | 缺方法攔截 | 呼叫不存在方法時的攔截點,常用來轉發 |
| **heredoc** | 多行字串語法 | `<<~SQL ... SQL`;裡面像程式碼也只是字串 |
| **class_eval** | 執行期求值 | 把字串當 class 內容在執行期執行,可動態定義方法 |
| **dispatch table** | 分派表 | 用 Hash 把字串 key 對應到 class / 邏輯,`HASH[key]` 查表 |
| **Sidekiq** | 背景工作框架 | Ruby background job 框架;worker 常用 YAML 排程 |

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
