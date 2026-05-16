# tech-blog — 學習歷程紀錄站

單頁靜態網站，記錄兩件事：

- **Part 1**：URL shortener / QR code 服務在高 QPS 下三種 click 計數架構的 k6 壓測比較，以及一個觀察者效應 confound 如何讓結論反轉。
- **Part 2**：用 Claude Code 多 agent harness 開發此專案的踩坑與「規則要有 enforcement 點」的心得。

## 結構

| 檔案 | 說明 |
| --- | --- |
| `index.html` | 首頁（hero + 關於我 + 兩篇文章入口，Tailwind CDN，無 build step） |
| `part1.html` | Part 1 · 壓測實驗（含 Chart.js 圖表） |
| `part2.html` | Part 2 · AI harness 心得 |
| `.nojekyll` | 關閉 GitHub Pages 的 Jekyll 處理 |

## 本地預覽

```sh
cd tech-blog
python3 -m http.server 8000
# 開啟 http://localhost:8000/
```

## 部署到 GitHub Pages（project repo）

1. 建一個 GitHub repo（例如 `tech-blog`），把本目錄內容 push 到 `main` branch 根目錄。
2. repo → **Settings** → **Pages**。
3. **Source** 選 `Deploy from a branch`，branch 選 `main`、folder 選 `/ (root)`，按 **Save**。
4. 等一兩分鐘，網站上線於 `https://<username>.github.io/<repo>/`。

> 全部資源走 CDN、頁內以 hash 錨點導覽，無本地 asset 路徑，因此 project repo 的子路徑（`/<repo>/`）不影響顯示，毋須額外 base path 設定。
