# Website Brief（Part 2）— 用 Claude Code 開發的 harness engineering 心得

> 這份文件是給 Claude Code 用來生成個人學習網站**第二部分**的內容 + 結構規格。
> 主題：用 Claude Code 多 agent 流程開發這個專案的過程中，怎麼設計 harness、怎麼下 prompt、踩過哪些坑、怎麼把教訓變成 enforcement。
> 與 Part 1 的關係：Part 1 講「壓測實驗本身」，Part 2 講「我怎麼用 AI agent 把它做出來、過程中對 AI 協作的理解怎麼演進」。同一個網站的兩個 section，可共用導覽。

---

## 0. 給生成網站的 Claude Code 的指示

- **技術選型**：跟 Part 1 同一個靜態站（GitHub Pages 部署）。同一套 Astro / 單頁 HTML + Tailwind，Part 2 是另一個錨點 section，不要另開 repo。
- **語氣**：第一人稱、誠實工程 blog。Part 2 的賣點是「我把 AI 當不可靠的初級工程師來管理，並把每次教訓變成自動化 enforcement」——不是「AI 很神」也不是「AI 很爛」，是**怎麼設計流程讓不可靠的 agent 產出可靠的結果**。
- **核心敘事**：Part 2 的靈魂是**第 4 節「規則寫進文件沒用，要有 enforcement 點」**。整段圍繞「踩坑 → 寫進文件 → 還是再踩 → 變成 test / workflow gate 才真的擋住」這個弧線。
- **用詞精確**：
  - ✅ 用「harness engineering」「multi-agent workflow」「context-passing protocol」「enforcement point / quality gate」。
  - ❌ 不要寫「我訓練了 AI」——沒有 fine-tune，是 prompt / workflow / agent 設定的工程，不是 model training。
  - ❌ 不要把 Claude Code 講成自動完成一切——重點是**人在迴圈裡設計約束**。
- **跟 Part 1 的連結**：第 3 節的 confound 故事要 cross-link 到 Part 1 第 5 節（同一件事的兩個視角：Part 1 看「實驗設計錯誤」，Part 2 看「AI 差點把錯誤結論寫進報告、我靠『數據反直覺』攔下來」）。

---

## 1. Hero / 一句話

**標題建議**：「把 AI 當一個聰明但健忘的初級工程師來管理」

**副標**：用 Claude Code 多 agent 流程開發一個壓測專案——以及為什麼「規則寫進 prompt」幾乎沒用，要把它變成跑得起來的 test 才擋得住。

**一句話總結**（放 hero 下方）：
> AI agent 會通過 lint、會說「完成了」，然後在第一次真的執行時連環爆。我學到的不是怎麼下更好的 prompt，是怎麼設計一個流程，讓 agent 的不可靠變成可被攔截的。

---

## 2. 我怎麼組這個 harness（章節：The Setup）

不是「開一個對話叫 AI 寫 code」。是一條多 agent 流水線，每個 agent 有固定職責，彼此用**文件**傳 context：

| Agent | 職責 | 不准做 |
| --- | --- | --- |
| **planner** | 把模糊需求 → spec + contract（+ 必要時 ADR） | 不寫 code |
| **generator** | 嚴格照 contract 實作 + 寫測試 + 跑測試 | 不重新討論需求 |
| **evaluator** | 驗收：跑測試、查 contract 每條打勾、出 QA report | 不改實作（只回報） |

**context 不靠對話記憶傳遞，靠檔案**：

- `docs/specs/` — 需求規格
- `docs/contracts/` — 給 generator 的明確交付清單（evaluator 逐條驗）
- `docs/decisions/` — ADR，記錄「為什麼這樣決定」，跨 sprint 不丟失
- `docs/CHANGELOG.md` — 每個 agent 跑完自己追加一條，下一個 agent / 下一個 sprint 讀得到

> 為什麼這樣設計：agent 的 context window 會爆、對話會被 compact、跨 sprint 記憶會斷。**把決策外部化成檔案 = 給健忘的協作者一份不會丟的工作交接。** 這跟人類團隊用 ADR / spec 是同一個道理，只是對 AI 更剛性需求。

