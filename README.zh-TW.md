# NCKU-RTOS-2026 Virtual Power Plant (VPP) Dynamic Scheduling System

![Python](https://img.shields.io/badge/language-Python-3776AB?logo=python&logoColor=white)
![pytest](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)
![Scheduling](https://img.shields.io/badge/scheduling-offline%20DFS%20%2B%20online%20admission-555)
![License: MIT](https://img.shields.io/badge/license-MIT-yellow)

[English](README.md) | 繁體中文

我們做了一個虛擬電廠的 72 小時排程器。週期任務以 Frame 為單位用 DFS 離線排程，sporadic 與 aperiodic 任務走線上准入控制，Level 2 另加入預測誤差、市場承諾售電與電池模型。可執行 10 種情境的批次分析或單一 Demo 檔，並可在 Level 1（基礎）與 Level 2（進階）之間切換。

## 快速執行指南

環境：
- Python 3（以 3.13 驗證），請在 repository 根目錄執行。
- `src/scheduler.py`、`src/evaluator.py`、`src/task_generator.py` 只用標準函式庫。
- `src/aperiodic_n_sporadic_gen.py` 需要 NumPy（`pip install numpy`）；測試需要 pytest（`pip install pytest`）。

### Level 1 / Level 2
預設為 Level 2（`LEVEL2_ENABLED = True`）。要測 Level 1，請在 `src/scheduler.py` 第 40 行改為 `False`：
```python
# src/scheduler.py
LEVEL2_ENABLED = False  # 設為 False 執行 Level 1，設為 True 執行 Level 2
```
Level 1 直接使用綠電預測值與 Level 1 電池模型，不做市場承諾售電，輸出檔名不帶 `level2_` 前綴。

### 批次模式與 Demo 模式
若 `input/aperiodic_n_sporadic.json` 存在，`src/scheduler.py` 執行 Demo 模式，否則執行批次模式。repository 附有範例檔，所以剛 clone 後執行 `python src/scheduler.py` 會進入 Demo 模式。

批次模式會模擬 `output/sporadic_aperiodic_task/` 下的 10 組 `scenario_*.json`。請先移走 Demo 檔：
```bash
mv input/aperiodic_n_sporadic.json input/aperiodic_n_sporadic.json.bak
python src/scheduler.py
```
- 各情境的檔案寫入 `output/`：`schedule_result_level2_<情境>.json`、`acceptance_test_log_level2_<情境>.json`、`evaluation_results_level2_<情境>.json`。
- 接著對所有情境執行 evaluator（`src/evaluator.py --batch-level2`，Level 1 為 `--batch-scenarios`）。總表為 `output/evaluation_results_level2_summary.json`（Level 1：`output/evaluation_results_summary.json`）。
- `scenario_01_uniform` 的結果也會複製成 `schedule_result.json`、`acceptance_test_log.json`、`evaluation_results.json`，因此 `evaluation_results.json` 只涵蓋 scenario_01。

要回到 Demo 模式，把檔案移回來：`mv input/aperiodic_n_sporadic.json.bak input/aperiodic_n_sporadic.json`。

Demo 模式：將突發任務檔放到 `input/aperiodic_n_sporadic.json`（取代範例檔），執行 `python src/scheduler.py`。批次情境會被跳過，結果直接覆蓋 `output/schedule_result.json`、`output/acceptance_test_log.json`、`output/evaluation_results.json`。repository 內這三個檔案是以附帶範例檔執行的結果。

### 執行測試
```bash
pip install pytest
python -m pytest
```
請用 `python -m pytest`，不要直接執行 `pytest`，否則無法 import `src`。這會執行 `tests/test_acceptance_tester.py` 與 `tests/test_state_machine.py` 中的 20 個測試。`tests/test_offline_pipeline.py` 是獨立整合測試腳本：`python -m tests.test_offline_pipeline`。

### 重新產生輸入資料（選用）
- `python src/task_generator.py` 重建 `output/task_set.json`。種子固定，產出與 repository 內的檔案相同。
- `python src/aperiodic_n_sporadic_gen.py` 覆寫 10 個情境檔（3 組 uniform、3 組 concentrated、4 組 mixed；種子 2027–2036）。repository 內的 `scenario_01_uniform.json` 與生成器目前的產出不同，重新執行會改變它。

## 檔案結構

```text
Embedded-RTOS-Scheduler/
├── report.pdf                       # 系統設計報告
├── runtime_config.json              # Level 2 參數的副本（程式不會讀取）
├── docs/level2_model.md             # Level 2 形式化模型（限制式與虛擬碼）
├── src/
│   ├── scheduler.py                 # 進入點：Level 1/2 切換、批次/Demo 模式
│   ├── evaluator.py                 # 評估與計分
│   ├── advanced_scheduler.py        # Level 2：預測誤差、電池模型、市場承諾售電、延後
│   ├── task_generator.py            # 週期任務集 -> output/task_set.json
│   ├── aperiodic_n_sporadic_gen.py  # 情境生成器 (NumPy)
│   └── engine/
│       ├── offline_planner.py       # 以 Frame 為單位的 DFS 排程器
│       ├── acceptance_tester.py     # 線上准入控制
│       ├── main_scheduler.py        # 72 小時模擬與分派迴圈
│       ├── power_tracer.py          # 注水式能量溯源（k 矩陣）
│       ├── state_machine.py         # 發電機 / 電池狀態轉移驗證
│       ├── data_loader.py           # JSON 載入
│       └── models.py                # 資料類別
├── tests/                           # pytest 測試 + 日前排程器整合測試腳本
├── input/                           # processor_settings.json、price_72hr.json、Demo 範例檔
└── output/                          # task_set.json、10 組情境、排程結果與總表
```

`runtime_config.json` 的數值與 `src/scheduler.py` 中的 `LEVEL2_CONFIG` 相同。排程器實際使用 `LEVEL2_CONFIG`，要調整 Level 2 參數請改它。

## 週期任務集生成

`src/task_generator.py` 抽出 `N = 6–10` 個任務。repository 內的 `output/task_set.json`（種子 2026）為 `N = 6`、`DW ≈ 0.99`，72 小時內共 44 個 Job。

- 執行時間為 `[1]*(N-4) + [2, 2, 3, 3]`。兩個 `e=2` 的任務不可中斷；兩個 `e=3`（`d=3`）的任務可中斷。不可中斷的 `e=3, d=3` 任務沒有鬆弛，必須剛好占一段連續 3 小時，會造成排程碎片化並讓 DFS 需要更多回溯。
- `(N-2)//2` 個任務使用 `p=6`，提供穩定的基底負載。生成器不以特定 DW 為目標；`validate()` 只接受 `0.7 ≤ DW ≤ 1.0`。
- 前 `N-2` 個任務中其餘的共用一個從 `{11, 12}` 抽出的週期，最後兩個任務從 `{15, 18, 21, 24}` 抽出兩個不同週期。前 `N-2` 個任務的 `d` 從 `[6, p]` 抽取，以滿足 `f=3` 的 Frame 可視性 (`2f−gcd(f,p)≤d`)。
- 最後兩個任務 `d=3`（`=e`），滿足「至少 20% 的任務 `d=e`」；前段保留寬鬆 Deadline (≥6)，讓排程器有空間平移能量與調整機組升降載。
- `RANDOM_SEED=2026`。`validate()` 檢查 Frame 可視性、`0.7 ≤ DW ≤ 1.0`、72 小時內 Job 數大於 30、至少 3 種不同週期，以及不可中斷任務 `d ≥ e`。生成器只抽一次，`main()` 只有在驗證通過時才寫入 `output/task_set.json`。

## 核心排程引擎

雙層架構：週期任務先離線排程，其餘任務再經線上准入與分派。

### 1. 日前離線排程
`src/engine/offline_planner.py`，處理週期任務。

對連續 24 個 3 小時 Frame（共 72 小時）做 DFS：
1. 針對當前 Frame 內的活躍 Job，列舉這 3 個小時的時段分配組合。
2. 候選排法依長度最長、最早優先嘗試，讓任務盡早完成。若某排法讓剩餘工作量超過 Deadline 前剩下的小時數，直接剪枝。
3. 每個組合逐小時驗證：先吸收綠電預測量，再由火力機組（出力、升降載速率、最短開/關機時間限制）與電池放電（放電上限與 SOC 限制）補足負載。若該 Frame 失敗或後續 Frame 無法完成，就還原發電機/電池快照並嘗試下一個組合（回溯）。

產出：72 小時基載排程表，以及每小時的剩餘算力 (Slack Capacity)（火力與電池剩餘容量）。

### 2. 線上准入控制
`src/engine/acceptance_tester.py`，處理 sporadic 與 aperiodic 任務。
- Sporadic（硬限制）：檢查 Deadline 前是否有足夠小時數（不可中斷任務需連續）其剩餘算力 ≥ 任務需求。有就預留，否則拒絕。
- Aperiodic（軟限制）：抵達時若剩餘算力完全放不下就拒絕，否則放入等候佇列。系統每小時依序掃描佇列：放得下的任務執行（可中斷任務每次 1 小時，不可中斷任務一次排完整段），放不下的先跳過，讓後面的任務仍可執行（Backfilling，避免隊頭阻塞）。佇列中的任務等待超過 24 小時，或剩餘執行時間超過 72 小時內剩下的時數，就會被丟棄。

### 3. 線上即時分派與能量溯源
`src/engine/main_scheduler.py`、`src/engine/power_tracer.py`。每小時：
1. 先全額吸收邊際成本為 0 的再生能源，得到淨負載。
2. 以各機組的最低出力為起點，依 `cost_variable` 升冪提高出力以補足淨負載。火力不足時用電池放電。
3. 若市場電價高於已開機機組的變動成本，將該機組推到升降載限制允許的最高出力，並賣出剩餘電力。
4. `power_tracer.py` 以注水演算法對應每個供電來源（綠電、發電機、電池）與個別任務，產出 $k_{j,i,t}$ 矩陣。任何任務供電不足都會拋出錯誤，剩餘電力記為售電量，因此每小時都滿足「供給 = 用電 + 售電」。

### 4. Level 2 進階動態重排程
`src/advanced_scheduler.py`，僅在 `LEVEL2_ENABLED = True` 時啟動；參數位於 `src/scheduler.py` 的 `LEVEL2_CONFIG`。
1. 實際出力 = 預測值乘上 ±20% 內均勻抽樣的相對誤差（`forecast_error_ratio`，種子 `random_seed = 2026`），並限制在機組容量以內。
2. 系統每小時承諾售出固定電量（`5.0 × commitment_ratio = 4.0` MWh）。缺口依 `penalty_rate`（每 MWh）計罰；超出承諾的售電以 `(realtime_price_multiplier − 1) × 電價` 取得額外收益。
3. 電池模型有充放電效率、自放電與 SOC 相依的放電上限。每放電 1 MWh 產生老化成本，計入 Level 2 調整後目標值。多餘電力透過 `<電池>_chg` 虛擬任務為電池充電。
4. 實際綠電低於預測時，系統記錄電池放電與火力升載餘裕可補足多少缺口（僅為估算）；接著由上述分派流程補足淨負載。若估算餘裕不足，該小時正在執行的 aperiodic job（ID 以 `a_` 開頭）會被延後並踢回佇列，優先保證週期性與 sporadic 任務的供電。
