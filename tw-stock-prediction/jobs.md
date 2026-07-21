# jobs — 分 Phase 工作項

> 上游文件：`PRD.md`（需求與現況盤點）、`MVP.md`（MVP 範圍）。
> 原則：每個 Phase 結束都有**獨立可用的產出**，可隨時停在任一 Phase 而不爛尾。
> 工作項編號 `J<phase>.<seq>`；每項含產出與完成條件（DoD）。
> 程式落點：`repos/openclaw-jojo-dataops`（遵循該 repo Data Line Pattern 與 stdlib-only cron 慣例）。

## 總覽

| Phase | 名稱 | 產出 | 對應 |
|-------|------|------|------|
| 0 | 資料驗證與還原價 | 驗證過深度的 DB + 還原權息價 + point-in-time 介面 | MVP |
| 1 | 特徵與規則訊號 v1 | 每日 Top10 候選清單（手動執行） | MVP |
| 2 | 回測引擎與報告 | ≥3 年回測績效報告 | MVP |
| 3 | 自動化整合與資料擴充 | cron + Telegram 日報 + 融資券/處置股/TAIEX TR + 訊號 v2 | 後 MVP |
| 4 | ML 排序模型 | LightGBM 排序 + walk-forward 報告 | 後 MVP |
| 5 | 監控與迭代 | live 追蹤、敏感度分析、滑價模型 | 持續 |

依賴：0 → 1 → 2 串行（MVP 主線）；3 依賴 2；4 依賴 3；5 與 4 可並行。

相對首版規劃的主要修正（盤點原始碼後）：三大法人、逐日排行留存、每日更新狀態機、
品質檢查、cron/Telegram 模式**皆已存在**，原 Phase 0/3 大幅縮水；
新增的最大資料工作是**還原權息價**（原規劃低估）。

---

## Phase 0 — 資料驗證與還原價

目標：把「能力存在」變成「資料驗證過可用」，並補上 MVP 唯一的硬資料缺口（還原價）。
對應 PRD §2.2 G1、§2.3 V1–V3。

> **進度（2026-07-20 晚）**：J0.1 ✅、J0.2 ✅（見前版記錄）。
> **J0.3 ✅ 完成**：`tw_corporate_actions.py` 三組官方來源皆已在 Mac Mini 活測入庫——
> 除權息（TWSE TWT49U + TPEX exDailyQ）、減資恢復買賣參考價、**面額變更恢復買賣參考價**
> （13 筆，含 7780/6919/2327/8422 等彈性面額股）；0050 分割手動登錄。
> **J0.4 gating 歸零**（`--rebuild-factors --validate` exit 0）：46 件 raw-jump suspect 經
> 五輪分流全數收斂——減資入庫、面額變更入庫、新上市前 5 個觀測日無漲跌幅豁免、
> gap 內已有 factor 豁免、sparse/no-close gap（中間列 close 全 NULL，官方 change_amount
> 以參考價計）列 review/deferred（2740/6103/3629/1435 共 7 筆，文件化於資料線 docs）。
> **J0.4 ✅ 完成（含 D2 抽驗）**：逐事件跨源核對 5 檔（0050 分割/4414 減資/7780 面額變更/
> 2429 除權/2740 sparse-gap）全過；累積跨源比對 vs Yahoo Adj Close——0050 差 0.000012pp、
> 2603 差 0.0067pp、2330 窗內 12 筆季配與實際吻合（排除整批漏抓）。**Phase 0 資料部分關閉**。
> **J0.5 ✅ 完成**：`schema/tw_strong_signal.sql` 已包含資本事件表、`adjust_factors`、
> `ca_fetch_runs`/raw payload logs，以及 Phase 1 落地前需要的 `features_cache`、
> `predictions`、`signal_runs`；schema 以 `CREATE TABLE IF NOT EXISTS` / `CREATE INDEX IF NOT EXISTS`
> 保持可重入。
> **J0.6 ✅ 完成**（PR #7 `tw_strong_signal_data.py`）：唯一取數入口，`as_of` 強制、四條
> lookahead 契約（日價/排行 ≤ as_of、法人 < as_of、factor ex_date ≤ as_of、point-in-time
> universe）、12 項 self-check + Mac Mini 實機驗證通過——2330 累積 factor 與 D2 文件逐位一致，
> universe 2,027 檔、**≥5,000 萬流動性池 789 檔**（= Phase 1 實際選股池）。
> **J0.7 ✅ 完成**：`tw_strong_signal.py` / `tw_backtest.py` 骨架已接上 `MarketData(as_of)`。
> signal contract 固定輸出 `as_of`、`trade_date`、`features`、`score`、`rank`、`signals`、
> `data_quality`、`debug`；backtest 只 import 同一個 `build_signal_snapshot()`，並驗證 dry-run
> adapter 與 fixed-symbol T+1 execution。Mac Mini smoke：signal self-check、backtest self-check、
> `2026-07-17/2330` JSON、`2026-07-01..2026-07-17` dry-run / fixed-symbol 全通。
> J0.8 ✅ 完成：workspace `scripts/tw_stock_rankings.py` 已改為 shim，指向 repo canonical script，消除雙版本漂移。
> 接續：Phase 1 特徵工程，直接接 `build_signal_snapshot()` 與 gateway，不另開取數路徑。
> 教訓回饋：資本事件遠不只除權息（減資、面額變更、新上市無漲跌幅、無成交參考價重設
> 全部撞過一次）；audit 的 hardcode known-exception 清單由 `capital_events.suspend_*` 取代（收尾中）。

