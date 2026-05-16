---
title: "Part 1 · 壓測實驗：高 QPS 下的 click 計數架構"
date: 2026-05-10
weight: 1
tags: ["壓測", "k6", "架構", "實驗設計"]
summary: "三種 click 計數架構的 k6 壓測比較。核心是一個觀察者效應 confound 如何讓結論從「−8%」反轉成「+37%」，以及我怎麼發現並修正它。"
ShowToc: true
TocOpen: false
---

用 k6 實測 URL shortener 三種 click 計數架構在高 QPS 下的承載上限——以及一個觀察者效應如何讓我的結論完全反轉。

> 重點不是我做了一個 QR code 服務，是我用數據驗證架構決策時，犯了一個經典的實驗設計錯誤、發現它、修正它，結論從「拆服務沒用」翻盤成「拆服務 +37%」。

## The Question — 專案目的 {#the-question}

我想回答一個具體問題：

> 在高 QPS 下，redirect 路徑的 click 計數要怎麼設計，承載上限差多少？

三個候選架構，用 k6 打到飽和，看每個架構的 RPS 天花板與 p99：

| 架構 | redirect handler 做什麼 | click 計數在哪算 |
| --- | --- | --- |
| **A. 同步 inline** | redirect 內直接 `HINCRBY` Redis | 同一個 request 同步等它完成 |
| **B. 異步 worker** | redirect 只 `XADD` 到 Redis Stream | 獨立 worker process 消費、聚合 |
| **C. 完全移除** | 不計數 | 不存在 |

## The Setup — 壓測工具與方法 {#the-setup}