網站可做成一張流程圖：需求 → planner → contract → generator → tests → evaluator → QA report → ship，箭頭上標「傳遞的是檔案不是記憶」。

---

## 3. AI 差點把錯誤結論寫進報告（章節：The Save — cross-link Part 1）

這段跟 **Part 1 第 5 節是同一件事的另一面**，務必互連。

第一輪壓測數據出來，顯示「拆 worker 反而慢 8%」。AI（在我引導下跑完測試）準備把這結論寫進 perf report。

**攔下來的不是 AI，是『這數據反直覺』的人類判斷。** 我要求深究，才挖出觀測性 confound（OTel 揹在被測服務上）。

教訓對 AI 協作的意義：
- AI 很會「把手上的數據整理成漂亮結論」——這正是危險的地方，它不會自己懷疑實驗設計。
- **人類在迴圈裡的價值不是寫 code，是『這不對勁』的直覺 + 要求重做的決斷。**
- Prompt 再好都不會自動產生「先懷疑自己的測量方法」這種後設判斷，那必須是人帶進來的。

網站呈現：放第一輪「-8%」表 → 一句「我覺得不對」→ 連到 Part 1 看完整 confound 拆解 → 回來這裡講「這就是為什麼 review 不能外包給生成它的 agent」。

---

## 4. 規則寫進文件沒用，要有 enforcement 點（章節：The Core Lesson — Part 2 重點）

這是 Part 2 的靈魂。**素材來源：`docs/incidents/2026-05-15-load-test-bootstrap.md`（一次 bootstrap 連環踩 4 個坑的 postmortem）。**

### 觀察到的模式

每個坑的生命週期都一樣：

```
踩坑 → 寫進 CLAUDE.md / 文件提醒 → 下次還是踩 → 變成跑得起來的 test / workflow gate → 才真的擋住
```

### 三個具體案例（網站做成「踩坑 → 為什麼文件擋不住 → 變成什麼 enforcement」三欄卡片）

**案例 A：`:latest` image 定時炸彈**
- 坑：`jaegertracing/all-in-one:latest` 被 Jaeger v2 淘汰，docker compose 起不來。文件早就寫「image 要 pin 版本」。
- 為什麼文件沒擋住：generator 寫 compose 時沒回頭讀那條，文件規則沒有執行點。
- enforcement：`tests/test_no_latest_images.py` — 掃所有 compose / Dockerfile，出現 `:latest` 直接 test fail。規則變成 CI 會紅的東西。

**案例 B：shell pipeline lint pass 但跑不起來**
- 坑：seed.js 的 docstring 寫了一條 k6 解析 pipe，依據「k6 console.log 寫 stdout」的舊印象。k6 v0.50+ 改寫 stderr + 包成 `level=info msg="..."`，pipe 完全解析不到，`tokens.json` 是空的。`test_k6_scripts.py` 是 syntax-only，抓不到。
- 為什麼文件沒擋住：沒有任何環節「真的執行那條 pipe 一次」。generator 只 lint，evaluator 只查 contract 打勾，CI 用 SQLite。
- enforcement：改 `~/.claude/agents/generator.md` 工作流，加一條強制步驟——**任何 shell pipeline 必須實際 smoke run 一次（哪怕只驗「執行成功 + 輸出非空」）才能算完成**。把「實跑」變成 generator 定義裡的硬性 gate，不是文件裡的軟提醒。

**案例 C：最諷刺的一個——JSDoc `*/` 被 sed pattern 提前關閉**
- 坑：修案例 B 時，新 pipe 寫進 JSDoc 註解，sed pattern `s/.*msg=...*/\1/p` 裡的 `.*/` 被當成 JSDoc 結束符 `*/`，node `--check` 直接 SyntaxError。
- 諷刺點：這條 bug 完美 mirror 整個主題——**寫了沒跑 → lint 還沒到那行 → 真的 node --check 才爆**。`test_k6_scripts.py` 早就在跑 node --check，但 generator 寫 docstring 當下沒跑那個 test。
- enforcement：同案例 B 的 generator workflow gate（smoke / 相關 test 必跑）+ sed 改用 `|` delimiter 避開 `/`。