| Job | 內容 | DoD |
|-----|------|-----|
| J0.1 | **Mac Mini 上**驗證實際資料深度：`backfill_progress` 游標、`daily_prices`/`institutional_flows`/`rank_snapshots` 起訖日與缺日率。腳本已完成：`scripts/tw_strong_signal_data_audit.py`（openclaw-workspace 分支 `claude/tw-strong-signal-data-audit`），在 Mac Mini 執行 `python3 scripts/tw_strong_signal_data_audit.py --write-report` | 一頁 `docs/strong-signal-data-audit.md`：各表日期範圍/缺日率/pass-warn-fail 判定 |
| J0.2 | 深度不足 3 年處，用既有 `--backfill-twse-daily` / `--backfill-tpex-daily` / `--backfill-institutional-flows`（resume + sleep 節流）補齊 | MVP D1 通過 |
| J0.3 | **資本事件資料線**：新增 `tw_corporate_actions.py` 抓 TWSE/TPEX 除權息公告 + **分割/併股/減資**事件（或 FinMind 備援），入 `dividends`/`capital_events` 表 + 還原係數表；含停牌區間記錄（audit known-exception 的資料來源）；stdlib-only | 抽 5 檔與公開資料核對事件無漏；**0050 2025-06 分割**為必測案例 |
| J0.4 | 還原價計算與驗證：還原報酬 vs 公開還原序列；分割還原後 0050 序列在 2025-06 必須連續（否則基準報酬在該日出現假暴跌） | MVP D2 通過 |
| J0.5 | `schema/tw_strong_signal.sql`：`dividends`、`adjust_factors`、`features_cache`、`predictions`、`signal_runs`（比照既有 `fetch_runs` 模式） | schema 進 Git；`init_db` 可重入 |
| J0.6 | Point-in-time 取數介面：ATTACH market + rankings 兩庫，`as_of` 強制；含法人資料時點規則（MVP §6 保守版） | 測試：`as_of=D` 取不到 D+1 資料；法人只到 D-1 |
| J0.7 | 專案骨架：`tw_strong_signal.py` / `tw_backtest.py` 骨架 + stdlib `--self-check`（不引 pytest）；signal/backtest 共用 `build_signal_snapshot()`，只透過 `MarketData(as_of)` 取數 | signal contract、PIT/no-lookahead、backtest adapter、deterministic fixed-symbol smoke 全綠 |
| J0.8 | 順手清理（PRD §2.4）：workspace `scripts/tw_stock_rankings.py` 改為 shim，消除雙版本 | 兩處行為一致 |

