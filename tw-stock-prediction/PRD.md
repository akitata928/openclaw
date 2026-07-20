# PRD — 台股強勢股預測與回測自動化系統

> Product Requirements Document。本文件定義專案目標、範圍、需求與限制。
> 實作切分見 `MVP.md`，各 Phase 工作項見 `jobs.md`。
> 現況盤點依據：`akitata928/openclaw-workspace` → `repos/openclaw-jojo-dataops`
> （`scripts/tw_market_core.py` 2,450 行、`scripts/tw_stock_rankings.py`、
> `schema/*.sql`、`docs/architecture.md`、`docs/cron.md`）。

## 1. 一句話目標

在既有 dataops 資料管線（`tw_market.sqlite` + `tw_stock_rankings.sqlite`）之上，
建立一個**每日自動產出強勢股候選清單、並能以歷史資料嚴謹回測該選股策略績效**的新資料線（data line）。

## 2. 背景與現況（已盤點原始碼）

### 2.1 已有能力（不必重做）

**`tw_market_core.py` → `data/tw_market.sqlite`**

| 表 | 內容 | 對本專案的意義 |
|----|------|----------------|
| `securities` | 上市+上櫃證券主檔：`instrument_type`、`security_category`（普通股/各類 ETF/ETN）、產業、ISIN、`listed_date`/`delisted_date`/`is_active` | 股票池過濾現成；**含下市記錄，可防存活者偏差** |
| `daily_prices` | 日 OHLCV + `turnover` + `txn_count`（TWSE MI_INDEX / TPEX 日收盤） | 特徵計算主資料；可回填至 2010-01-04（`--backfill-*`，含 resume 游標） |
| `institutional_flows` | **三大法人買賣超**（TWSE T86 + TPEX 3insti），含回填 | 籌碼特徵已有，原規劃誤判為缺口 |
| `etf_navs` | ETF 淨值/市價/折溢價（MoneyDJ），0050 在列 | 0050 基準資料現成 |
| `fetch_runs` / `daily_update_runs` / `backfill_progress` | 抓取執行記錄、每日更新狀態機（ok/incomplete/skipped/error、attempt 標籤）、回填游標 | 自動化骨架已存在 |
| `data_quality_checks` / `raw_payloads` | 品質檢查記錄、原始 payload 留存（sha256） | 資料可信度機制已存在（`--verify`、`--audit-securities-master`、`--cross-check-rankings`） |

**`tw_stock_rankings.py` → `data/tw_stock_rankings.sqlite`**

- Yahoo 台股排行 API，成交量/成交值**各前 100 名**逐日快照（Telegram 只顯示前 20）。
- 已含排名動能欄位：`previous_rank`、`rank_delta_1d`、`entered_top20`、`captured_at_slot`。
- 跨庫查詢用 SQLite ATTACH（repo 既有模式）。

**自動化（OpenClaw cron，已在 Mac Mini 運轉）**

- 排行 14:00；市場核心資料次日 06:00 主跑 + 12:00 重試（週二–週六）；皆有 Telegram 回報。
- Phase 3 的「自動化」是**沿用此模式加一條 job**，不是從零建排程。

**可選的進階特徵來源（同 repo 其他資料線）**

- `tw_market_news.py`：Yahoo 股市新聞 RSS，含個股 symbol 連結。
- `tw_coverage_graph.py`：1,700+ 檔研究報告的供應鏈/題材知識圖譜。
- `tw_theme_heat.py`：題材熱度（排行動能 × 法人 × 新聞）聚合。

### 2.2 已確認缺口（本專案要補的）

| # | 缺口 | 說明 |
|---|------|------|
| G1 | **還原權息價** | `daily_prices` 為原始價；除息日會被誤判為下跌，報酬計算必須先解決。範圍含分割/併股/減資（0050 2025-06 分割案例證實）。**實作已交付**（`tw_corporate_actions.py`，workspace PR #4），待 Mac Mini 驗證 |
| G2 | 融資融券餘額 | 未入庫；訊號 v2 才需要 |
| G3 | 處置股/注意股/全額交割股清單 | 未入庫；股票池過濾先用近似規則 |
| G4 | 訊號 → 預測 → 回測閉環 | 完全不存在，本專案主體 |
| G5 | 加權報酬指數（TAIEX Total Return）歷史 | 未入庫；0050 可先當唯一基準 |

### 2.3 仍需在 Mac Mini 上驗證（雲端 repo 不含 DB）

