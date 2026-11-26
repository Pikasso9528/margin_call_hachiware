# 融資維持率監測（同步 ＋ 自動接手 ＋ 歷史保存）

每個交易日盤後更新台股上市、上櫃個股與大盤的推估融資／融券維持率，並把每天的結果永久保存，網頁上可以看大盤和個股的歷史走勢。

全部跑在 GitHub 雲端：Actions 定時執行 → 結果提交回 repo → GitHub Pages 顯示網頁。不用自己的電腦、不用伺服器。

## 運作方式

1. **平常：同步 [msbk-tw/margin-monitor](https://github.com/msbk-tw/margin-monitor)**
   讀取他公開的成本紀錄 `state/cost_state.json`，直接沿用。數字和他的網站完全一致，也不會對證交所發請求。
2. **他沒更新時：自動接手**
   從手上最後一份成本紀錄（他前一天的）接著抓證交所、櫃買資料自己算。成本是接續的，所以不需要暖機。
   他連續停更多天也會逐日補上；他恢復後會自動改回同步。
3. **他的格式變了、repo 不見了**：同樣改成自己算，Actions 會顯示黃色警告。

網頁頂端會顯示當天資料來源（同步／自行推估），走勢圖上自行推估的日子用空心點標示。

## 雲端架設 SOP（只用瀏覽器）

1. **解壓縮** `margin-monitor.zip`。
2. **建立 repo**：GitHub 右上角「+」→ New repository，名稱例如 `margin-monitor`，選 **Public** → Create repository。
3. **上傳檔案**：Add file → Upload files，把資料夾**裡面**的 `docs`、`scripts`、`state`、`tests`、`README.md`、`requirements.txt` 拖進去 → Commit changes。
   - `.github` 是隱藏資料夾，通常傳不上去。請改用 Add file → **Create new file**，檔名輸入 `.github/workflows/update.yml`，用記事本打開 zip 裡同名檔案、全部複製貼上 → Commit changes。
4. **開寫入權限**：Settings → Actions → General → 最下方 Workflow permissions 選 **Read and write permissions** → Save。
5. **開網頁**：Settings → Pages → Source 選 Deploy from a branch，Branch 選 `main`、資料夾 `/docs` → Save。
6. **跑第一次**：Actions 分頁（出現提示就按啟用）→ 左邊「每日更新融資維持率」→ Run workflow → 欄位**留空**直接按綠色按鈕。約 1–2 分鐘完成。
7. **打開** `https://你的帳號.github.io/margin-monitor/`（第一次部署可能要再等 1–2 分鐘）。

之後不用做任何事。排程（台北時間）：
- 平日 23:05：同步；他還沒更新就先自己算
- 隔天 00:35：再同步一次，他補上了就改用他的
- 隔天 08:05：最後確認

GitHub 的排程常延遲 10–30 分鐘，屬正常現象。

## 疑難排解

| 狀況 | 處理 |
|---|---|
| Actions 紅色叉叉、push 403 | 確認 SOP 第 4 步有存檔 |
| 黃色警告「無法使用 msbk 的成本紀錄」 | 他那邊格式變了或 repo 不在了，網站仍會自己算；若持續很多天，把警告內容貼給我調整 |
| 黃色警告「自行計算失敗」 | 證交所暫時連不上，下一次排程會再試 |
| 紀錄裡印出「欄位對不上」 | 交易所改版，把那段紀錄複製下來修正 `scripts/fetch.py` |
| 收到 GitHub 信說排程被停用 | 到 Actions 頁面按 Enable workflow |
| 某幾天歷史缺漏（例如排程整晚沒跑） | 在自己電腦執行下面的匯入指令補回 |

## 補匯入 msbk 的過去資料（選用）

他每天覆蓋 `latest.json`，但每次都有 commit，舊資料都在 git 歷史裡：

```bash
git clone https://github.com/msbk-tw/margin-monitor.git /tmp/msbk
python scripts/import_git_history.py /tmp/msbk
```

自己算過的日期不會被覆蓋。這個壓縮檔已經包含 2026/11/18–11/26 的歷史。

## 檔案說明

| 路徑 | 用途 |
|---|---|
| `scripts/upstream.py` | 讀取、檢查並轉換 msbk 的成本紀錄 |
| `scripts/fetch.py` | 抓證交所／櫃買的融資餘額、收盤行情、發行股數（自行計算時才用） |
| `scripts/calc.py` | 成本推估、維持率公式、每日快照 |
| `scripts/history.py` | 歷史保存與個股序列 |
| `scripts/update.py` | 主程式：同步優先，必要時自算 |
| `scripts/import_git_history.py` | 從 msbk 的 git 歷史補匯入 |
| `state/stocks.json` | 目前的成本紀錄（同步或自算的最後一天） |
| `docs/data/latest.json` | 最新一天的完整資料 |
| `docs/data/history/daily/YYYYMMDD.json` | 每天一份完整快照，永久保存（約 120 KB／天） |
| `docs/data/history/market.json` | 大盤每日維持率、追繳／接近檔數、資料來源 |
| `docs/data/history/series.json` | 個股最近 120 個交易日走勢（網頁用） |

## 計算方式（與 msbk 相同）

- 融資成本：每天新增張數以當日成交均價計入，減少張數按平均成本扣除。
- 融資維持率 ≈ 收盤價 ÷（融資成本 × 成數），上市 60%、上櫃 50%。
- 融券維持率 ≈ 融券成本 × 1.9 ÷ 收盤價。
- 整體取兩者較低；< 130% 標「追繳」，130–140% 標「接近」。
- 參考價與前日收盤差超過 20%／25% 視為分割或減資，成本同比例調整。
- 大盤維持率 = Σ(融資餘額 × 收盤) ÷ Σ(融資餘額 × 成本 × 成數)。

以 2026/11/26 實測，同步後算出的大盤與 2,065 檔個股維持率、市值，和 msbk 網站完全相同。
這些都是推估值，不是任何帳戶的實際數字，通常低於券商的整戶擔保維持率。

## 注意

- msbk 的 repo 沒有放授權條款。自己使用沒問題；若網站要公開給別人看，建議先到他的 Issues 打聲招呼，頁尾已註明資料來源。
- 自行計算時，程式每次請求間隔 3.5 秒以上，避免被證交所封鎖，請勿調快。
- 想完全不依賴 msbk：Run workflow 時填 `60`，會自己從頭回補（約 30–40 分鐘）；之後若要停止同步，把 `update.yml` 裡的指令加上 `--self-only`。

## 本機測試（選用）

```bash
pip install -r requirements.txt
python tests/test_calc.py
python scripts/update.py
cd docs && python -m http.server   # 開 http://localhost:8000
```