**Phase 0 產出**：驗證過的歷史資料 + 還原價 + 取數介面。停在這裡，資料庫本身已升級。

---

## Phase 1 — 特徵與規則訊號 v1

目標：每日可手動產出 Top10 強勢股候選清單。刻意用簡單規則當基線。
（法人與排行動能特徵直接進 v1，因為資料現成——修正原規劃把法人排到 Phase 3 的安排。）

> **狀態校正（2026-07-21）**：J1.0 盤點完成，J1.1 blocker 已修復。正式
> `data/tw_strong_signal.sqlite` 已套新版 schema 並入庫 `official_dispositions`；
> 2026-07-20 smoke 確認 active common-stock 處置股會被 `MarketData(as_of).universe()`
> 排除。J1.1/J1.2 現可支援 J1.3。
>
> **交接（2026-07-21，Claude 接手）**：J1.3/J1.4/J1.5 實作完成——`tw_strong_signal_scoring.py`
> 新增（跨股 z-score 加權評分、缺值中性降級、`rank_top_n`/`build_reason`，13 項獨立 self-check），
> `tw_strong_signal.py` 改為對**整個 J1.1 股票池**評分後取 Top N（而非先截斷池子再算特徵，
> 修正了舊版「`selected=universe[:limit]`」讓 Top N 失去排序意義的問題），新增 `persist_snapshot()`
> 落地 `predictions`/`features_cache`/`signal_runs`（獨立可寫連線、單一 transaction、
> 同 `(as_of, symbol, signal_version)` upsert 不重複、CLI 預設不寫入避免除錯/backtest 污染正式表）
> 與 `build_report_text()`（`--format report`，cron/Telegram 慣例）。全部本機重跑驗證：
> `tw_backtest.py --self-check` 不受影響（`build_signal_snapshot()` 契約不變）。
> Mac Mini D6 已驗收：對正式 DB 連續 5 個交易日跑 `--persist --format report` 成功。

