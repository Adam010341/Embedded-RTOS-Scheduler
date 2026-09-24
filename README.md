# NCKU-RTOS-2026 Virtual Power Plant (VPP) Dynamic Scheduling System

![Python](https://img.shields.io/badge/language-Python-3776AB?logo=python&logoColor=white)
![pytest](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)
![Scheduling](https://img.shields.io/badge/scheduling-offline%20DFS%20%2B%20online%20admission-555)
![License: MIT](https://img.shields.io/badge/license-MIT-yellow)

**English** | [繁體中文](README.zh-TW.md)

## 🚀 Quick Start & Execution Guide

The project supports both **automated batch analysis of 10 scenarios** and **single-file Demo testing**, and can be switched at any time between Level 1 (baseline requirements) and Level 2 (advanced dynamic scheduling).

### 1. Switching Between Level 1 and Level 2
The project runs in **Level 2 advanced dynamic scheduling mode** by default.
To test Level 1, go to around line 40 of `src/scheduler.py` and set the `LEVEL2_ENABLED` variable to `False`:
```python
# src/scheduler.py
LEVEL2_ENABLED = False  # False runs Level 1, True runs Level 2
```
Re-run the program after switching; all resource constraints and scheduling logic fall back to what Level 1 requires.

### 2. Reproducing the 10-Scenario Analysis (Batch Analysis)
During normal development and report writing, the system automatically loads the 10 `scenario_*.json` files under `output/sporadic_aperiodic_task/` by default.
Make sure the `input/` folder does **not** contain a file named `aperiodic_n_sporadic.json`, then run:
```bash
python src/scheduler.py
```
**Results**:
- The system runs the 10 scenarios in sequence and writes scenario-suffixed files to `output/` (e.g. `schedule_result_level2_scenario_01_uniform.json`).
- For compatibility with the grading system, the program automatically copies the results of the first scenario (`scenario_01`) to the standard submission file names (`schedule_result.json`, etc.).
- It also invokes the evaluator for an overall evaluation, producing an `evaluation_results.json` that covers all scenarios.

### 3. Demo Testing
For the Demo:
1. Name the test file of burst (sporadic/aperiodic) tasks `aperiodic_n_sporadic.json`.
2. Place it in the project's `input/` directory (`input/aperiodic_n_sporadic.json`).
3. Run the program:
```bash
python src/scheduler.py
```
**Results**:
The program prints `[*] 偵測到 Demo 專用測資` ("Demo test data detected"), **skips the batch scenarios entirely**, and simulates only that Demo file. The generated output files **overwrite** the standard file names directly:
- `output/schedule_result.json`
- `output/acceptance_test_log.json`
- `output/evaluation_results.json`

---

## 📂 Project Structure

```text
NCKU-RTOS-2026-level2/
├── README.md                  # This document
├── report.pdf                 # System design report
├── src/
│   ├── scheduler.py           # System entry point (Offline / Online scheduling and mode-switching logic)
│   ├── evaluator.py           # Evaluation and scoring module
│   ├── engine/                # Core scheduling engine (Planner, Tester, Tracer, etc.)
│   ├── advanced_scheduler.py  # Level 2 only: handles market-commitment defaults and renewable forecast error
│   └── task_generator.py      # Task Set generator
├── input/                     # Input data; drop-in location for Demo test data
├── output/                    # Simulation results and Evaluator output
└── runtime_config.json        # Additional Level 2 configuration file
```

---

## 📦 Periodic Task Set Generation Strategy and Additional Design Notes

Beyond meeting the assignment's baseline specification, this module adds the following engineering decisions so that the generated Task Set is both mathematically schedulable and stable for the `Demo`. Teammates and graders can refer to this logic to understand the characteristics of the input data:

| Additional design / constraint | Rationale and impact on the scheduler |
|:---|:---|
| **Non-preemptive bound to `e=2` instead of `e=3`** | The original plan made the longest-running tasks non-preemptive, but `e=3, d=3` creates a zero-slack contiguous block that easily fragments the schedule and causes MILP solver oscillation. Switching to `e=2` still satisfies the `e≠1` requirement while making contiguous-slot allocation much easier, which speeds up Scheduler convergence and raises the insertion success rate of the Acceptance Test. |
| **Fixed share of short-period `p=6` tasks** | Used to pin the Workload Density (DW) precisely within `0.75~0.95`. This avoids a DW so low that scheduling is trivial, or a DW > 1.0 that is theoretically overloaded. The short periods provide a stable base load, make cross-Frame slack easier to compute, and reduce the risk of sharp swings in storage SOC. |
| **Mixed periods (`6`, `11/12`, `15~24`) with a defensive lower bound on deadlines** | Periods are not all forced to be multiples of 3; `11/12` is kept to reflect realistic, unaligned periods. To guarantee Frame visibility for `f=3` (`2f−gcd(f,p)≤d`), the first `N-2` tasks automatically get a `d≥6` lower bound. This always satisfies the constraint mathematically (e.g. for `p=11`, `2×3−gcd(3,11)=5 ≤ 6`) without hard-coding parameters, and also tests the scheduler's handling of unaligned release points. |
| **Tiered deadlines: relaxed early tasks (≥6) / tight final tasks (d=3)** | Fixing `d=3` for the final tasks reliably satisfies requirement 1-6 (`≥20% d=e`) and the `f=3` boundary condition. Keeping relaxed deadlines for the earlier tasks gives the scheduler room to shift energy (storage charge/discharge) and adjust generator ramping, and prevents every Job from competing for the same time window, which helps optimize `f2` (generation cost) and `f3` (electricity sales revenue). |
| **Deterministic seed + automatic fallback validation** | `RANDOM_SEED=2026` is fixed and a built-in `validate()` assertion is applied. If a random draw exceeds the DW limit, expands into too few Jobs, or fails Frame visibility, it is automatically redrawn. Every run therefore produces a deterministic input that is valid and schedulable, which simplifies debugging for teammates, CI validation, and Demo reproduction. |

---

## ⚙️ Scheduling Engine Architecture

The scheduling core of this Virtual Power Plant (VPP) uses a **Two-Tier Scheduling Architecture** to keep the system alive under Hard Real-Time constraints while maximizing economic return. The Level 2 version adds advanced dynamic rescheduling to cope with market and weather uncertainty. The engine is organized into four stages:

### 1. Offline Pre-scheduling (Day-Ahead)
* **Module:** `src/engine/offline_planner.py`
* **Handles:** Periodic Tasks, whose release times and periods are known in advance.
* **Algorithm (Greedy DFS):**
  A depth-first search (DFS) that uses **a 3-hour search window (Frame)**.
  1. For the tasks due within the current 3 hours, the system generates every possible scheduling combination.
  2. It first tries an Aggressive Greedy strategy that "fills up the time slots", so tasks finish as early as possible.
  3. Each combination is checked against the ramp rate and start/stop limits of the conventional thermal units and the battery capacity limits. If it passes, the search advances to the next Frame; if a future Deadline Miss is foreseen, an **implicit Backtrack** is triggered.
* **Output:** A stable 72-hour base-load schedule plus the hourly global **Slack Capacity**, which serves as the foundation for the online defenses that follow.

### 2. Online Admission Control
* **Module:** `src/engine/acceptance_tester.py`
* **Handles:** Sporadic Tasks and Aperiodic Tasks.
* **How it works:**
  This is a dynamic process. When the system reaches hour `t` and a new task arrives:
  1. **Sporadic tasks (hard constraint):** The system checks the `slack_capacity` left by the Offline Planner to decide whether the task's energy demand can be absorbed before its Deadline. If capacity is insufficient, the task is strictly **rejected (Reject)** so it cannot drag down the whole system.
  2. **Aperiodic tasks (soft constraint):** The task is placed in a waiting Queue, which the system drains in order whenever it has spare capacity. A task is dropped on timeout (Timeout Drop) only when it has waited so long that it can no longer physically complete, which avoids Head-of-Line Blocking.

### 3. Real-time Dispatch & Tracing
* **Modules:** `src/engine/main_scheduler.py` and `src/engine/power_tracer.py`
* **How it works:**
  Every hour, the system collects all tasks that should run now (both Offline tasks and Online tasks that passed admission) and computes the total power demand.
  1. **Must-take first:** Renewable energy, whose marginal cost is 0, is absorbed in full first, giving the Net Load.
  2. **Survival defense:** Generators are started in ascending order of variable cost (`cost_variable`) to cover the Net Load. If thermal capacity is still insufficient at its limit, battery discharge is used as the last line of defense.
  3. **High-price arbitrage:** When the current market price is higher than the cost of a thermal unit that is already online, the system pushes that unit to its maximum output and sells the surplus capacity to the grid to maximize sales revenue.
  4. **Water-filling tracing:** `power_tracer.py` uses a water-filling algorithm to map the supply-demand flow between each generator/battery and each individual task, producing the $k_{j,i,t}$ matrix and guaranteeing strict energy conservation.

### 4. Level 2 Advanced Dynamic Rescheduling
* **Module:** `src/advanced_scheduler.py` (active only when `LEVEL2_ENABLED = True`)
* **How it works:**
  Moves beyond the idealized setting by introducing a realistic electricity-market mechanism and weather randomness:
  1. **Renewable uncertainty:** Actual renewable generation deviates randomly from the day-ahead forecast by up to ±20% (`forecast_error_ratio`).
  2. **Market default mechanism:** The system must make a Day-Ahead Commitment to sell power to the grid. If it cannot deliver because real-time renewable output falls short, it incurs a heavy default penalty (`penalty_rate`).
  3. **Dynamic rescue strategy:** During an acute power shortage, the system overrides the original schedule, forces expensive thermal units to ramp up, factors battery degradation (aging) cost into battery discharge, and can even push deferrable Aperiodic tasks back into the waiting queue (Defer), so that hard-constrained tasks and market commitments are honored first.
