# ruby_indexer_test

> Rubydex vs 純 LLM — 誰找得全 Ruby 程式碼關聯?
> 一個 Rust 寫的 Ruby 語意 indexer,接上 MCP 後能不能讓 LLM 更**準**、更**省**地完成「找全程式碼關聯」任務?

**14 cases × 6 條件 = 84 結果點**,跨 3 個生產 Rails 專案(匿名 App-A / B / C),3 個模型對照,以 **recall-first** 評分。

📊 **[線上互動報告 →](https://spenguinlui.github.io/ruby_indexer_test/)**　·　📄 [完整報告 report.md](./report.md)

> 🆕 **[V2 報告 →](https://spenguinlui.github.io/ruby_indexer_test/v2.html)** — 把 prompt 最大模糊化、開放全部工具(含 Glob)、with/without 只差載不載 MCP(**自然選用**),外加 refactor 影響面 case。15 cases × 3 models × 2 = 90 結果點。

---

## 一句話結論

- **rubydex 是「結構性查詢」的精準工具,但對 Rails 動態 / 文本場景幫不上,甚至會鎖死 Claude(case-1)。**
- 以 recall-first 衡量,「不開 rubydex 的 Haiku」和「開 rubydex 的 GPT-5.5」並列最穩 —— 各只漏 **1** 個 hit。
- 真正的「LLM 找不到」幾乎不存在 —— 錯誤集中在**過報**而非漏報。**純 LLM 的程式碼搜尋能力被低估了。**

## 數據亮點

| 條件 | 不漏率(recall-first) | 漏掉總 hits |
|---|---|---|
| Haiku · grep | **13/14 (93%)** | **−1** |
| Codex · +rubydex | **13/14 (93%)** | **−1** |
| Codex · grep | 12/14 (86%) | −3 |
| Opus · grep | 11/14 (79%) | −3 |
| Haiku · +rubydex | 10/13 (77%) | **−27** |
| Opus · +rubydex | 10/14 (71%) | **−29** |

> 開了 rubydex 的兩個 Claude 條件 recall 反而最差,主因是 case-1 的「rubydex 鎖死陷阱」。詳見[報告](./report.md)。

## rubydex 何時該用

| ✓ 該用(結構性查詢) | ✕ 不該用(文本 / 動態) |
|---|---|
| constant / class / module 名引用 | 字串 literal / symbol |
| 多 namespace 同名 class 區分 | 動態組合 method 名 / class 名 |
| 短形式 ↔ FQN 同 class 識別 | 外部資源 yml / routes / locales |

## 這個 repo 有什麼

```
ruby_indexer_test/
├── index.html     ← V1 互動報告(GitHub Pages 首頁)
├── v2.html        ← V2 互動報告(模糊 prompt × 自然選用 × refactor)
├── report.md      ← 完整方法論 + 數據 + 結論
└── README.md      ← 本檔
```

`index.html` / `v2.html` 都是單一自含檔案(無 build step、無外部相依,僅載 Google Fonts),共用同一套樣式,頂部可互相切換,可直接掛 GitHub Pages。

## V1 vs V2 差在哪

| | V1（index.html） | V2（v2.html） |
|---|---|---|
| prompt | 詳細(給 FQN / 形式 / 排除 / RESULT_SUMMARY) | **最大模糊**(只給短名) |
| 工具 | 文字強迫「用 rubydex」或「用 grep」 | **不限制,模型自選**(含 Glob) |
| with/without 差別 | 強迫用 vs 強迫不用 | **只差載不載 MCP** |
| 額外 | — | **refactor 影響面 case** |

**V2 核心結論**:自然選用下,把 rubydex 放進工具箱對 recall **一致中性偏負**(6 題 ×3 模型 = 18 組對照中從沒提升,唯一例外 codex case-1),Opus 端成本還 +53%。但 V1 的「rubydex 鎖死 Claude」陷阱在 V2 **沒有重演** —— 模型會自己在 rubydex 沒用的題改走 grep。rubydex 的價值高度集中在「對的題(namespace)+ 對的模型 + 明確指示」這個窄帶。

## 關於匿名化

本報告所有**數字(hits / GT / token / cost / 排名)皆為真實測試結果**。為保護受測來源:

- 生產專案名一律以 `App-A / B / C` 表示
- 業務 class / 模組 / 路徑已換成中性虛構名(保留結構難度、抽掉業務語意)
- 深入真實程式碼的逐題 ground-truth 推導**不在公開範圍**

## 測試環境

- **對照模型**:Claude Haiku 4.5、Claude Opus 4.7 (low effort)、GPT-5.5 (codex CLI)
- **測試期間**:2026-05-16(Claude)、2026-05-20/21(codex / GPT-5.5)
- **目標**:3 個生產 Rails 專案(動態 DSL、Zeitwerk autoload、Sidekiq、i18n、polymorphic association 等真實場景)