| Job | 內容 | DoD |
|-----|------|-----|
| J1.0 | **完成（2026-07-21）**：Phase 1 盤點任務。核對 PRD/MVP/jobs/RESET_HANDOFF 與實作現況；盤點 `tw_strong_signal.py`、`tw_strong_signal_data.py`、`tw_strong_signal_features.py`、`tw_official_dispositions.py`、`schema/tw_strong_signal.sql` 是否符合 Phase 1 範圍；確認 J1.1/J1.2 已完成項、未完成項、Phase 3 延後項、測試覆蓋、資料依賴、official disposition 最小入庫邊界與不做事項 | 判定：J1.1/J1.2 可支援 J1.3。必要驗證見下方「J1.0 盤點結果」 |
| J1.1 | **完成（2026-07-21）**：股票池模組：`security_category='common_stock'`、當日 active（`listed_date`/`delisted_date` + 當日 price row；避免用 mutable `is_active` 污染歷史）、日均成交值 ≥ 5,000 萬、最小官方 TWSE/TPEx 處置清單 schema/script + 正式 DB 入庫 + `MarketData(as_of)` 讀取排除。完整注意股、全額交割股、處置歷史回填與治理留 Phase 3 | self-check 含下市股情境（防存活者偏差）、官方處置 PIT 可見性、ETF/未上市/低流動性排除；正式 DB smoke：2026-07-20 `official_disposition_count=33`、>=5,000 萬池由 783 降至 762、移除 21 檔 active common-stock 處置股 |
| J1.2 | **完成**：特徵計算（純函式）：5/20/60 日**還原**報酬、量比（5 日均量/60 日均量）、波動度、排行動能（`entered_top20`、`rank_delta_1d`、成交值排名變化）、法人 5 日累計淨買超/成交量比；已接 `build_signal_snapshot()`，score 留 J1.3 | 手算 fixture 驗證；2026-07-17 實際資料全市場候選 789 檔約 3.1 秒 |
| J1.3 | **完成（2026-07-21）**：規則評分 `tw_strong_signal_scoring.py`：特徵 z-score 加權合成 → Top N；權重集中 `FEATURE_WEIGHTS`；缺值降級為中性貢獻（0，非填補/非剔除），母體已知值 <2 檔時整體降級為 undetermined | 13 項獨立 self-check（強/弱/中/缺值候選排序、tie-break、單樣本母體、reason 一致）+ wiring 5 項（top-N 排序、rank 對應、deterministic）全過 |
| J1.4 | **完成（2026-07-21）**：`persist_snapshot()` 落地 `predictions`/`features_cache`/`signal_runs`（獨立可寫連線、單一 transaction、upsert 冪等）；`tw_strong_signal.py` 主流程：取數 → 特徵 → 評分 → 快照 → Telegram-ready 摘要塊 `build_report_text()`（`--format report`，cron 交付慣例，`docs/cron.md`）；CLI `--persist` 預設關閉 | self-check 6 項（寫入正確性、upsert 冪等、run history 累加、report 非空）全過；Mac Mini formal DB D6：2026-07-14/15/16/17/20 連續 5 交易日 `--persist --format report` 手動執行成功 |
| J1.5 | **完成（2026-07-21）**：快照含「理由欄位」`build_reason()`：`top_contributors`（依 \|contribution\| 排序）+ `missing_features`，供人工覆核 | self-check 含 shape 驗證、missing 揭露、決定性排序 |

**Phase 1 產出**：每天一份可人工參考的候選清單（尚未驗證績效）。

### J1.0 盤點結果（2026-07-21）

驗證已過：

- `python3 scripts/tw_official_dispositions.py --self-check`
- `python3 scripts/tw_strong_signal_features.py`
- `python3 scripts/tw_strong_signal_data.py --self-check`
- `python3 scripts/tw_strong_signal.py --self-check`
- `python3 scripts/tw_backtest.py --self-check`
- `python3 -m py_compile scripts/tw_official_dispositions.py scripts/tw_strong_signal_data.py scripts/tw_strong_signal_features.py scripts/tw_strong_signal.py scripts/tw_backtest.py`

實機 smoke（2026-07-20）：

- `tw_strong_signal_data.py --as-of 2026-07-20 --symbol 2330`：latest trading day 2026-07-20；common-stock universe 2,012；>=5,000 萬池 783；法人最大日 2026-07-17；ranking rows 1,400。
- `tw_strong_signal.py --as-of 2026-07-20 --limit 783 --format json`：全池 783 檔、`no_lookahead_ok=true`；有 ranking feature 262 檔、缺 ranking feature 521 檔；法人缺 1 檔；`return_60d_adj`/`volume_ratio_5d_60d` 缺 3 檔、`volatility_20d_adj` 缺 2 檔、`turnover_rank_delta_1d` 缺 605 檔。ranking 缺口多數是設計預期：Yahoo 快照只留各榜前 100，J1.3 必須把 null/degrade 當正式輸入處理。
- 官方處置 temp DB smoke：`tw_official_dispositions.py --fetch-all --start-date 2026-07-20 --end-date 2026-07-20 --db /tmp/tw_official_dispositions_inventory.sqlite` 成功，TWSE 13、TPEX 22、0 skipped；其中 23 筆為 `common_stock`。

已修復：