主 SQLite 檔不進 Git（repo 政策），故以下只能在本機驗證：

- V1：`daily_prices` 實際回填深度與缺日率（回填**能力**到 2010，**實際進度**看 `backfill_progress`）。
- V2：`institutional_flows` 實際回填深度。
- V3：排行快照實際起始日與逐日連續性。

### 2.4 順手清理項

- `openclaw-workspace/scripts/tw_stock_rankings.py`（489 行）是舊完整副本，與
  `repos/openclaw-jojo-dataops/scripts/tw_stock_rankings.py`（556 行）不一致；
  `tw_market_core.py` 已用 20 行 shim 模式，rankings 建議比照，避免雙版本漂移。

## 3. 既有方案盤點（preflight）

原則：資料層沿用自有系統；回測與模型不重造輪子——但要尊重 repo 的既有工程模式。

**重要約束：dataops repo 全部腳本刻意只用 Python 標準庫**（urllib/sqlite3/zoneinfo，無 pandas）。
這讓 cron 路徑零依賴、可長期運轉。因此：

| 方案 | 定位 | 採用決策 |
|------|------|----------|
| 每日訊號路徑（cron 執行） | — | **維持 stdlib-only**，與既有資料線一致；動能/量比/z-score 用 SQL + 純 Python 足夠 |
| vectorbt / backtrader | 回測框架 | 回測屬**研究路徑**（手動、離線），允許獨立 venv 引入；Phase 2 先做 vectorbt 技術評估，不合用則自寫向量化引擎（pandas/numpy） |
| FinMind | 台股開源資料 API | **補缺口用**（G1 除權息、G2 融資券），優先於新寫爬蟲；官方 API 可得時仍以官方為準 |
| twstock | 台股抓取套件 | 不採用，被 `tw_market_core.py` 覆蓋 |
| FinLab 等付費平台 | 雲端策略平台 | 不採用（付費、黑盒、無法接自有 SQLite） |
| LightGBM | ML 排序 | Phase 4 採用（研究路徑，非 cron 路徑） |

結論：自建的只有「膠水」——特徵 SQL、台股交易規則、與既有 cron/Telegram 模式的整合。

## 4. 使用者與情境

單一使用者（個人投資研究）。三個核心情境：

1. **每日盤後**：cron 自動算訊號 → 產出強勢股候選 Top N（分數 + 理由欄位）→ Telegram 摘要。
2. **策略研究**：修改訊號/參數後，一鍵對歷史區間回測，得到績效報告並與基準比較。
3. **事後追蹤**：系統留存每日 point-in-time 預測快照，逐日對答案，累積 live 命中率。

## 5. 系統範圍

### 5.1 In scope

- **資料補缺**：還原權息價（G1）、加權報酬指數（G5）；後期融資券（G2）、處置股（G3）。
- **特徵/訊號層**：動能、量能、排行動能（rank_delta/entered_top20）、法人買賣超；規則評分 →（後期）ML 排序。
- **預測層**：每日 Top N 候選清單落地為新資料線 `data/tw_strong_signal.sqlite`（point-in-time 快照）。
- **回測層**：歷史模擬「每日照清單換股」，含台股真實交易成本與漲跌停限制。
- **自動化層**：沿用 OpenClaw cron + Telegram 回報模式新增一條 job。
- **追蹤層**：預測 vs 實際的滾動績效統計。

### 5.2 Out of scope（明確不做）

- 盤中即時行情、當沖、tick/分 K。（排行雖有盤中 slot，訊號只用收盤後資料）
- 自動下單（券商 API）。產出僅供研究參考。
- 期貨、選擇權、權證；ETF 僅作基準不入選股池（`security_category` 現成可濾）。
- 財報基本面深度模型。
- 多使用者、Web 服務化。

## 6. 功能需求

| ID | 需求 | 優先級 |
|----|------|--------|
| F1 | 還原權息價計算與入庫（G1），含驗證 | P0 |
| F2 | Point-in-time 取數介面：ATTACH 跨庫（market + rankings），`as_of` 強制 | P0 |
| F3 | 特徵計算：5/20/60 日報酬、量比、波動、排行動能、法人淨買超；規則評分 Top N 逐日落地 | P0 |
| F4 | 回測引擎:指定區間、持有天數、部位數、成本模型 | P0 |
| F5 | 績效報告：總報酬、CAGR、Sharpe、MDD、勝率、換手率、vs 0050（後加 TAIEX TR） | P0 |
| F6 | cron job + Telegram 日報（沿用既有模式與 cron 安全規範 `docs/cron.md`） | P1 |
| F7 | live 追蹤：每日預測對答案、滾動命中率報表 | P1 |
| F8 | 融資券、處置股入庫與訊號 v2 | P2 |
| F9 | ML 排序（LightGBM）+ walk-forward，與規則基線比較 | P2 |
| F10 | 參數掃描/敏感度分析 | P2 |