- **負載工具**：[k6](https://k6.io/) — 在被測服務**外面**產生流量，k6 自己量 RPS / p50 / p95 / p99 / error rate。**這是唯一的量測來源。**
- **被測服務**：FastAPI + Redis（URL cache + PNG cache）+ Postgres，single uvicorn process，跑在 Docker（Apple M4 Pro / 8 vCPU 配額）。

三個負載情境：

1. **redirect_hot** — 1,000 個預建 token 隨機打 `/r/{token}`，全部 Redis cache hit。量 redirect 路徑吞吐天花板。負載曲線：30s warm-up → 2m @ 200 VU → 30s spike @ 500 VU → 30s ramp-down。
2. **redirect_cold** — 每個 iteration 先 `POST` 建新 token 再 redirect。量 DB 寫入路徑。constant 50 VU × 3 min。
3. **image_mixed** — 50% cache hit / 50% miss 打 `/v1/qr_code_image`。量 CPU-bound PNG 生成。100 VU × 2 min。

{{< callout type="neutral" title="公平性控制" >}}
每輪測試前 `FLUSHDB` + `TRUNCATE` + 重 seed，三個架構版本的 logging 條件一致（structlog-only，開銷 ~100µs/req，可忽略）。
{{< /callout >}}

## The Journey — 架構演進時間軸 {#the-journey}

1. **Prototype** — 單體 FastAPI，redirect 同步 `HINCRBY` 計數。能跑，沒測過極限。
2. **拆 worker** — 把 click 計數拆成 producer（`XADD`）+ 獨立 consumer worker，用 Redis Streams、consumer group、`XPENDING`/`XCLAIM` crash recovery、dedupe key。
3. **加觀測性** — 見下方紅框，這步是錯的。
4. **壓測** — 用 k6 跑三情境，量 baseline vs 各架構。
5. **發現 confound** — 數據說「拆 worker 反而變慢」。不對勁。深究發現是觀測性 stack 的鍋。
6. **scope 收斂** — 移除 click counting（ADR-0003）、移除 OTel + Prometheus（ADR-0004），只留 structlog。
7. **重測（乾淨版）** — 三架構觀測性條件拉平，只變 click 架構。**結論反轉。**

{{< callout type="bad" title="步驟 3：加觀測性 ← 這步是錯的" >}}
為了「能觀測微服務」加了 OpenTelemetry tracing + Prometheus metrics + Grafana 全套。**這步污染了實驗，見下一節。**
{{< /callout >}}

## 我把實驗搞砸了（全站重點） {#the-mistake}

{{< callout type="warn" title="★ 全站重點" >}}
**犯的錯**：為了「能觀測微服務」，我把 **OpenTelemetry + Prometheus + Grafana** 整套裝進**被測程式本身**。

這是典型的**觀察者效應**：你為了量一個東西，而你的量測手段改變了被測對象。OTel SDK 在每個 request 建 span、每個 Redis op 包 instrumentation，每 request 多 ~5–10ms。
{{< /callout >}}

**後果：結論完全錯誤。** 第一輪對比：

| | 同步 inline（無觀測）| 異步 worker（**揹了全套 OTel**）|
| --- | --- | --- |
| hot RPS | 2,060 | 1,901 ⚠ 被污染的錯誤數據 |

數據說「拆 worker 慢了 8%」。我差點據此寫下「worker 拆分對 throughput 沒幫助」的結論。

**為什麼差點沒抓到** — 兩個變因同時動了：

- click 架構（同步 → 異步 worker）
- 觀測性 stack（無 → 全套 OTel）

異步版的架構優勢被它自己揹的 OTel overhead 蓋掉，淨效果看起來是「變慢」。**歸因錯誤。**

**修正**：重新設計實驗，**觀測性固定成 structlog-only**（最小、一致），只讓 click 架構這一個變因變動。

關鍵教訓：

1. **不要 instrument 被測對象本身** — 壓測量測應該在外部（k6 自己就會量），server 內部加 instrumentation = 改變被測物。
2. **一次只變一個變因** — A/B 對比的鐵律。我同時變了兩個，data 就說謊。
3. **數據反直覺時先懷疑實驗設計** — 「拆 worker 變慢」反直覺，該深究而不是寫進報告。

> → 從 AI 協作的角度看這同一件事：[Part 2 §3「AI 差點把錯誤結論寫進報告」]({{< relref "posts/part2-ai-harness.md#the-save" >}})

## The Real Numbers — 乾淨的結果 {#real-numbers}

觀測性拉平（全部 structlog-only）後，k6 量測，**這是唯一有效的數據**：

{{< perfcharts >}}

| Scenario | A. 同步 inline HINCRBY | B. 異步 Streams + worker | C. 完全移除 |
| --- | --- | --- | --- |
| **redirect_hot RPS** | 1,620 | **2,227 (+37%)** | **2,603 (+61%)** |
| redirect_hot p50 | 77.8 ms | 58.4 ms | 50.9 ms |
| redirect_hot p95 | 279.8 ms | 204.9 ms | 167.1 ms |
| **redirect_hot p99** | 329 ms | **234 ms (−29%)** | **190 ms (−42%)** |
| **redirect_cold RPS** | 1,121 | 1,301 (+16%) | 1,416 (+26%) |
| redirect_cold p99 | 202 ms | 152 ms | 77 ms (−62%) |
| **image_mixed RPS** | 458 | 486 | 458 |
| image_mixed p99 | 251 ms | 240 ms | 263 ms |
| error rate（全部）| 0% | 0% | 0% |

**結論：**

- **異步 worker 拆分讓 hot path 承載 +37%（1,620 → 2,227 RPS）、p99 −29%**。confound 修正後，跟第一輪「−8%」完全相反。
- 完全移除 click 計數再 +17%（少一個 Redis op）。
- **image 三架構持平**——因為瓶頸是 CPU-bound PNG 生成，與 click 架構無關。

{{< callout type="warn" title="誠實註記 — 數據邊界的 caveat" >}}
A 的 endpoint 是 sync `def`（FastAPI 丟 threadpool，~40 thread 上限），B/C 是 `async def`（event loop）。所以 A→B 的差距包含「sync→async endpoint」+「同步→worker offload」兩個因素一起——這是這兩個架構**實際存在時的真實樣子**，是有效的真實對比，但**不是單一變因隔離**。我知道自己數據的邊界在哪。
{{< /callout >}}

## Where It Breaks — 各情境瓶頸定位 {#where-it-breaks}

| 情境 | 瓶頸 | 怎麼確認的 | 突破方向 |
| --- | --- | --- | --- |
| redirect_hot | 單 process uvicorn 的 event loop | spike 500 VU 時 throughput 不升反降（past-peak）| 多 uvicorn worker / 多 instance |
| redirect_cold | SQLAlchemy connection pool（飽和在 pool_size+overflow=3）| pool 全程滿、但 Postgres CPU 閒置（queue 在 app 層）| 調大 pool / 升 DB tier |
| image_mixed | CPU-bound `qrcode` PNG 生成（~10–20ms/req）| api container CPU 接近 100%、與 click 架構無關 | 預生成常見 spec / Rust QR lib / offload thread pool |

## Scaling Reality — 要 50,000 RPS 怎麼辦 {#scaling}

單機天花板 ~2,000 RPS。要 50k：

- 不是單機調參能解決——是 **CDN offload + 水平擴展**問題。
- 95%+ 的 redirect 是 cache-hot 固定 URL → 放 Cloud CDN，origin 只處理 cache miss。
- Cloud Run autoscaling 拉 instance、Memorystore 升 Cluster、Postgres 升 tier。

**結論：50k RPS 是架構問題不是 tuning 問題。**

## Decisions / ADRs — 工程判斷紀錄 {#adrs}

每個「加入」與「移除」都有 ADR。主軸：**knows when NOT to add complexity**——展現的是減法的成熟度，不是加法的能力。

- **ADR-0001**：選 Redis Streams（而非 Pub/Sub / Kafka）做 click pipeline——已有 Redis、consumer group 夠用。
- **ADR-0003**：移除 click counting——對 redirect MVP 是 pure cost。背後是一個**讀寫職責分離（CQRS-style tiering）**的設計判斷：redirect 是讀密集路徑（Redis cache + Postgres），click/analytics 是寫密集路徑，兩者不該共用同一個 store；分析資料應該導向獨立 analytics tier（如 BigQuery）讓 redirect tier 不被寫入污染。
- **ADR-0004**：移除 OTel + Prometheus，保留 structlog——量出 ~5–10% overhead 對 single-process 不值得，Cloud Run console 已 cover。

{{< callout type="warn" title="⚠ ADR-0003 重要標注" >}}
這個 analytics tier 只是**架構決策，未實作、未壓測**。本實驗測的是 click 計數機制本身（同步 / 異步 worker / 移除）的承載差異，**不是讀寫職責分離拓樸的效能**。
{{< /callout >}}

## Takeaways {#takeaways}

{{< callout type="accent" big="true" >}}
1 · 壓測量測要在被測對象外部——instrument 被測物 = 觀察者效應。
{{< /callout >}}

{{< callout type="accent" big="true" >}}
2 · 一次只變一個變因——否則歸因錯誤，數據會說謊。
{{< /callout >}}

{{< callout type="accent" big="true" >}}
3 · 減法是工程成熟度——我移除的東西（click pipeline、OTel stack）跟我加的一樣重要，而且都有 ADR 紀錄理由。
{{< /callout >}}

---

下一篇 → [Part 2 · 把 AI 當聰明但健忘的初級工程師來管理]({{< relref "posts/part2-ai-harness.md" >}})