- **J1.1 blocker fixed（2026-07-21）**：正式 `data/tw_strong_signal.sqlite` 已套新版 schema；補入 `official_dispositions`、`signal_runs`、`features_cache`、`predictions`。正式入庫 smoke：TWSE 13、TPEX 22、0 skipped；其中 23 筆為 `common_stock`。排除 smoke：2026-07-20 流動性池由 783 降至 762，移除 21 檔 active common-stock 處置股（1515、1718、2434、2466、2492、3055、3090、4169、4542、4556、4707、6173、6174、6617、6831、6907、7714、8027、8096、8261、8383）。
- 正式 DB migration 前已備份：`data/analysis/tw_strong_signal_pre_j11_schema_20260721T1122.sqlite`，SHA-256 `cc508798c4ec1935dff0aefb460487fd63fa7c05c1e4632da4caa186a38e02a4`。

### J1.3–J1.5 實作結果（2026-07-21，Claude）

已完成：

- `scripts/tw_strong_signal_scoring.py`（新檔，純函式無 DB 存取）：`FEATURE_WEIGHTS` 單一 config（v1 等權重，含 `return_5d_adj`/`return_20d_adj`/`volume_ratio_5d_60d`/`entered_top20`/`rank_delta_1d`/`institutional_net_buy_volume_ratio_5d`，對齊 MVP.md §2.3）；`cross_sectional_zscores()`、`score_universe()`、`rank_top_n()`（score 降冪、symbol 升冪 tie-break，確保輸出不受 DB 迭代順序影響）、`build_reason()`。
- `tw_strong_signal.py` 改寫評分路徑：**永遠對整個 `scoring_pool`（J1.1 `universe`，`symbol` 查詢時額外併入該檔）算特徵與分數，取 Top N 才切片**；修正舊版「先截斷再算特徵」的設計缺陷（原本 `selected=universe[:limit]` 只是 DB 回傳順序，不是依訊號強度排序）。
- `persist_snapshot(snapshot, db_path=None)`：獨立可寫連線（與 `MarketData` 唯讀 ATTACH 分開）、單一 transaction 寫 `signal_runs`+`features_cache`+`predictions`；同 `(as_of, symbol, signal_version)` upsert，`signal_runs` 每次執行新增一列（執行歷史）。
- `build_report_text(snapshot, run_id)`：`--format report`，狀態行在前、內容永不為空（`docs/cron.md` 慣例）。
- CLI 新增 `--persist`（預設關閉，避免除錯/backtest 呼叫寫入正式表）與 `--format report`。

驗證（本機重跑，非僅信任交接文件）：

- `python3 scripts/tw_strong_signal_scoring.py`：13 項 PASS（強/弱/中/缺值候選排序方向、tie-break 決定性、單樣本母體不可比、reason 前 3 貢獻排序、缺值揭露）。
- `python3 scripts/tw_strong_signal.py --self-check`：28 項 PASS（含既有 J0.6/J1.1/J1.2 斷言 + 新增 top-N 排序、rank 對應、跨呼叫 deterministic、`persist_snapshot` 寫入正確性、upsert 冪等、report 非空）。
- `python3 scripts/tw_backtest.py --self-check`：7 項 PASS，未受影響（`build_signal_snapshot()` 外部契約不變，backtest 完全不知道內部評分邏輯換了）。
- 端到端 CLI 冒煙（合成 DB，透過環境變數指定，模擬真實 CLI 呼叫路徑）：`--persist --format report` exit 0，`signal_runs`/`predictions` 正確寫入且 rank/score 與快照一致。
- 手算驗證評分數學：fixture 中 1111 vs 4444 的 5/20 日報酬 z-score、缺值降級（母體已知值 <2 → 全體 None，非個別 0）、`score = Σ(weight × z)` 逐項對得上。

### Mac Mini MVP D6 驗收（2026-07-21，JoJo）

正式 DB 驗收：

