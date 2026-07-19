# MVP — 最小可行版本定義

> 上游文件：`RPD.md`（需求與現況盤點）。工作項拆解：`jobs.md`。
> MVP 對應 jobs.md 的 Phase 0–2；Phase 3 之後為 MVP 後迭代。

## 1. MVP 一句話

**一條可信的閉環**：規則式訊號每日選出 Top 10 強勢股候選 → 同一套邏輯跑多年回測
→ 產出含成本、vs 0050 的績效報告。cron 整合留到 Phase 3，MVP 允許手動執行。

判斷 MVP 成功的問題只有一個：「這個訊號扣完成本，歷史上到底行不行？」——
MVP 完成時必須能用數字回答。

## 2. 範圍

### In（MVP 必做）

1. **資料驗證與補缺**（既有能力多，工作比原估少）：
   - 在 Mac Mini 驗證 `daily_prices` / `institutional_flows` 實際深度（RPD §2.3），
     不足 3 年用既有 `--backfill-*`（含 resume）補。
   - **還原權息價**（RPD G1）：新增除權息事件入庫 + 還原係數計算。這是 MVP 最大的新資料工作。
2. **Point-in-time 取數介面**：ATTACH `tw_market.sqlite` + `tw_stock_rankings.sqlite`，
   `as_of` 參數強制；訊號與回測唯一取數入口。
3. **規則式訊號 v1**（刻意簡單，當基線；特徵比原規劃多是因為法人與排行動能資料現成）：
   - 股票池：`security_category = 'common_stock'` 且當日 active（下市判斷用主檔欄位），
     日均成交值 ≥ 5,000 萬。
   - 特徵：20 日/5 日還原報酬、量比（5 日均量/60 日均量）、
     排行動能（`entered_top20`、`rank_delta_1d`）、法人 5 日累計淨買超佔成交量比。
   - 評分：特徵 z-score 加權合成，取 Top 10。權重集中一個 config。
4. **回測引擎**（研究路徑，允許 pandas/numpy；優先評估 vectorbt，不合用則自寫向量化）：
   - 每日等權重持有 Top 10，持有 H=5 日換股（參數可調）。
   - D+1 開盤價成交、漲停不追買、手續費 0.1425%×2 + 證交稅 0.3%。
5. **績效報告**（Markdown/CSV）：總報酬、CAGR、Sharpe、MDD、勝率、平均持有報酬、
   換手率、逐年分拆，vs 0050 buy-and-hold（資料現成）。
6. **每日產出腳本**（cron 路徑，stdlib-only）：手動執行 → 落地當日 Top 10 快照
   （股票、分數、各特徵值、訊號版本號）到 `data/tw_strong_signal.sqlite`。
7. **前視偏差測試**：自動化測試證明「回測在日期 D 的選股 == 只餵 D 以前資料的 live 選股」。

### Out（MVP 明確不做，屬後續 Phase）

- cron job 註冊與 Telegram 日報（Phase 3；模式已存在，接上即可）。
- 融資融券、真實處置股清單（Phase 3 → 訊號 v2）。
- TAIEX Total Return 基準（Phase 3 補；MVP 只用 0050）。
- ML、walk-forward（Phase 4）。
- 新聞/題材熱度/供應鏈圖譜特徵（Phase 4+ 候選）。
- 參數掃描圖表、滑價模型（Phase 5）。
- 任何下單、盤中訊號。

## 3. 架構（MVP）

遵循 dataops repo 的 Data Line Pattern（`docs/architecture.md`），落在
`repos/openclaw-jojo-dataops`：

```
repos/openclaw-jojo-dataops/
├── schema/
│   └── tw_strong_signal.sql        # dividends、adjust_factors、features_cache、
│                                   # predictions、signal_runs
├── scripts/
│   ├── tw_corporate_actions.py     # 除權息事件抓取 + 還原係數（cron 路徑, stdlib）
│   ├── tw_strong_signal.py         # 每日：point-in-time 取數 → 特徵 → 評分 →
│   │                               # Top10 快照 + 驗證 + Telegram-ready 摘要（stdlib）
│   └── tw_backtest.py              # 研究：區間回測 → 報告（允許 pandas/vectorbt，
│                                   # 獨立 venv；cron 不跑此路徑）
└── docs/                           # data line 註記：來源/驗證/cron/回復
```