### 一句話 highlight（網站大字）

> **「我把規則寫進 CLAUDE.md」幾乎等於沒做。規則要變成一個會讓 CI 變紅、或 agent 定義裡跑不過就不算完成的東西，才真的存在。**

---

## 5. 下 prompt 踩過的坑（章節：Prompt Pitfalls）

做成「踩坑集」清單，每條：症狀 → 根因 → 怎麼改下次 prompt / 流程。

1. **批次操作含 metacharacter = 自爆**
   - 我叫 agent 用一行 batch sed 改 8 個檔案的 pipe。replacement 字串裡含 `|`，跟 sed 的 `|` delimiter 互吃，原片段被疊加 4 次，**寫崩 7 個檔案**。
   - 教訓：替換字串含 regex / shell metacharacter（`| ( ) / $`）時，永遠別用 batch sed。改用 Edit tool 的 `replace_all` 一檔一檔做——慢但安全。這已寫進「給未來自己的提醒」。
   - 對 prompt 的啟示：要求 agent 做批次文字替換時，明確指定「用 Edit 逐檔，不准 batch sed」，不要讓它自己選工具。

2. **「請依研究結果實作」這種 prompt 把理解外包出去**
   - 把「你研究完就修 bug」丟給 agent = 它的綜合判斷會很淺。有效的 prompt 是我自己先讀懂、給出檔案路徑 + 行號 + 具體要改什麼。
   - 教訓：**不要把『理解』delegate 出去**。delegate 的是執行，不是判斷。

3. **agent 說「完成了」≠ 真的能跑**
   - generator 回報 contract 全打勾、測試綠燈。第一次本地實跑連撞 3 個坑（schema 沒建 / k6 pipe / sed 自爆）。
   - 教訓：「lint 通過」「測試綠」只覆蓋受控環境。real-environment smoke 是分開的、必要的 gate。**驗收 agent 的產出時，看實際 diff / 實際執行，不信它的總結。**

4. **沙箱限制造成的「合理範圍切割」會留下驗證盲區**
   - Sprint C 故意把「腳本完備」跟「實際數字」切開（subagent 無法在 sandbox 可靠跑 docker），這切割本身合理，但**沒有任何後續環節補回『實跑驗證』**，盲區就一路漏到第一次本地跑。
   - 教訓：每次因為環境限制做 scope 切割，要顯式問「被切掉的那塊由誰、在哪個環節補驗」，不能讓它隱形消失。

---

## 6. 結尾（章節：Takeaways）

三條帶走的東西，網站結尾大字：

1. **把 AI 當健忘的初級工程師**——context 用檔案傳（spec / contract / ADR / CHANGELOG），不靠對話記憶；職責用 agent 邊界切（planner / generator / evaluator），不讓一個 agent 校驗自己。
2. **規則沒有 enforcement 點就不存在**——寫進 CLAUDE.md 的提醒會被漏讀，變成 CI 會紅的 test / agent workflow 跑不過的 gate 才真的擋住。
3. **人在迴圈裡的價值是判斷不是打字**——AI 很會把數據整理成漂亮結論，「這不對勁、重做」這種後設判斷必須人帶進來（呼應 Part 1 的 confound）。

---

## 附：可放上網站的素材清單

- `docs/incidents/2026-05-15-load-test-bootstrap.md` — 連環踩坑 postmortem（4 個 bug + Action Items + 給未來自己的提醒），第 4、5 節主要素材
- `~/.claude/agents/{planner,generator,evaluator}.md` — agent 職責定義（可節錄展示 context-passing 設計）
- `docs/CHANGELOG.md` — 每個 agent 自己追加的執行記錄（展示「決策外部化」實例）
- `docs/contracts/` + `docs/specs/` — contract / spec 範例（展示「傳檔案不傳記憶」）
- `tests/test_no_latest_images.py` — 把規則變 test 的具體例子（案例 A enforcement）
- `docs/decisions/000{1,3,4}-*.md` — ADR（跨 sprint 不丟失決策的載體）