- D6 前備份：`data/analysis/tw_strong_signal_pre_d6_persist_20260721T1330.sqlite`，SHA-256 `f3809f759fa02b4130448ec8ffa89cfae463740046cac70b2494505785e07ba0`。
- 使用 PR #8 merge commit `3eb41af` 的乾淨 worktree 執行，避免本機 dirty workspace 的 unrelated changes 影響驗收。
- 自測：`tw_strong_signal.py --self-check` 28 項 PASS；`tw_strong_signal_scoring.py` 13 項 PASS；`tw_backtest.py --self-check` 7 項 PASS；`py_compile` PASS。
- 最近 5 個交易日 `--persist --format report` 全成功：2026-07-14、2026-07-15、2026-07-16、2026-07-17、2026-07-20。
- 正式 DB 寫入結果：`signal_runs=5`、`predictions=50`、`features_cache=50`；每個 as_of 都有 rank 1–10、`status=ok`、`prediction_count=10`。
- Report 輸出可讀且非空；每檔含 score、top contributors、missing feature 揭露。2026-07-20 Top 10：6505、6243、2634、4541、1810、2527、1232、6957、8039、6213。
- `reason_json` shape 驗證通過：50 筆皆含 `top_contributors` 與 `missing_features`；其中 27 筆揭露 missing features。

未完成 / 待處理：

- backtest 仍是 J0.7 skeleton，只驗證 adapter/T+1 route；策略績效與成本規則仍在 Phase 2（J2.1 起）。
- 權重為 v1 等權重基準，非回測調校結果；Phase 2 有真實績效數字後可回頭調整並記錄理由。

下一步判定：

- **Phase 1（J1.0–J1.5）全部完成，MVP D6 已通過**。
- 可進 Phase 2：J2.1 vectorbt 技術評估/自寫向量化引擎、J2.2 成本與成交規則、J2.3 回測引擎（import 同一份 J1.2/J1.3 特徵與評分函式）。

---

## Phase 2 — 回測引擎與報告（MVP 完成線）

目標：用數字回答「這個訊號扣完成本行不行」。研究路徑，允許 pandas/numpy/vectorbt。

> **進度（2026-07-21，Claude）**：J2.1 完成，**決定採用開源 `vectorbt`**（不自寫向量化引擎）。
> 獨立 venv 實測（非文件推測）：`size_type="targetpercent"` + rank 權重矩陣表達每日/H 日排名
> 換股；`price` 陣列設 NaN 精確擋下漲停日成交（0 筆訂單，直接查驗）；`fees` 逐筆陣列驗證買
> 0.1425%/賣 0.4425%（含稅）不對稱成本。效能：合成 800 檔 × 750 交易日、每日全換股最壞情境
> 28 秒完成（DoD 門檻 10 分鐘）。CAGR/Sharpe/MDD/勝率皆內建，J2.5 免手刻。依賴隔離：
> `requirements-backtest.txt` 建獨立 venv，cron 路徑（`tw_strong_signal.py`）維持 stdlib-only
> 不受影響。決策記錄：`docs/tw-backtest-engine.md`。
>
> **J2.2/J2.3 完成（2026-07-21，Claude）**：`tw_backtest_costs.py`（純 stdlib，17 項 self-check）
> 封裝台股 tick 網格、買 0.1425%/賣 0.4425% 不對稱成本、漲停擋買/跌停擋賣。`tw_backtest.py`
> 重寫為 vectorbt 引擎，**每換股日呼叫同一個 `build_signal_snapshot()`**（結構保證 J2.4）；
> 進場等權重、持有到出榜才賣（每筆訂單方向明確→成本精確）；T+1 開盤成交、漲停 NaN 拒單。
> **效能修正**：J1.2 `build_features` 原每檔掃全表 O(N²)→3 年回測外推 11 分鐘超標；新增
> `build_features_pooled()`（一次分組 O(N)，輸出逐 byte 相同，self-check 斷言），
> `build_signal_snapshot()` 改用它，**live 訊號與回測同時快 9.5 倍**；合成 800 檔每換股日
> 4.47s→0.47s，3 年回測外推 70 秒 = 1.2 分鐘。cron 路徑（features/scoring/data/signal）
> 無 venv self-check 仍全過。

