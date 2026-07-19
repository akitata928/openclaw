# jobs — 分 Phase 工作項

> 上游文件：`RPD.md`（需求與現況盤點）、`MVP.md`（MVP 範圍）。
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
對應 RPD §2.2 G1、§2.3 V1–V3。

| Job | 內容 | DoD |
|-----|------|-----|
| J0.1 | **Mac Mini 上**驗證實際資料深度：`backfill_progress` 游標、`daily_prices`/`institutional_flows`/`rank_snapshots` 起訖日與缺日率（可沿用 `data_quality_checks` 機制落地結果） | 一頁 `docs/strong-signal-data-audit.md`：各表日期範圍/缺日率/結論 |
| J0.2 | 深度不足 3 年處，用既有 `--backfill-twse-daily` / `--backfill-tpex-daily` / `--backfill-institutional-flows`（resume + sleep 節流）補齊 | MVP D1 通過 |
| J0.3 | **除權息資料線**：新增 `tw_corporate_actions.py` 抓 TWSE/TPEX 除權息公告（或 FinMind 備援），入 `dividends` 表 + 還原係數表；stdlib-only | 抽 5 檔與公開資料核對事件無漏 |
| J0.4 | 還原價計算與驗證：還原報酬 vs 公開還原序列 | MVP D2 通過 |
| J0.5 | `schema/tw_strong_signal.sql`：`dividends`、`adjust_factors`、`features_cache`、`predictions`、`signal_runs`（比照既有 `fetch_runs` 模式） | schema 進 Git；`init_db` 可重入 |
| J0.6 | Point-in-time 取數介面：ATTACH market + rankings 兩庫，`as_of` 強制；含法人資料時點規則（MVP §6 保守版） | 測試：`as_of=D` 取不到 D+1 資料；法人只到 D-1 |
| J0.7 | 專案骨架：`tw_strong_signal.py` / `tw_backtest.py` 空殼 + 測試框架（比照 repo 現有腳本自帶 `--test-mode` 慣例，或加 pytest——擇一記錄決策） | 空跑綠燈 |
| J0.8 | 順手清理（RPD §2.4）：workspace `scripts/tw_stock_rankings.py` 改為 shim，消除雙版本 | 兩處行為一致 |

**Phase 0 產出**：驗證過的歷史資料 + 還原價 + 取數介面。停在這裡，資料庫本身已升級。

---

## Phase 1 — 特徵與規則訊號 v1

目標：每日可手動產出 Top10 強勢股候選清單。刻意用簡單規則當基線。
（法人與排行動能特徵直接進 v1，因為資料現成——修正原規劃把法人排到 Phase 3 的安排。）

| Job | 內容 | DoD |
|-----|------|-----|
| J1.1 | 股票池模組：`security_category='common_stock'`、當日 active（`listed_date`/`delisted_date`/`is_active`）、日均成交值 ≥ 5,000 萬、「連續漲停」近似剔除處置股 | 單元測試含下市股情境（防存活者偏差） |
| J1.2 | 特徵計算（純函式）：5/20/60 日**還原**報酬、量比（5 日均量/60 日均量）、波動度、排行動能（`entered_top20`、`rank_delta_1d`、成交值排名變化）、法人 5 日累計淨買超/成交量比 | 2–3 檔手算樣本驗證；全市場單日 < 5 分鐘（stdlib + SQL） |
| J1.3 | 規則評分：特徵 z-score 加權合成 → Top N；權重集中一個 config；含降級路徑（排行資料缺時僅用價量+法人，RPD §10 Yahoo 風險） | 確定性測試：同輸入同輸出 |
| J1.4 | `predictions` 落地 + `tw_strong_signal.py` 主流程：取數 → 特徵 → 評分 → 快照（股票、分數、特徵值、訊號版本號）→ Telegram-ready 摘要塊（cron 交付慣例，`docs/cron.md`） | 連續 5 交易日手動執行成功（MVP D6） |
| J1.5 | 快照含「理由欄位」：每檔列主要得分來源，供人工覆核 | 隨機抽查合理 |

**Phase 1 產出**：每天一份可人工參考的候選清單（尚未驗證績效）。

---

## Phase 2 — 回測引擎與報告（MVP 完成線）

目標：用數字回答「這個訊號扣完成本行不行」。研究路徑，允許 pandas/numpy/vectorbt。

| Job | 內容 | DoD |
|-----|------|-----|
| J2.1 | vectorbt 技術評估：玩具策略驗證能否表達「每日排名換股 + 漲停不成交 + 台股成本」；不合用則自寫向量化引擎。含依賴管理決策（獨立 venv，不污染 cron 路徑） | 一頁決策記錄（採用/自寫 + 理由） |
| J2.2 | 成本與成交規則：手續費 0.1425%×2（可設折扣）、證交稅 0.3%、D+1 開盤成交、漲停不買/跌停不賣 | MVP D4 通過 |
| J2.3 | 回測引擎 `tw_backtest.py`：區間、持有 H 日、Top N、等權重；**import Phase 1 同一份特徵/評分函式** | 3 年回測 < 10 分鐘 |
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
| J3.1 | 新增 cron job（建議 `daily_tw_strong_signal_1700`，在 06:00/12:00 資料更新與 14:00 排行之後；Gateway `command` payload，遵循 `docs/cron.md` 安全規範與非空摘要交付規則） | 連續 10 交易日零人工（RPD 成功指標）；cron note 進 docs |
| J3.2 | 失敗處理：依賴資料未就緒（`daily_update_runs` 非 ok）時 skip-with-notice；比照既有 primary/retry 兩段式模式加重試 slot | 人為斷資料演練：收到告警且隔日自動補齊 |
| J3.3 | 法人時點規則放寬：cron 固定 17:00 後跑則 D 日法人可用；改規則 + 重跑前視測試 + 回測對照 | 前視測試通過；對照報告落地 |
| J3.4 | 融資融券餘額入庫（TWSE/TPEX 官方或 FinMind；RPD G2） | 回填 ≥ 1 年，品質檢查通過 |
| J3.5 | 處置股/注意股清單入庫（RPD G3），股票池改用真實清單 | 取代「連續漲停」近似 |
| J3.6 | TAIEX Total Return 指數歷史入庫（RPD G5），報告加第二基準 | 基準曲線抽查相符 |
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
