# NCKU-RTOS-2026 Virtual Power Plant (VPP) Dynamic Scheduling System

![Python](https://img.shields.io/badge/language-Python-3776AB?logo=python&logoColor=white)
![pytest](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)
![Scheduling](https://img.shields.io/badge/scheduling-offline%20DFS%20%2B%20online%20admission-555)
![License: MIT](https://img.shields.io/badge/license-MIT-yellow)

**English** | [繁體中文](README.zh-TW.md)

## 🚀 Quick Start & Execution Guide

The project supports both **automated batch analysis of 10 scenarios** and **single-file Demo testing**, and can be switched between Level 1 (baseline requirements) and Level 2 (advanced dynamic scheduling).

### 0. Requirements
- Python 3 (the commands below were checked with Python 3.13). Run them from the repository root.
- The scheduler (`src/scheduler.py`), the evaluator (`src/evaluator.py`) and the periodic task-set generator (`src/task_generator.py`) use only the Python standard library.
- The scenario generator `src/aperiodic_n_sporadic_gen.py` needs NumPy: `pip install numpy`.
- The tests need pytest: `pip install pytest`.

### 1. Switching Between Level 1 and Level 2
The project runs in **Level 2 advanced dynamic scheduling mode** by default (`LEVEL2_ENABLED = True`).
To test Level 1, go to line 40 of `src/scheduler.py` and set the `LEVEL2_ENABLED` variable to `False`:
```python
# src/scheduler.py
LEVEL2_ENABLED = False  # False runs Level 1, True runs Level 2
```
Re-run the program after switching; the simulation then uses the renewable forecast as-is and the Level 1 battery model, with no market commitment. In Level 1 the output file names have no `level2_` prefix.

### 2. Batch Mode vs. Demo Mode
`src/scheduler.py` chooses the mode with a single check: if `input/aperiodic_n_sporadic.json` exists it runs **Demo mode**, otherwise **Batch mode**.
The repository ships with a sample `input/aperiodic_n_sporadic.json`, so on a fresh clone `python src/scheduler.py` runs **Demo mode**.

### 3. Reproducing the 10-Scenario Analysis (Batch Mode)
Batch mode simulates the 10 `scenario_*.json` files under `output/sporadic_aperiodic_task/`. Move or rename the Demo file first, then run:
```bash
mv input/aperiodic_n_sporadic.json input/aperiodic_n_sporadic.json.bak
python src/scheduler.py
```
**Results**:
- The 10 scenarios run in sequence, and per-scenario files are written to `output/`: `schedule_result_level2_<scenario>.json`, `acceptance_test_log_level2_<scenario>.json` and `evaluation_results_level2_<scenario>.json` (e.g. `schedule_result_level2_scenario_01_uniform.json`; no `level2_` prefix in Level 1).
- The evaluator is then run over all scenarios (`src/evaluator.py --batch-level2`, or `--batch-scenarios` in Level 1). The cross-scenario summary is `output/evaluation_results_level2_summary.json` (Level 1: `output/evaluation_results_summary.json`).
- For compatibility with the grading system, the results of the first scenario (`scenario_01_uniform`) are copied to the standard submission file names `schedule_result.json`, `acceptance_test_log.json` and `evaluation_results.json`. `evaluation_results.json` therefore covers **scenario_01 only**; use the summary file for all scenarios.

To return to Demo mode, move the file back: `mv input/aperiodic_n_sporadic.json.bak input/aperiodic_n_sporadic.json`.

### 4. Demo Testing
For the Demo:
1. Name the test file of burst (sporadic/aperiodic) tasks `aperiodic_n_sporadic.json` (it replaces the bundled sample).
2. Place it in the project's `input/` directory (`input/aperiodic_n_sporadic.json`).
3. Run the program:
```bash
python src/scheduler.py
```
**Results**:
The program prints `[*] 偵測到 Demo 專用測資: input/aperiodic_n_sporadic.json` ("Demo test data detected"), **skips the batch scenarios entirely**, and simulates only that Demo file. The generated output files **overwrite** the standard file names directly:
- `output/schedule_result.json`
- `output/acceptance_test_log.json`
- `output/evaluation_results.json`

The copies of these three files committed in the repository come from a Demo-mode run on the bundled sample file.

### 5. Running the Tests
```bash
pip install pytest
python -m pytest
```
Use `python -m pytest` from the repository root so that the `src` package is importable (a bare `pytest` fails with `ModuleNotFoundError: No module named 'src'`). This runs 20 tests in `tests/test_acceptance_tester.py` and `tests/test_state_machine.py`. `tests/test_offline_pipeline.py` has no pytest test functions; it is a standalone integration script for the offline planner, run with `python -m tests.test_offline_pipeline`.

### 6. Regenerating the Inputs (Optional)
- `python src/task_generator.py` rebuilds `output/task_set.json` (the periodic task set; see the design notes below). With the fixed seed it reproduces the committed file exactly.
- `python src/aperiodic_n_sporadic_gen.py` (needs NumPy) rewrites the 10 scenario files in `output/sporadic_aperiodic_task/` (3 uniform, 3 concentrated, 4 mixed; seeds 2027–2036). The committed `scenario_01_uniform.json` differs from what the generator currently produces, so re-running it changes that scenario.

---

## 📂 Project Structure

```text
Embedded-RTOS-Scheduler/
├── README.md                        # This document (English)
├── README.zh-TW.md                  # Traditional Chinese version
├── LICENSE                          # MIT License
├── report.pdf                       # System design report
├── runtime_config.json              # Reference copy of the Level 2 parameters (not read by any code, see below)
├── docs/
│   └── level2_model.md              # Level 2 formal model (constraints and pseudo-code)
├── src/
│   ├── scheduler.py                 # Entry point: Offline / Online scheduling, Level 1/2 switch, Batch/Demo mode
│   ├── evaluator.py                 # Evaluation and scoring (single run, --batch-scenarios or --batch-level2)
│   ├── advanced_scheduler.py        # Level 2 only: renewable forecast error, battery model, market commitment, aperiodic defer
│   ├── task_generator.py            # Periodic task-set generator -> output/task_set.json
│   ├── aperiodic_n_sporadic_gen.py  # Sporadic/aperiodic scenario generator (NumPy) -> output/sporadic_aperiodic_task/
│   └── engine/                      # Core scheduling engine
│       ├── offline_planner.py       # Offline frame-based DFS planner
│       ├── acceptance_tester.py     # Online admission control for sporadic / aperiodic tasks
│       ├── main_scheduler.py        # 72-hour online simulation and dispatch loop
│       ├── power_tracer.py          # Water-filling power tracing (k matrix)
│       ├── state_machine.py         # Generator / battery transition validation
│       ├── data_loader.py           # JSON input loaders
│       └── models.py                # Data classes
├── tests/
│   ├── test_acceptance_tester.py    # pytest
│   ├── test_state_machine.py        # pytest
│   └── test_offline_pipeline.py     # Standalone offline-planner integration script
├── input/
│   ├── processor_settings.json      # Generator, renewable (capacity + forecast) and storage settings
│   ├── price_72hr.json              # 72-hour market price
│   └── aperiodic_n_sporadic.json    # Sample Demo file (its presence selects Demo mode)
└── output/
    ├── task_set.json                # Periodic task set (from task_generator.py)
    ├── sporadic_aperiodic_task/     # The 10 batch scenarios
    └── ...                          # Schedules, acceptance logs, evaluation results and summaries
```

`runtime_config.json` is not read by any code. Its values duplicate `LEVEL2_CONFIG` in `src/scheduler.py`, which is what the scheduler actually uses; edit `LEVEL2_CONFIG` to change the Level 2 parameters.

---

## 📦 Periodic Task Set Generation Strategy and Additional Design Notes

Beyond meeting the assignment's baseline specification, `src/task_generator.py` makes the following design decisions. It draws `N = 6–10` tasks; the committed `output/task_set.json` (seed 2026) has `N = 6`, `DW ≈ 0.99` and 44 jobs over 72 hours. Teammates and graders can refer to this logic to understand the characteristics of the input data:

| Additional design / constraint | Rationale and impact on the scheduler |
|:---|:---|
| **Non-preemptive bound to `e=2` instead of `e=3`** | Execution times are fixed as `[1]*(N-4) + [2, 2, 3, 3]`. The two `e=2` tasks are non-preemptive; the two `e=3` tasks (with `d=3`) stay preemptive. A non-preemptive `e=3, d=3` task would have zero slack: it would need one exact contiguous 3-hour block, which fragments the schedule and forces more backtracking in the offline DFS planner. Using `e=2` still satisfies the `e≠1` requirement while making contiguous-slot allocation easier. |
| **Fixed share of short-period `p=6` tasks** | `(N-2)//2` tasks use `p=6`. The short periods provide a stable base load, make cross-Frame slack easier to compute, and reduce the risk of sharp swings in storage SOC. The generator does not target a specific Workload Density (DW); `validate()` accepts only `0.7 ≤ DW ≤ 1.0`, rejecting sets that are too light or over-utilized (DW > 1.0). |
| **Mixed periods (`6`, `11/12`, `15/18/21/24`) with a defensive lower bound on deadlines** | Besides the `p=6` tasks, the rest of the first `N-2` tasks share one period drawn from `{11, 12}`, and the last two tasks take two distinct periods from `{15, 18, 21, 24}`. Periods are not all forced to be multiples of 3; `11/12` is kept to reflect realistic, unaligned periods. To guarantee Frame visibility for `f=3` (`2f−gcd(f,p)≤d`), the first `N-2` tasks draw `d` from `[6, p]`. This always satisfies the constraint (e.g. for `p=11`, `2×3−gcd(3,11)=5 ≤ 6`; the last two tasks have `d=3` and `p` a multiple of 3, so `2×3−3=3 ≤ 3`) without hard-coding parameters, and also tests the scheduler's handling of unaligned release points. |
| **Tiered deadlines: relaxed early tasks (≥6) / tight final tasks (d=3)** | Fixing `d=3` (`=e`) for the final two tasks satisfies requirement 1-6 (`≥20% d=e`, since `2/N ≥ 20%` for `N ≤ 10`) and the `f=3` boundary condition. Keeping relaxed deadlines for the earlier tasks gives the scheduler room to shift energy (storage charge/discharge) and adjust generator ramping, and prevents every Job from competing for the same time window, which helps optimize `f2` (generation cost) and `f3` (electricity sales revenue). |
| **Deterministic seed + validation** | `RANDOM_SEED=2026` is fixed, so every run produces the same task set. `validate()` returns `(ok, message)` and checks: Frame visibility `2f−gcd(f,p)≤d` for `f=3`, `0.7 ≤ DW ≤ 1.0`, more than 30 Jobs in 72 hours, at least 3 distinct periods, and `d ≥ e` for non-preemptive tasks. The generator draws **once** and does not redraw: `main()` prints the message and writes `output/task_set.json` only if validation passes; on failure nothing is saved. With seed 2026 the draw passes (`OK`), which keeps debugging, CI checks and Demo runs reproducible. |

---

## ⚙️ Scheduling Engine Architecture

The scheduling core of this Virtual Power Plant (VPP) uses a **Two-Tier Scheduling Architecture** to keep the system alive under Hard Real-Time constraints while maximizing economic return. The Level 2 version adds advanced dynamic rescheduling to cope with market and weather uncertainty. The engine is organized into four stages:

### 1. Offline Pre-scheduling (Day-Ahead)
* **Module:** `src/engine/offline_planner.py`
* **Handles:** Periodic Tasks, whose release times and periods are known in advance.
* **Algorithm (Greedy DFS):**
  A depth-first search (DFS) over 24 consecutive **3-hour Frames** (72 hours).
  1. For the jobs active in the current Frame (released before the Frame ends, not yet finished, deadline after the Frame starts), the system enumerates slot-assignment combinations for the Frame's three hours.
  2. Candidate patterns are tried longest (and earliest) first, an Aggressive Greedy order that "fills up the time slots" so tasks finish as early as possible. Patterns that would leave more remaining work than hours left before the job's deadline are pruned.
  3. Each combination is checked hour by hour: renewable forecast is absorbed first, then the conventional thermal units (within their output, ramp-rate and minimum up/down-time limits) and battery discharge (within its discharge and SOC limits) must cover the load. If it passes, the search advances to the next Frame; if the Frame fails or no later Frame can be completed, the generator/battery state is restored from a snapshot and the next combination is tried (**backtracking**).
* **Output:** A 72-hour base-load schedule plus the hourly **Slack Capacity** (remaining thermal and battery headroom), which serves as the foundation for the online admission control that follows.

### 2. Online Admission Control
* **Module:** `src/engine/acceptance_tester.py`
* **Handles:** Sporadic Tasks and Aperiodic Tasks.
* **How it works:**
  This is a dynamic process. When the system reaches hour `t` and a new task arrives:
  1. **Sporadic tasks (hard constraint):** The system checks the `slack_capacity` left by the Offline Planner for enough hours (contiguous hours for non-preemptive tasks) with slack ≥ the task's demand before its Deadline. If they exist they are reserved; otherwise the task is strictly **rejected (Reject)** so it cannot drag down the whole system.
  2. **Aperiodic tasks (soft constraint):** On arrival, a task that cannot fit anywhere in the remaining slack is rejected; otherwise it joins a waiting Queue. Every hour the queue is scanned in order: tasks that fit run (preemptive tasks one hour at a time, non-preemptive tasks as a whole block), and tasks that do not fit are skipped so later tasks can still run (backfilling, which avoids Head-of-Line Blocking). A queued task is dropped when it has waited more than 24 hours (Timeout Drop) or when its remaining execution time exceeds the hours left in the 72-hour horizon.

### 3. Real-time Dispatch & Tracing
* **Modules:** `src/engine/main_scheduler.py` and `src/engine/power_tracer.py`
* **How it works:**
  Every hour, the system collects all tasks that should run now (both Offline tasks and Online tasks that passed admission) and computes the total power demand.
  1. **Must-take first:** Renewable energy, whose marginal cost is 0, is absorbed in full first, giving the Net Load.
  2. **Survival defense:** Starting from each generator's minimum legal output, generators are raised in ascending order of variable cost (`cost_variable`) to cover the Net Load. If thermal capacity is still insufficient at its limit, battery discharge is used as the last line of defense.
  3. **High-price arbitrage:** When the current market price is higher than the variable cost of a thermal unit that is already online, the system pushes that unit to the highest output its ramp limits allow and sells the surplus to the grid to maximize sales revenue.
  4. **Water-filling tracing:** `power_tracer.py` uses a water-filling algorithm to map the flow from each supply source (renewable, generator, battery) to each individual task, producing the $k_{j,i,t}$ matrix. Any task left short raises an error, and leftover supply is recorded as sold energy, so supply = consumption + sales holds every hour.

### 4. Level 2 Advanced Dynamic Rescheduling
* **Module:** `src/advanced_scheduler.py` (active only when `LEVEL2_ENABLED = True`; parameters in `LEVEL2_CONFIG` in `src/scheduler.py`)
* **How it works:**
  Moves beyond the idealized setting by introducing an electricity-market commitment and weather randomness:
  1. **Renewable uncertainty:** Each hour's actual renewable output is the forecast scaled by a random relative error drawn uniformly from ±20% (`forecast_error_ratio`, seed `random_seed = 2026`), clipped to the unit's capacity.
  2. **Market commitment and penalty:** The system commits to sell a fixed amount every hour (`5.0 × commitment_ratio = 4.0` MWh). If actual sales fall short, the shortfall is penalized at `penalty_rate` per MWh; sales above the commitment earn extra revenue at `(realtime_price_multiplier − 1) × price`.
  3. **Realistic battery model:** Battery discharge and charging use efficiencies, self-discharge, and an SOC-dependent discharge limit; every discharged MWh adds a degradation cost that is included in the Level 2 adjusted objective. Surplus supply charges the batteries through a `<battery>_chg` pseudo-job.
  4. **Dynamic rescue strategy:** When actual renewable output falls below the forecast, the system logs how much of the gap battery discharge and thermal ramp-up headroom could cover (advisory estimates); the hourly dispatch above then covers the Net Load with thermal units in cost order and battery discharge. If the estimated headroom still cannot cover the gap, aperiodic jobs running in that hour (IDs starting with `a_`, as produced by `aperiodic_n_sporadic_gen.py`) are deferred back to the waiting queue (Defer), so that periodic and sporadic (hard-deadline) jobs keep their power.