| Job | 內容 | DoD |
|-----|------|-----|
| J2.1 | **完成（2026-07-21）**：vectorbt 技術評估：玩具策略驗證能否表達「每日排名換股 + 漲停不成交 + 台股成本」；不合用則自寫向量化引擎。含依賴管理決策（獨立 venv，不污染 cron 路徑） | 一頁決策記錄（採用/自寫 + 理由）：`docs/tw-backtest-engine.md`，三項硬需求 + H=5 持有 + 效能全部實測通過 |
| J2.2 | **完成（2026-07-21）**：成本與成交規則 `tw_backtest_costs.py`（純 stdlib）：手續費 0.1425%×2（可設折扣）、證交稅 0.3%、D+1 開盤成交、漲停不買/跌停不賣、台股 tick 網格漲跌停價 | 17 項 self-check（買賣不對稱成本、折扣、tick、漲跌停價、方向性擋單）全過；回測引擎整合的擋單端到端測試通過（**MVP D4**） |
| J2.3 | **完成（2026-07-21）**：回測引擎 `tw_backtest.py`（vectorbt）：區間、持有 H 日、Top N、進場等權重；**每換股日 import 同一個 `build_signal_snapshot()`**（含 J1.2/J1.3 特徵評分）；含 `build_features_pooled` O(N) 效能修正 | 11 項 self-check（T+1、不對稱費、決定性、選股==live、漲停擋單端到端）全過；合成 800 檔 3 年回測外推 1.2 分鐘 **< 10 分鐘 DoD**；正式 DB 實測時間留 Mac Mini |
| J2.4 | 前視偏差測試：抽 20 個歷史日期，回測選股 == 截斷資料 live 選股；含法人時點規則驗證 | MVP D3 通過（不過則 Phase 2 不算完成） |
| J2.5 | 績效報告：總報酬/CAGR/Sharpe/MDD/勝率/換手率/逐年分拆，vs 0050 buy-and-hold（`daily_prices` + `etf_navs` 現成） | MVP D5 通過 |
| J2.6 | 可重現性：同參數重跑結果一致 | MVP D7 通過 |

**Phase 2 產出 = MVP**：閉環完成。此時停下來讀報告，決定訊號值不值得繼續投入。

---

## Phase 3 — 自動化整合與資料擴充

目標：接上既有 OpenClaw cron/Telegram 體系做到零人工，並補次要資料缺口。
（排程、重試、Telegram 回報模式已存在——本 Phase 是「加一條資料線」，不是建基礎設施。）

| Job | 內容 | DoD |
|-----|------|-----|
| J3.1 | 新增 cron job（建議 `daily_tw_strong_signal_1700`，在 06:00/12:00 資料更新與 14:00 排行之後；Gateway `command` payload，遵循 `docs/cron.md` 安全規範與非空摘要交付規則） | 連續 10 交易日零人工（PRD 成功指標）；cron note 進 docs |
| J3.2 | 失敗處理：依賴資料未就緒（`daily_update_runs` 非 ok）時 skip-with-notice；比照既有 primary/retry 兩段式模式加重試 slot | 人為斷資料演練：收到告警且隔日自動補齊 |
| J3.3 | 法人時點規則放寬：cron 固定 17:00 後跑則 D 日法人可用；改規則 + 重跑前視測試 + 回測對照 | 前視測試通過；對照報告落地 |
| J3.4 | 融資融券餘額入庫（TWSE/TPEX 官方或 FinMind；PRD G2） | 回填 ≥ 1 年，品質檢查通過 |
| J3.5 | 處置股/注意股清單入庫（PRD G3），股票池改用真實清單 | 取代「連續漲停」近似 |
| J3.6 | TAIEX Total Return 指數歷史入庫（PRD G5），報告加第二基準 | 基準曲線抽查相符 |
| J3.7 | 訊號 v2：納入融資券特徵（券資比、融資增減率），回測對照 v1 | v1 vs v2 對照報告 |
| J3.8 | 日報升級：每日清單 + 昨日預測對答案摘要進 Telegram | 每日自動收到 |

