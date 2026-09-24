# NCKU-RTOS-2026 Virtual Power Plant (VPP) Dynamic Scheduling System

![Python](https://img.shields.io/badge/language-Python-3776AB?logo=python&logoColor=white)
![pytest](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)
![Scheduling](https://img.shields.io/badge/scheduling-offline%20DFS%20%2B%20online%20admission-555)
![License: MIT](https://img.shields.io/badge/license-MIT-yellow)

[English](README.md) | **繁體中文**

## 🚀 快速執行指南 (Quick Start & Execution Guide)

本專案同時支援 **「自動批次分析 10 種情境」** 以及 **「單一檔案 Demo 測試」**，並且可以在 Level 1 (基礎要求) 與 Level 2 (進階動態排程) 之間切換。

### 0. 執行環境
- Python 3（以下指令已用 Python 3.13 驗證），請在 repository 根目錄執行。
- 排程主程式 (`src/scheduler.py`)、評估程式 (`src/evaluator.py`) 與週期任務生成器 (`src/task_generator.py`) 只使用 Python 標準函式庫。
- 情境生成器 `src/aperiodic_n_sporadic_gen.py` 需要 NumPy：`pip install numpy`。
- 測試需要 pytest：`pip install pytest`。

### 1. Level 1 / Level 2 模式切換
專案預設開啟 **Level 2 進階動態排程模式**（`LEVEL2_ENABLED = True`）。
若須測試 Level 1，請至 `src/scheduler.py` 第 40 行，將 `LEVEL2_ENABLED` 變數修改為 `False`：
```python
# src/scheduler.py
LEVEL2_ENABLED = False  # 設為 False 執行 Level 1，設為 True 執行 Level 2
```
切換後重新執行程式，模擬會直接使用綠電預測值與 Level 1 的電池模型，且不做市場承諾售電。Level 1 的輸出檔名不帶 `level2_` 前綴。

### 2. 批次模式與 Demo 模式
`src/scheduler.py` 只用一個條件判斷模式：若 `input/aperiodic_n_sporadic.json` 存在就執行 **Demo 模式**，否則執行 **批次模式**。
repository 內已附一份範例 `input/aperiodic_n_sporadic.json`，因此剛 clone 下來時執行 `python src/scheduler.py` 會進入 **Demo 模式**。

### 3. 重現 10 種情境分析 (批次模式)
批次模式會模擬 `output/sporadic_aperiodic_task/` 底下的 10 組 `scenario_*.json`。請先移走或改名 Demo 檔案，然後執行：
```bash
mv input/aperiodic_n_sporadic.json input/aperiodic_n_sporadic.json.bak
python src/scheduler.py
```
**執行結果**：
- 系統會依序跑完 10 種情境，並在 `output/` 目錄下產生各情境的檔案：`schedule_result_level2_<情境>.json`、`acceptance_test_log_level2_<情境>.json` 與 `evaluation_results_level2_<情境>.json`（例如 `schedule_result_level2_scenario_01_uniform.json`；Level 1 不帶 `level2_` 前綴）。
- 接著會對所有情境執行 evaluator（`src/evaluator.py --batch-level2`，Level 1 為 `--batch-scenarios`），跨情境總表為 `output/evaluation_results_level2_summary.json`（Level 1：`output/evaluation_results_summary.json`）。
- 為了相容批改系統，程式會將第一組情境 (`scenario_01_uniform`) 的結果複製成標準繳交檔名 `schedule_result.json`、`acceptance_test_log.json` 與 `evaluation_results.json`。因此 `evaluation_results.json` **只涵蓋 scenario_01**；所有情境的結果請看總表檔案。

要回到 Demo 模式時，把檔案移回來即可：`mv input/aperiodic_n_sporadic.json.bak input/aperiodic_n_sporadic.json`。

### 4. Demo 測試
在 Demo 時：
1. 將測試用的突發任務 (sporadic/aperiodic) 檔案命名為 `aperiodic_n_sporadic.json`（取代附帶的範例檔）。
2. 將該檔案放入專案的 `input/` 目錄中 (`input/aperiodic_n_sporadic.json`)。
3. 執行程式：
```bash
python src/scheduler.py
```
**執行結果**：
程式會印出 `[*] 偵測到 Demo 專用測資: input/aperiodic_n_sporadic.json` 的提示，並 **直接跳過批次情境**，單獨針對該 Demo 檔案進行模擬。產生的輸出檔案會 **直接覆蓋** 標準檔名：
- `output/schedule_result.json`
- `output/acceptance_test_log.json`
- `output/evaluation_results.json`

repository 內這三個檔案目前的內容，來自以附帶範例檔執行的 Demo 模式。

### 5. 執行測試
```bash
pip install pytest
python -m pytest
```
請在 repository 根目錄以 `python -m pytest` 執行，才能 import `src` 套件（直接執行 `pytest` 會出現 `ModuleNotFoundError: No module named 'src'`）。這會執行 `tests/test_acceptance_tester.py` 與 `tests/test_state_machine.py` 中的 20 個測試。`tests/test_offline_pipeline.py` 沒有 pytest 測試函式，是日前排程器的獨立整合測試腳本，請用 `python -m tests.test_offline_pipeline` 執行。

### 6. 重新產生輸入資料（選用）
- `python src/task_generator.py` 會重建 `output/task_set.json`（週期性任務集，設計說明見下方）。由於種子固定，產出與 repository 內的檔案完全相同。
- `python src/aperiodic_n_sporadic_gen.py`（需要 NumPy）會覆寫 `output/sporadic_aperiodic_task/` 內的 10 個情境檔（3 組 uniform、3 組 concentrated、4 組 mixed；種子 2027–2036）。repository 內的 `scenario_01_uniform.json` 與生成器目前的產出不同，重新執行會改變該情境。

---

## 📂 檔案結構

```text
Embedded-RTOS-Scheduler/
├── README.md                        # 英文說明文件
├── README.zh-TW.md                  # 本說明文件（繁體中文）
├── LICENSE                          # MIT License
├── report.pdf                       # 系統設計報告
├── runtime_config.json              # Level 2 參數的參考副本（程式不會讀取，見下方說明）
├── docs/
│   └── level2_model.md              # Level 2 形式化模型（限制式與虛擬碼）
├── src/
│   ├── scheduler.py                 # 系統進入點：Offline / Online 排程、Level 1/2 切換、批次/Demo 模式
│   ├── evaluator.py                 # 評估與計分（單次、--batch-scenarios 或 --batch-level2）
│   ├── advanced_scheduler.py        # Level 2 專用：綠電預測誤差、電池模型、市場承諾售電、aperiodic 延後
│   ├── task_generator.py            # 週期性 Task Set 生成器 -> output/task_set.json
│   ├── aperiodic_n_sporadic_gen.py  # Sporadic/aperiodic 情境生成器 (NumPy) -> output/sporadic_aperiodic_task/
│   └── engine/                      # 核心排程引擎
│       ├── offline_planner.py       # 以 Frame 為單位的日前 DFS 排程器
│       ├── acceptance_tester.py     # Sporadic / aperiodic 線上准入控制
│       ├── main_scheduler.py        # 72 小時線上模擬與分派迴圈
│       ├── power_tracer.py          # 注水式能量溯源（k 矩陣）
│       ├── state_machine.py         # 發電機 / 電池狀態轉移驗證
│       ├── data_loader.py           # JSON 輸入載入
│       └── models.py                # 資料類別
├── tests/
│   ├── test_acceptance_tester.py    # pytest
│   ├── test_state_machine.py        # pytest
│   └── test_offline_pipeline.py     # 日前排程器獨立整合測試腳本
├── input/
│   ├── processor_settings.json      # 發電機、再生能源（容量 + 預測）與儲能設定
│   ├── price_72hr.json              # 72 小時市場電價
│   └── aperiodic_n_sporadic.json    # Demo 範例檔（此檔存在即進入 Demo 模式）
└── output/
    ├── task_set.json                # 週期性任務集（由 task_generator.py 產生）
    ├── sporadic_aperiodic_task/     # 批次模式的 10 組情境
    └── ...                          # 排程結果、准入紀錄、評估結果與總表
```

`runtime_config.json` 不會被任何程式讀取。其數值與 `src/scheduler.py` 中的 `LEVEL2_CONFIG` 相同，排程器實際使用的是 `LEVEL2_CONFIG`；要調整 Level 2 參數請修改 `LEVEL2_CONFIG`。

---

## 📦 Periodic Task Set 生成策略與額外設計說明

`src/task_generator.py` 在滿足作業基礎規範外，另做了以下設計。生成器抽出 `N = 6–10` 個任務；repository 內的 `output/task_set.json`（種子 2026）為 `N = 6`、`DW ≈ 0.99`，72 小時內共 44 個 Job。組員與評分時可參考此邏輯理解輸入資料特性：

| 額外設計 / 限制 | 設計動機與對排程器的影響 |
|:---|:---|
| **Non-preemptive 綁定於 `e=2` 而非 `e=3`** | 執行時間固定為 `[1]*(N-4) + [2, 2, 3, 3]`。兩個 `e=2` 的任務設為不可中斷；兩個 `e=3`（且 `d=3`）的任務維持可中斷。若 `e=3, d=3` 不可中斷，就是零鬆弛：必須剛好占用一段連續 3 小時，容易造成排程碎片化，並讓日前 DFS 排程器需要更多回溯。改為 `e=2` 可在滿足 `e≠1` 規範的前提下，降低連續時段分配難度。 |
| **固定比例 `p=6` 短週期任務** | 共 `(N-2)//2` 個任務使用 `p=6`。短週期提供穩定的基底負載，便於計算跨 Frame 的 Slack 餘裕，並降低儲能 SOC 劇烈波動的風險。生成器本身不以特定 Workload Density (DW) 為目標；`validate()` 只接受 `0.7 ≤ DW ≤ 1.0`，排除負載過輕或過載 (DW > 1.0) 的組合。 |
| **混合週期設計 (`6`, `11/12`, `15/18/21/24`) 與防禦性 Deadline 下限** | 除了 `p=6` 的任務，前 `N-2` 個任務中其餘的任務共用一個從 `{11, 12}` 抽出的週期，最後兩個任務則從 `{15, 18, 21, 24}` 抽出兩個不同的週期。未強制所有 period 為 3 的倍數，保留 `11/12` 以貼近真實非對齊週期情境。為確保 `f=3` 的 Frame 可視性 (`2f−gcd(f,p)≤d`)，前 `N-2` 個任務的 `d` 從 `[6, p]` 抽取。此設計恆滿足限制式（例：`p=11` 時 `2×3−gcd(3,11)=5 ≤ 6`；最後兩個任務 `d=3` 且 `p` 為 3 的倍數，`2×3−3=3 ≤ 3`），避免寫死參數，同時測試排程器處理非對齊釋放點的能力。 |
| **Deadline 分層策略：前段寬鬆(≥6) / 末段緊迫(d=3)** | 最後兩個任務固定 `d=3`（`=e`），以滿足 1-6 (`≥20% d=e`；`N ≤ 10` 時 `2/N ≥ 20%`) 與 `f=3` 的邊界條件；前段保留寬鬆 Deadline 則提供排程器進行能量平移（儲能充放）與機組 Ramp 調整的彈性空間，避免所有 Job 競爭同一時間窗，利於優化 `f2` (發電成本) 與 `f3` (售電收益)。 |
| **確定性種子 + 驗證** | 固定 `RANDOM_SEED=2026`，每次執行都產生相同的任務集。`validate()` 回傳 `(ok, message)`，檢查項目為：`f=3` 的 Frame 可視性 `2f−gcd(f,p)≤d`、`0.7 ≤ DW ≤ 1.0`、72 小時內 Job 數大於 30、至少 3 種不同週期、不可中斷任務 `d ≥ e`。生成器只抽 **一次**，不會重抽：`main()` 印出訊息，只有驗證通過時才寫入 `output/task_set.json`，失敗時不儲存任何檔案。種子 2026 的結果會通過驗證 (`OK`)，方便組員除錯、CI 驗證與 Demo 重現。 |

---

## ⚙️ 核心調度引擎設計 (Scheduling Engine Architecture)

本虛擬電廠 (VPP) 的排程核心採用「雙層調度架構 (Two-Tier Scheduling Architecture)」，以確保在硬即時 (Hard Real-Time) 約束下達成系統存活與經濟效益最大化。在 Level 2 版本更導入了進階動態重排程來應對市場與氣候的不確定性。整體引擎分為四大階段：

### 1. 日前離線排程 (Offline Pre-scheduling)
* **負責模組：** `src/engine/offline_planner.py`
* **處理對象：** 週期性任務 (Periodic Tasks)，發布時間與週期提前已知。
* **演算法 (Greedy DFS)：**
  對連續 24 個 **3 小時 Frame**（共 72 小時）進行深度優先搜尋 (DFS)。
  1. 針對當前 Frame 內的活躍 Job（在 Frame 結束前已釋放、尚未完成、Deadline 在 Frame 開始之後），列舉這 3 個小時的時段分配組合。
  2. 候選排法依「長度最長（且最早）優先」嘗試，即「把時間排滿」的激進貪婪策略 (Aggressive Greedy)，讓任務盡早完成。若某排法會讓剩餘工作量超過 Deadline 前剩下的小時數，則直接剪枝。
  3. 每個組合逐小時驗證：先全額吸收綠電預測量，再由傳統火力機組（在出力上下限、升降載速率與最短開/關機時間限制內）與電池放電（在放電上限與 SOC 限制內）補足負載。驗證通過則推進至下一 Frame；若該 Frame 失敗或後續 Frame 無法完成，就從快照還原發電機/電池狀態並嘗試下一個組合（**回溯 Backtrack**）。
* **產出：** 一份 72 小時基載排程表，與每小時的「剩餘算力 (Slack Capacity)」（火力與電池剩餘可用容量），作為後續線上准入控制的基底。

### 2. 線上准入控制 (Online Admission Control)
* **負責模組：** `src/engine/acceptance_tester.py`
* **處理對象：** 突發任務 (Sporadic Tasks) 與 非週期任務 (Aperiodic Tasks)。
* **運作邏輯：**
  這是一個動態發生的過程。當系統運行到第 `t` 小時，新任務抵達時：
  1. **Sporadic 任務 (硬限制)：** 系統檢查 Offline Planner 留下的 `slack_capacity`，在該任務 Deadline 之前是否有足夠的小時數（不可中斷任務需為連續時段）其剩餘算力 ≥ 任務需求。若有則預留；否則為了避免拖垮全系統，會執行嚴格的 **直接拒絕 (Reject)**。
  2. **Aperiodic 任務 (軟限制)：** 抵達時若剩餘算力中完全放不下就直接拒絕，否則放入等候佇列 (Queue)。系統每小時依序掃描佇列：放得下的任務就執行（可中斷任務每次執行 1 小時，不可中斷任務一次排完整段），放不下的任務先跳過，讓後面的任務仍可執行（Backfilling，解決隊頭阻塞 Head-of-Line Blocking）。佇列中的任務若等待超過 24 小時（Timeout Drop），或剩餘執行時間超過 72 小時內剩下的時數，就會被丟棄。

### 3. 線上即時分派與能量溯源 (Real-time Dispatch & Tracing)
* **負責模組：** `src/engine/main_scheduler.py` 與 `src/engine/power_tracer.py`
* **運作邏輯：**
  每小時盤點當下所有應執行的任務（包含 Offline 與 Online 准入的任務），計算「總用電需求」。
  1. **Must-take 優先：** 優先全額吸收邊際成本為 0 的再生能源，算出「淨負載 (Net Load)」。
  2. **保命防禦：** 以各機組的最低合法出力為起點，依照機組變動成本 (`cost_variable`) 升冪提高發電機出力以補足淨負載。若火力極限仍不足，則以電池放電作為最後防線。
  3. **高價套利：** 若當下市場電價高於已開機火力機組的變動成本，系統主動將該機組推升至升降載限制允許的最高出力，將剩餘電力賣給電網以極大化售電收益。
  4. **注水溯源：** `power_tracer.py` 透過注水演算法，對應每個供電來源（綠電、發電機、電池）與個別任務的供需流向，產出 $k_{j,i,t}$ 矩陣。任何任務供電不足都會拋出錯誤，剩餘電力則記為售電量，因此每小時都滿足「供給 = 用電 + 售電」。

### 4. Level 2 進階動態重排程 (Advanced Dynamic Rescheduling)
* **負責模組：** `src/advanced_scheduler.py` (僅在 `LEVEL2_ENABLED = True` 時啟動；參數位於 `src/scheduler.py` 的 `LEVEL2_CONFIG`)
* **運作邏輯：**
  打破理想狀態，引入電力市場承諾售電與氣候隨機性：
  1. **綠電不確定性：** 每小時的實際綠電出力 = 預測值乘上一個在 ±20% 內均勻抽樣的相對誤差 (`forecast_error_ratio`，種子 `random_seed = 2026`)，並限制在機組容量以內。
  2. **市場承諾與罰款：** 系統每小時承諾售出固定電量（`5.0 × commitment_ratio = 4.0` MWh）。若實際售電不足，缺口依 `penalty_rate` (每 MWh) 計罰；超出承諾的售電則以 `(realtime_price_multiplier − 1) × 電價` 取得額外收益。
  3. **真實電池模型：** 電池充放電考慮效率、自放電與 SOC 相依的放電上限；每放電 1 MWh 會產生老化成本，並計入 Level 2 調整後目標值。多餘的電力會透過 `<電池>_chg` 虛擬任務為電池充電。
  4. **動態救援策略：** 當實際綠電低於預測時，系統會記錄電池放電與火力升載餘裕可補足多少缺口（建議性估算）；接著由上述每小時分派流程依成本順序拉高火力機組並以電池放電補足淨負載。若估算餘裕仍不足以補足缺口，會將該小時正在執行的 aperiodic job（ID 以 `a_` 開頭，即 `aperiodic_n_sporadic_gen.py` 產生的格式）延後並踢回等候佇列 (Defer)，優先保證週期性與 sporadic（硬截止期）任務的供電。
