---
title: "Part 2 · 把 AI 當聰明但健忘的初級工程師來管理"
date: 2026-05-09
weight: 2
tags: ["AI", "Claude Code", "harness", "multi-agent", "工程流程"]
summary: "用 Claude Code 多 agent 流程開發這個專案的踩坑。核心是「規則寫進文件沒用，要有 enforcement 點」——踩坑怎麼變成擋得住的 test / workflow gate。"
ShowToc: true
TocOpen: false
---

用 Claude Code 多 agent 流程開發一個壓測專案——以及為什麼「規則寫進 prompt」幾乎沒用，要把它變成跑得起來的 test 才擋得住。

> AI agent 會通過 lint、會說「完成了」，然後在第一次真的執行時連環爆。我學到的不是怎麼下更好的 prompt，是怎麼設計一個流程，讓 agent 的不可靠變成可被攔截的。

## The Setup — 我怎麼組這個 harness {#harness-setup}

不是「開一個對話叫 AI 寫 code」。是一條 multi-agent workflow，每個 agent 有固定職責，彼此用**文件**傳 context：

| Agent | 職責 | 不准做 |
| --- | --- | --- |
| **planner** | 把模糊需求 → spec + contract（+ 必要時 ADR）| 不寫 code |
| **generator** | 嚴格照 contract 實作 + 寫測試 + 跑測試 | 不重新討論需求 |
| **evaluator** | 驗收：跑測試、查 contract 每條打勾、出 QA report | 不改實作（只回報）|

context 不靠對話記憶傳遞，靠檔案：

- `docs/specs/` — 需求規格
- `docs/contracts/` — 給 generator 的明確交付清單（evaluator 逐條驗）
- `docs/decisions/` — ADR，記錄「為什麼這樣決定」，跨 sprint 不丟失
- `docs/CHANGELOG.md` — 每個 agent 跑完自己追加一條，下一個 agent / 下一個 sprint 讀得到

{{< callout type="neutral" title="為什麼這樣設計" >}}
agent 的 context window 會爆、對話會被 compact、跨 sprint 記憶會斷。**把決策外部化成檔案 = 給健忘的協作者一份不會丟的工作交接。** 這跟人類團隊用 ADR / spec 是同一個道理，只是對 AI 更剛性需求。

流程：`需求 → planner → contract → generator → tests → evaluator → QA report → ship`。箭頭上傳遞的是**檔案**不是**記憶**。
{{< /callout >}}

## AI 差點把錯誤結論寫進報告 {#the-save}

{{< callout type="accent" title="★ cross-link — 這是 Part 1 §5 同一件事的另一面" >}}
第一輪壓測數據出來，顯示「拆 worker 反而慢 8%」。AI（在我引導下跑完測試）準備把這結論寫進 perf report。

**攔下來的不是 AI，是『這數據反直覺』的人類判斷。** 我要求深究，才挖出觀測性 confound（OTel 揹在被測服務上）。完整拆解見 [Part 1 §5「我把實驗搞砸了」]({{< relref "posts/part1-load-test.md#the-mistake" >}})。
{{< /callout >}}

教訓對 AI 協作的意義：

- AI 很會「把手上的數據整理成漂亮結論」——這正是危險的地方，它不會自己懷疑實驗設計。
- **人類在迴圈裡的價值不是寫 code，是『這不對勁』的直覺 + 要求重做的決斷。**
- Prompt 再好都不會自動產生「先懷疑自己的測量方法」這種後設判斷，那必須是人帶進來的。

這就是為什麼 review 不能外包給生成它的 agent。

## 規則寫進文件沒用，要有 enforcement 點（Part 2 重點） {#enforcement}

每個坑的生命週期都一樣：

```text
踩坑 → 寫進 CLAUDE.md / 文件提醒 → 下次還是踩 → 變成跑得起來的 test / workflow gate → 才真的擋住
```

**案例 A · `:latest` image 定時炸彈**

- 坑：`jaegertracing/all-in-one:latest` 被 Jaeger v2 淘汰，docker compose 起不來。文件早就寫「image 要 pin 版本」。
- 為什麼文件擋不住：generator 寫 compose 時沒回頭讀那條，文件規則沒有執行點。
- enforcement：`tests/test_no_latest_images.py` — 掃所有 compose / Dockerfile，出現 `:latest` 直接 test fail。規則變成 CI 會紅的東西。

