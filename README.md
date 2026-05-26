# ruby_indexer_test

> Rubydex vs 純 LLM — 一個 Rust 寫的 Ruby 語意 indexer,接上 MCP 後能不能讓 LLM 更**準**、更**省**地完成「找全程式碼關聯」任務?
> 三代 benchmark 拍同一批題,只變「rubydex 怎麼用」。

📊 **三份線上報告**:
- 📘 **[V1 報告 →](https://spenguinlui.github.io/ruby_indexer_test/index.html)** — 強迫工具:「請用 rubydex」vs「請用 grep」(14 cases × 3 模型 × 2 條件)
- 📗 **[V2 報告 →](https://spenguinlui.github.io/ruby_indexer_test/v2.html)** — 放工具箱:prompt 模糊化、工具全開,只差載不載 MCP(15 cases × 3 模型 × 2 條件;含 MCP 污染勘誤後乾淨重跑)
- 📕 **[V3 報告 →](https://spenguinlui.github.io/ruby_indexer_test/v3.html)** — 選擇器:rubydex 作為「先廣 grep、結構題才驗證」的單一 selector arm,全 15 題 vs V2 兩臂(per-case 單一 judge,跨模型可比)+ 逐題詳表

---

## 一句話結論(V3 後最新)

**給模型 rubydex —— 不管「隨時可用」(V2) 還是「叫它自己路由」(V3 selector) —— 在三個模型上都比純 grep 好;但「叫模型自己當選擇器」這條 prompt 捷徑只多贏一點點,真正的槓桿在外層 orchestrator(V4)。**

- value 題品質:**grep < rubydex < selector 三模型全部單調上升**(Haiku 5.5→6.5→7.0、Opus 6.5→7.5→8.0、Codex 7.5→8.0→8.5 /9)
- rubydex 加分集中在 **namespace / constant 計數 / rename 影響面** 這類「結構題」 —— 跟機器題的發現一致
- **越弱的模型幫越多**(Haiku +1.5、Codex +1.0):強模型純 grep 已接近天花板
- ⚠️ **selector 路由不可靠**:模型實際呼叫 rubydex 又稀疏又沒對準題型(結構主場常 0 次、字串盲區反而猛叫)→ 要靠外層 orchestrator 主動路由,不能丟工具箱賭模型自己挑

## rubydex 何時該用 / 何時不該用

| ✓ 該用(結構題,V3 確認正向) | ✕ 不該用(盲區,純付 MCP context 稅) |
|---|---|
| rename 影響面(`find_constant_references`) | 字串 / YAML 引用(Sidekiq、polymorphic source_type) |
| 多 namespace 同名 class 區分(case-4 / 4A) | monkey-patch / 重開 class |
| 短形式 ↔ FQN 計數(case-8) | hash 動態 dispatch / method_missing |
| has_one / delegate 關聯鏈(\*要先廣 grep,別讓 index 跳過 grep) | 動態組合 method / class 名 |
| routes helper 反查 controller#action | 外部資源 yml / locales / i18n key |

## 三代怎麼問同一批題

| | V1(index.html) | V2(v2.html) | V3(v3.html) |
|---|---|---|---|
| **對 rubydex 的姿勢** | 強迫:點名「用 rubydex」/「用 grep」 | 放工具箱:載不載 MCP 由環境決定,模型自選用不用 | **選擇器**:rubydex 在工具箱 + 策略明寫「先廣 grep、結構引用才用 rubydex 驗證、盲區走 grep」,單一 arm |
| **prompt** | 詳細(給 FQN / 形式 / RESULT_SUMMARY) | 最大模糊(只給短名) | 模糊 + 策略 header |
| **對照** | with vs without(各 1 arm) | with vs without(各 1 arm) | 跟 V2 兩臂並排當對照 |
| **cases** | 14 | 15(新增 case-14 rename) | 15(沿用) |
| **結論方向** | 工具陷阱:rubydex 開了反而鎖死 Haiku case-1 | 修 MCP 污染後 rubydex 對 recall 幫忙偏正,但仍混合 | **grep < rubydex < selector 三模型單調升**,加分集中在結構題;selector 路由不可靠 → 外層 orchestrator(V4) |

## 這個 repo 有什麼

```
ruby_indexer_test/
├── index.html              ← V1 互動報告(GitHub Pages 首頁)
├── v2.html                 ← V2 互動報告(模糊 prompt × 自然選用 + MCP 污染勘誤)
├── v3.html                 ← V3 互動報告(selector × 全 15 題 + 逐題詳表 + 模型 toggle)
├── v3_compare_data.json    ← V3 主表原始資料(machine recall + value 判讀,3 模型 × 3 條件)
├── v3_detail_data.json     ← V3 逐題詳表原始資料(5 臂 × 全指標:正確性 / 讀檔 / rubydex / context / token / 速度)
├── v3_data.json            ← V3 早期 6 題版資料(歷史)
├── report.md               ← V1 完整方法論 + 數據 + 結論
└── README.md               ← 本檔
```

三份 HTML 都是單一自含檔(無 build step、僅載 Google Fonts),頂部 vswitch 可互相切換。

## 重要的方法論修正(時序記錄)

V1 → V2 → V3 一路發現並修正了三個問題,這裡集中講:

1. **MCP 載入污染**(2026-05-23 修):V1/V2 原本「不載 MCP」對照組,因 CLI auto-discovery 仍把 rubydex 載進來 → with/without 差異被污染。修法:Claude 加 `--strict-mcp-config`、codex 加 `--ignore-user-config`。乾淨重跑後 V2 結論方向改變(rubydex 對 recall 的幫助比污染版明顯)。詳見 v2.html 勘誤段。
2. **answer key 漂移**(2026-05-24 修):case-12 舊 key 寫 `ledger_inferred_types=1`,核 live code 發現實際是 4(`LedgerFactory#steps` 回 `build_ledger_by_user` 的 4 分支)→ 答 4 的模型其實是對的;key 已修,v2.html 受影響的判定也更新。
3. **跨模型 judge 變異**(2026-05-24 修):V3 第一輪每模型一個 subagent 判讀 → 不同 judge 嚴格度不一致 → 跨模型分數失真。改成「per-case 一個 judge 判該題全 9 格」後,結論才真正可比(`grep < rubydex < selector` 三模型一致單調升是這版的乾淨結論)。

## 關於匿名化

本報告所有**數字(recall / counts / token / cost / 速度)皆為真實測試結果**。為保護受測來源:
- 生產專案名一律 `core_web` / `e_trading` / `sf` 等中性化
- 業務 class / 模組 / 路徑已換成中性虛構名(保留結構難度、抽掉業務語意)
- 深入真實程式碼的逐題 ground-truth 推導**不在公開範圍**

## 測試環境

- **對照模型**:Claude Haiku 4.5、Claude Opus 4.7 (low effort)、GPT-5.5(codex CLI)
- **測試期間**:V1 2026-05-16~21、V2 2026-05-22~23(乾淨重跑 24)、V3 2026-05-24
- **目標**:3 個生產 Rails 專案(動態 DSL、Zeitwerk autoload、Sidekiq、i18n、polymorphic association 等真實場景)
- **採樣**:n=1(單次採樣,變異大;所有結論皆方向性訊號)

## 下一步(V4)

V3 證明的命題:**「rubydex 對結構題有料,但靠 prompt 叫模型自己路由不可靠」→ 要外層 orchestrator 主動按 intent 路由**。V4 = 把這層做出來:
```
LLM ─query─▶ Orchestrator ─┬─▶ ripgrep / rubydex / ruby-lsp-rails / rails runtime
                           └─▶ intent 路由 + 打分 → top-k 候選 ──▶ LLM
```
分階段建議:**V4-α**(MVP:rg + rubydex + intent 分類)→ **V4-β**(+ruby-lsp-rails)→ **V4-γ**(+rails runtime introspection)。未開工。