- **雙路徑原則**：cron 路徑維持 repo 的 stdlib-only 慣例（零依賴、長期穩定）；
  回測研究路徑手動離線執行，允許引入 pandas/numpy/vectorbt（J2.1 評估後定案）。
- **訊號邏輯共用**：特徵與評分實作在 `tw_strong_signal.py` 內為純函式，
  `tw_backtest.py` import 同一份；這是防前視偏差的結構性保證，不是靠小心。
- 儲存：新資料線一顆 `data/tw_strong_signal.sqlite`；跨庫用 ATTACH（repo 既有模式）；
  主 SQLite 不進 Git，schema/exports 進 Git。
- `tw_market_core.py` / `tw_stock_rankings.py` 不改動（發現缺欄位才小幅擴充）。

## 4. 資料流

```
每日盤後（MVP 手動；Phase 3 接 cron）：
  tw_market_core --daily-update（既有 06:00/12:00 cron 已在跑）
  tw_stock_rankings（既有 14:00 cron 已在跑）
  tw_corporate_actions ──► 還原係數更新
        ──► tw_strong_signal：ATTACH 取數（as_of=今日）──► 特徵 ──► 評分
        ──► Top10 快照落地 predictions 表 ──► 摘要輸出

研究回測：
  tw_backtest：同一 point-in-time 介面取歷史區間
        ──► 同一份特徵/評分函式 ──► 回測引擎（成本/漲跌停）──► 報告（vs 0050）
```

## 5. 驗收標準（Definition of Done）

| # | 驗收項 | 通過條件 |
|---|--------|----------|
| D1 | 資料深度 | 近 3 年上市+上櫃 `daily_prices` 缺日率 < 0.5%（用官方交易日曆對），抽 10 檔與官方收盤價全對 |
| D2 | 還原價正確 | 抽 5 檔有除息個股（含 1 檔高股息），還原報酬與公開還原序列誤差 < 0.1% |
| D3 | 前視測試 | 任取 20 個歷史日期，回測選股 == 截斷資料後 live 選股，全數相等 |
| D4 | 成本模型 | 單元測試覆蓋：買賣成本、賣出稅、漲停不成交情境 |
| D5 | 回測報告 | 一鍵產出 ≥ 3 年報告，含第 2 節第 5 點全部指標與 0050 對照 |
| D6 | 每日快照 | 連續 5 個交易日手動執行成功，predictions 可查歷史、含特徵值與版本號 |
| D7 | 可重現 | 同參數重跑回測，數字完全一致 |

## 6. 刻意的簡化（記錄在案）

- 等權重、無停損——MVP 只驗證「選股訊號有無 alpha」，資金管理後移。
- 「D+1 開盤全額成交」偏樂觀（未計滑價），報告標注；滑價模型 Phase 5。
- 處置股/全額交割股以「連續漲停剔除」近似；Phase 3 入庫真實清單後替換。
- 法人特徵時點保守處理：D 日訊號僅用 D-1（含）以前的法人資料，
  避免「法人資料尚未公布就被用來選股」的隱性前視（實務公布在盤後，D 日 16:00 後跑則可用 D 日；
  MVP 先取保守版，Phase 3 依 cron 實際執行時間放寬並測試）。
- T+2 交割資金限制不模擬。
- 基準僅 0050；TAIEX TR Phase 3 補。

## 7. MVP 後的下一步（摘要，詳見 jobs.md）

Phase 3 自動化整合（OpenClaw cron + Telegram 日報 + 融資券/處置股/TAIEX TR 入庫 + 訊號 v2）
→ Phase 4 ML 排序（LightGBM + walk-forward；候選特徵加新聞熱度/題材圖譜）
→ Phase 5 監控迭代（live 追蹤、敏感度、滑價）。
