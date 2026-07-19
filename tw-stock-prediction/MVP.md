# MVP — 最小可行版本定義

> 上游文件：`RPD.md`（需求與限制）。工作項拆解：`jobs.md`。
> MVP 對應 jobs.md 的 Phase 0–2；Phase 3 之後為 MVP 後迭代。

## 1. MVP 一句話

**一條可信的閉環**：規則式訊號每日選出 Top 10 強勢股候選 → 同一套邏輯跑 3 年回測
→ 產出含成本、含基準比較的績效報告。不含 ML、不含通知、允許手動執行。

判斷 MVP 成功的問題只有一個：「這個排行策略，歷史上扣完成本到底行不行？」——
MVP 完成時必須能用數字回答。

## 2. 範圍

### In（MVP 必做）

1. **資料整備**：確認/補齊 `tw_market_core.py` 資料庫的 OHLCV 歷史（目標 ≥ 3 年）、
   還原權息價、每日 Top20 排行歷史留存。
2. **Point-in-time 資料介面**：`get_market_data(as_of_date)` 一類的唯一取數入口，
   訊號與回測都只能經過它拿資料。
3. **規則式訊號 v1**（刻意簡單，當基線）：
   - 股票池：上市+上櫃普通股，剔除日均成交值 < 5,000 萬。
   - 特徵：20 日報酬、5 日報酬、量比（5 日均量/60 日均量）、是否進入成交值 Top20。
   - 評分：特徵標準化後加權合成，取 Top 10。
4. **回測引擎**（優先評估 vectorbt，不合用則自寫向量化版本）：
   - 每日等權重持有 Top 10，持有 H=5 日換股（參數可調）。
   - 次日開盤價成交、漲停不追買、手續費 0.1425%×2 + 證交稅 0.3%。
5. **績效報告**（Markdown/CSV 輸出）：
   總報酬、CAGR、Sharpe、最大回撤、勝率、平均持有報酬、換手率、逐年分拆，
   vs 0050 buy-and-hold。
6. **每日產出腳本**：手動執行 `daily_run.py` → 更新資料 + 落地當日 Top 10 快照（含分數與特徵值）。
7. **前視偏差測試**：自動化測試證明「回測在日期 D 的選股 == 只餵 D 以前資料的 live 選股」。

### Out（MVP 明確不做，屬後續 Phase）

- cron 自動排程、失敗告警、訊息通知（Phase 3）。
- 三大法人、融資融券特徵（Phase 3 資料擴充後進訊號 v2）。
- ML 模型、walk-forward（Phase 4）。
- 參數掃描 UI/圖表 dashboard（Phase 5）。
- 任何下單、盤中資料。

## 3. 架構（MVP）

```
tw-stock-prediction/
├── RPD.md / MVP.md / jobs.md      # 規劃文件（本組文件）
└── src/
    ├── data/
    │   ├── core_db.py             # 包 tw_market_core.py 的 DB 為唯一取數介面（point-in-time）
    │   └── adjustments.py         # 還原權息價
    ├── features/
    │   └── momentum_volume.py     # 特徵計算（純函式：DataFrame in/out）
    ├── signal/
    │   └── rule_v1.py             # 評分與 Top N（回測與 live 共用）
    ├── backtest/
    │   ├── engine.py              # vectorbt 封裝或自寫向量化引擎
    │   ├── costs.py               # 台股成本/漲跌停規則
    │   └── report.py              # 績效指標與報告輸出
    ├── daily_run.py               # 每日：更新資料 → 產出快照到 predictions 表
    └── backtest_run.py            # 研究：指定區間回測 → 報告
```

- 語言：Python 3.11+；核心依賴 pandas、numpy、（候選）vectorbt、pytest。
- 儲存：沿用既有 SQLite；新增 `predictions`（每日快照）與 `features` 快取表。
- `tw_market_core.py` 不重寫，以 adapter（`core_db.py`）包起來；發現缺欄位才回頭小幅擴充它。

## 4. 資料流

```
每日盤後（手動）：
  tw_market_core 抓取更新 ──► core_db（point-in-time 介面）
        ──► features ──► rule_v1 評分 ──► Top10 快照落地 predictions 表

研究回測：
  core_db 歷史區間 ──► features ──► rule_v1（同一份程式碼）
        ──► backtest.engine（成本/漲跌停）──► report（vs 0050）
```

關鍵設計：**features 與 rule_v1 是純函式，被 daily 與 backtest 兩邊呼叫同一份**。
這是防前視偏差的結構性保證，不是靠小心。

## 5. 驗收標準（Definition of Done）

| # | 驗收項 | 通過條件 |
|---|--------|----------|
| D1 | 資料完整性 | 近 3 年上市+上櫃 OHLCV 缺日率 < 0.5%，抽 10 檔與官方網站收盤價全對 |
| D2 | 還原價正確 | 抽 5 檔有除息個股，還原報酬與公開來源誤差 < 0.1% |
| D3 | 前視測試 | pytest：任取 20 個歷史日期，回測選股 == 截斷資料後 live 選股，全數相等 |
| D4 | 成本模型 | 單元測試覆蓋：買賣成本、賣出稅、漲停不成交情境 |
| D5 | 回測報告 | 一鍵產出 3 年報告，含第 2 節第 5 點全部指標與 0050 對照 |
| D6 | 每日快照 | 連續 5 個交易日手動執行 `daily_run.py` 成功，predictions 表可查歷史 |
| D7 | 可重現 | 同參數重跑回測，報告數字完全一致 |

## 6. 刻意的簡化（記錄在案）

- 等權重、無風控停損——MVP 只驗證「選股訊號有無 alpha」，資金管理後移。
- 成交假設「次日開盤全額成交」偏樂觀（未計滑價），報告需標注；滑價模型 Phase 5。
- 處置股/全額交割股清單若抓取困難，MVP 先以「連續漲停剔除」近似，Phase 3 補真實清單。
- T+2 交割資金限制不模擬。

## 7. MVP 後的下一步（摘要，詳見 jobs.md）

Phase 3 自動化（cron + 告警 + 法人資料）→ Phase 4 ML 排序（LightGBM + walk-forward）
→ Phase 5 監控迭代（live 追蹤、參數敏感度、滑價模型）。