**Phase 3 產出**：不碰鍵盤，每天自動收到候選清單與追蹤摘要。

---

## Phase 4 — ML 排序模型

目標：LightGBM 排序取代（或輔助）規則加權，walk-forward 誠實評估。研究路徑。
前置：資料 ≥ 3 年含法人/融資券；Phase 2 規則基線績效已知。

| Job | 內容 | DoD |
|-----|------|-----|
| J4.1 | 標籤設計：未來 H 日相對基準超額報酬（回歸或分箱排序標籤），文件化理由 | `docs/label-design.md` |
| J4.2 | 訓練資料管線：由 point-in-time 介面產生特徵矩陣，嚴防標籤洩漏 | 洩漏測試（打亂日期應使績效歸零） |
| J4.3 | LightGBM 排序 + walk-forward（如訓 2 年、驗 3 月、滾動），固定 seed | 各期指標落地 |
| J4.4 | 模型 vs 規則 v1/v2 回測對照（同成本、同區間） | 對照報告；ML 未顯著勝出則明文記錄並停留規則版 |
| J4.5 | 每日推論整合：模型檔版本化，predictions 記 model version；cron 路徑依賴決策（推論需 lightgbm，違反 stdlib-only——選項：venv 包裝 / 匯出線性近似 / 推論留研究路徑，J4.5 內定案） | 快照可追溯產出模型 |
| J4.6 | 候選新特徵評估：新聞熱度（`tw_market_news`）、題材熱度（`tw_theme_heat`）、供應鏈圖譜（`tw_coverage_graph`）——各做增量 ablation | 每特徵一行結論（進/不進） |
| J4.7 | 特徵重要度與各期穩定性檢查 | 報告落地 |

**Phase 4 產出**：一份誠實的「ML 有沒有比簡單規則強」結論；強才上線，不強留規則。

---

## Phase 5 — 監控與迭代（持續性）

目標：live 表現持續可見，防回測自欺與策略衰退。

| Job | 內容 | DoD |
|-----|------|-----|
| J5.1 | live 追蹤報表:滾動 20/60 日命中率、live 累積超額報酬 vs 回測期望 | 週報自動產出 |
| J5.2 | 回測-實盤落差監控：live 績效落出回測分佈（如低於歷史滾動 5 百分位）即告警 | 告警演練通過 |
| J5.3 | 參數敏感度：對 N、H、過濾門檻掃描，確認績效非單點運氣 | 敏感度熱圖報告 |
| J5.4 | 滑價模型：以成交值估衝擊成本，取代「開盤全額成交」假設 | 報告加註滑價後數字 |
| J5.5 | 季度回顧：訊號衰退、資料品質、重訓/調參決策 | 每季一頁記錄 |

---

## 里程碑與時間感（個人 side project 節奏，僅供參考）

| 里程碑 | 內容 | 粗估投入 |
|--------|------|----------|
| M1 | Phase 0 完成：資料驗證 + 還原價 | 2–3 個週末（主要花在除權息資料線） |
| M2 | Phase 1 完成：每日清單 | 1–2 個週末 |
| M3 | **Phase 2 完成：MVP，回測報告出爐** | 2–3 個週末 |
| M4 | Phase 3 完成：全自動 + 資料擴充 | 1–2 個週末（cron 模式現成） |
| M5 | Phase 4 完成：ML 對照結論 | 3–4 個週末 |

原則提醒：M3 是決策點——回測若顯示扣成本後無 alpha，優先迭代訊號（回 Phase 1）
而不是直接跳 ML；ML 救不了沒有資訊量的特徵。