**案例 B · shell pipeline lint pass 但跑不起來**

- 坑：seed.js 的 docstring 寫了一條 k6 解析 pipe，依舊印象寫「k6 console.log 寫 stdout」。k6 v0.50+ 改寫 stderr + 包成 `level=info msg="..."`，pipe 完全解析不到，`tokens.json` 是空的。syntax-only 測試抓不到。
- 為什麼文件擋不住：沒有任何環節「真的執行那條 pipe 一次」。generator 只 lint，evaluator 只查 contract 打勾，CI 用 SQLite。
- enforcement：改 generator 工作流，加一條強制步驟——**任何 shell pipeline 必須實際 smoke run 一次**（哪怕只驗「執行成功 + 輸出非空」）才能算完成。把「實跑」變成 agent 定義裡的硬性 gate，不是文件裡的軟提醒。

**案例 C · 最諷刺的一個**

- 坑：修案例 B 時，新 pipe 寫進 JSDoc 註解，sed pattern `s/.*msg=...*/\1/p` 裡的 `.*/` 被當成 JSDoc 結束符 `*/`，node `--check` 直接 SyntaxError。
- 諷刺點：完美 mirror 整個主題——寫了沒跑 → lint 還沒到那行 → 真的 node --check 才爆。test 早就在跑 node --check，但 generator 寫 docstring 當下沒跑那個 test。
- enforcement：同案例 B 的 generator workflow gate（smoke / 相關 test 必跑）+ sed 改用 `|` delimiter 避開 `/`。

{{< callout type="bad" big="true" >}}
「我把規則寫進 CLAUDE.md」幾乎等於沒做。規則要變成一個會讓 CI 變紅、或 agent 定義裡跑不過就不算完成的東西，才真的存在。
{{< /callout >}}

## Prompt Pitfalls — 下 prompt 踩過的坑 {#prompt-pitfalls}

1. **批次操作含 metacharacter = 自爆** — 我叫 agent 用一行 batch sed 改 8 個檔案的 pipe。replacement 字串裡含 `|`，跟 sed 的 `|` delimiter 互吃，原片段被疊加 4 次，**寫崩 7 個檔案**。教訓：替換字串含 regex / shell metacharacter（`| ( ) / $`）時，永遠別用 batch sed。改用 Edit tool 的 `replace_all` 一檔一檔做——慢但安全。
2. **「請依研究結果實作」這種 prompt 把理解外包出去** — 把「你研究完就修 bug」丟給 agent = 它的綜合判斷會很淺。有效的 prompt 是我自己先讀懂、給出檔案路徑 + 行號 + 具體要改什麼。**不要把『理解』delegate 出去**，delegate 的是執行，不是判斷。
3. **agent 說「完成了」≠ 真的能跑** — generator 回報 contract 全打勾、測試綠燈。第一次本地實跑連撞 3 個坑。「lint 通過」「測試綠」只覆蓋受控環境。real-environment smoke 是分開的、必要的 gate。**驗收 agent 的產出時，看實際 diff / 實際執行，不信它的總結。**
4. **沙箱限制造成的「合理範圍切割」會留下驗證盲區** — Sprint C 故意把「腳本完備」跟「實際數字」切開，這切割本身合理，但**沒有任何後續環節補回『實跑驗證』**，盲區就一路漏到第一次本地跑。每次因環境限制做 scope 切割，要顯式問「被切掉的那塊由誰、在哪個環節補驗」。

## Takeaways {#takeaways}

{{< callout type="accent" big="true" >}}
1 · 把 AI 當健忘的初級工程師——context 用檔案傳（spec / contract / ADR / CHANGELOG），不靠對話記憶；職責用 agent 邊界切，不讓一個 agent 校驗自己。
{{< /callout >}}

{{< callout type="accent" big="true" >}}
2 · 規則沒有 enforcement 點就不存在——寫進 CLAUDE.md 的提醒會被漏讀，變成 CI 會紅的 test / agent workflow 跑不過的 gate 才真的擋住。
{{< /callout >}}

{{< callout type="accent" big="true" >}}
3 · 人在迴圈裡的價值是判斷不是打字——AI 很會把數據整理成漂亮結論，「這不對勁、重做」這種後設判斷必須人帶進來。
{{< /callout >}}

---

← 回 [Part 1 · 壓測實驗]({{< relref "posts/part1-load-test.md" >}})