## 7. 非功能需求

- **Point-in-time 正確性（最高優先）**：日期 D 的訊號只能用 D 收盤（含）以前可得的資料。
  回測與每日 live 產出必須走同一條程式路徑，杜絕前視偏差。
  注意法人資料公布時間（盤後約 15:00–17:00）：訊號若用 D 日法人資料，實際可交易時點為 D+1，回測需一致。
- **可重現**：同一資料庫 + 同一參數 → 相同回測結果；ML 固定 seed。
- **存活者偏差**：股票池以 `securities.is_active` + `listed_date`/`delisted_date` 判「當日存在」；不得用今日清單回測過去。
- **官方 API 禮貌性**：沿用既有 `--sleep-seconds`（預設 1.0s）與 retry 模式；回填大量歷史時提高間隔。
- **效能**：cron 路徑（stdlib）全市場單日特徵 < 5 分鐘；研究路徑 5 年回測 < 10 分鐘。
- **repo 慣例**：新資料線遵循 dataops「Data Line Pattern」——script + schema + export +
  verification + cron note + recovery note（`docs/architecture.md`）；主 SQLite 不進 Git。

## 8. 台股領域規則（回測正確性的硬約束）

1. **交易成本**：手續費 0.1425%（買賣各一次，可設折扣），證交稅 0.3%（賣出）。
   來回約 0.585%（無折扣）——高換手策略會被成本吃光，必須如實計入。
2. **漲跌幅 ±10%**：訊號日收漲停者隔日常買不到；回測支援「漲停不買/跌停不賣」開關。
3. **成交價假設**：訊號用 D 日收盤資料，成交用 D+1 開盤價；不可同日收盤成交。
4. **流動性過濾**：剔除日均成交值 < 5,000 萬（`daily_prices.turnover` 現成）；處置股先以「連續漲停」近似（G3 補齊後改真實清單）。
5. **除權息**：報酬一律用還原價（G1 是 P0 的原因）。
6. **T+2 交割**：不模擬資金交割限制，文件註明。
7. **基準**：MVP 用 0050 buy-and-hold（資料現成）；G5 補齊後加 TAIEX Total Return。

## 9. 成功指標

- **系統面**：連續 10 個交易日 cron 全自動產出清單 + Telegram 日報，零人工介入；
  回測與 live 路徑對同一日輸出相同清單。
- **研究面**（不承諾獲利，承諾「可信」）：報告能明確回答
  「扣除成本後是否勝過基準、回撤多大、對參數是否敏感」。
- **反指標**：前視偏差或漏計成本被抓到 = P0 缺陷。

## 10. 風險

| 風險 | 影響 | 對策 |
|------|------|------|
| Mac Mini 上實際回填深度不足（§2.3） | 回測區間縮短 | Phase 0 先驗證，不足則用既有 backfill（含 resume）補 |
| Yahoo 排行 API 非官方、可能改版 | 排行特徵中斷 | 特徵設計讓排行動能可降級（缺時僅用價量）；官方成交值排名可從 daily_prices 自算 |
| 還原價算錯 | 全部報酬失真 | 與 FinMind/公開還原序列抽樣比對（MVP 驗收 D2） |
| 過擬合 | 回測漂亮、live 賠錢 | walk-forward、參數敏感度、live 追蹤對照 |
| 前視偏差 | 全部結論作廢 | 訊號/回測共用同一 point-in-time 介面 + 專門測試 |
| 個人時間碎片化 | 爛尾 | Phase 切小、每 Phase 有獨立可用產出（jobs.md） |

## 11. 名詞

- **強勢股**：操作型定義＝依訊號分數排名前 N、預期未來 H 日相對基準有超額報酬的個股；
  N、H 為參數（MVP 預設 N=10、H=5）。
- **Data line**：dataops repo 的單位——一條「來源 → 腳本 → schema → 驗證 → cron → 回復」完整鏈。
- **Point-in-time**：只用當時已知資訊。
- **Walk-forward**：滾動訓練/驗證切分，模擬真實逐期重訓。
