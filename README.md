# NCKU-RTOS-2026 Virtual Power Plant (VPP) Dynamic Scheduling System

![Python](https://img.shields.io/badge/language-Python-3776AB?logo=python&logoColor=white)
![pytest](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)
![Scheduling](https://img.shields.io/badge/scheduling-offline%20DFS%20%2B%20online%20admission-555)
![License: MIT](https://img.shields.io/badge/license-MIT-yellow)

English | [繁體中文](README.zh-TW.md)

We built a 72-hour scheduler for a virtual power plant. Periodic tasks are planned offline with a frame-based DFS. Sporadic and aperiodic tasks go through online admission control. Level 2 adds forecast error, a market commitment and a battery model. It can run a 10-scenario batch or a single Demo file, in Level 1 (baseline) or Level 2 (advanced) mode.

## Quick Start

Requirements:
- Python 3 (checked with 3.13). Run everything from the repository root.
- `src/scheduler.py`, `src/evaluator.py` and `src/task_generator.py` use only the standard library.
- `src/aperiodic_n_sporadic_gen.py` needs NumPy (`pip install numpy`); the tests need pytest (`pip install pytest`).

### Level 1 / Level 2
Level 2 is the default (`LEVEL2_ENABLED = True`). For Level 1, set it to `False` at line 40 of `src/scheduler.py`:
```python
# src/scheduler.py
LEVEL2_ENABLED = False  # False runs Level 1, True runs Level 2
```
Level 1 uses the renewable forecast as-is and the Level 1 battery model, with no market commitment. Its output file names have no `level2_` prefix.

### Batch vs. Demo mode
If `input/aperiodic_n_sporadic.json` exists, `src/scheduler.py` runs Demo mode; otherwise it runs Batch mode. The repository ships a sample file, so `python src/scheduler.py` on a fresh clone runs Demo mode.

Batch mode simulates the 10 `scenario_*.json` files in `output/sporadic_aperiodic_task/`. Move the Demo file away first:
```bash
mv input/aperiodic_n_sporadic.json input/aperiodic_n_sporadic.json.bak
python src/scheduler.py
```
- Per-scenario files go to `output/`: `schedule_result_level2_<scenario>.json`, `acceptance_test_log_level2_<scenario>.json`, `evaluation_results_level2_<scenario>.json`.
- The evaluator then runs over all scenarios (`src/evaluator.py --batch-level2`, or `--batch-scenarios` in Level 1). The summary is `output/evaluation_results_level2_summary.json` (Level 1: `output/evaluation_results_summary.json`).
- Scenario `scenario_01_uniform` is also copied to `schedule_result.json`, `acceptance_test_log.json` and `evaluation_results.json`, so `evaluation_results.json` covers scenario_01 only.

Move the file back (`mv input/aperiodic_n_sporadic.json.bak input/aperiodic_n_sporadic.json`) to return to Demo mode.

Demo mode: put your burst-task file at `input/aperiodic_n_sporadic.json` (replacing the sample) and run `python src/scheduler.py`. The batch scenarios are skipped and the results overwrite `output/schedule_result.json`, `output/acceptance_test_log.json` and `output/evaluation_results.json`. The committed copies of these three files come from a run on the bundled sample.

### Tests
```bash
pip install pytest
python -m pytest
```
Use `python -m pytest`, not bare `pytest`, so `src` is importable. This runs 20 tests in `tests/test_acceptance_tester.py` and `tests/test_state_machine.py`. `tests/test_offline_pipeline.py` is a standalone integration script: `python -m tests.test_offline_pipeline`.

### Regenerating inputs (optional)
- `python src/task_generator.py` rebuilds `output/task_set.json`. The seed is fixed, so it reproduces the committed file.
- `python src/aperiodic_n_sporadic_gen.py` rewrites the 10 scenario files (3 uniform, 3 concentrated, 4 mixed; seeds 2027–2036). The committed `scenario_01_uniform.json` differs from the generator's current output, so re-running changes it.

## Project Structure

```text
Embedded-RTOS-Scheduler/
├── report.pdf                       # System design report
├── runtime_config.json              # Copy of the Level 2 parameters (not read by any code)
├── docs/level2_model.md             # Level 2 formal model (constraints, pseudo-code)
├── src/
│   ├── scheduler.py                 # Entry point: Level 1/2 switch, Batch/Demo mode
│   ├── evaluator.py                 # Evaluation and scoring
│   ├── advanced_scheduler.py        # Level 2: forecast error, battery model, market commitment, defer
│   ├── task_generator.py            # Periodic task set -> output/task_set.json
│   ├── aperiodic_n_sporadic_gen.py  # Scenario generator (NumPy)
│   └── engine/
│       ├── offline_planner.py       # Frame-based DFS planner
│       ├── acceptance_tester.py     # Online admission control
│       ├── main_scheduler.py        # 72-hour simulation and dispatch loop
│       ├── power_tracer.py          # Water-filling power tracing (k matrix)
│       ├── state_machine.py         # Generator / battery transition validation
│       ├── data_loader.py           # JSON loaders
│       └── models.py                # Data classes
├── tests/                           # pytest tests + offline-planner integration script
├── input/                           # processor_settings.json, price_72hr.json, Demo sample
└── output/                          # task_set.json, 10 scenarios, results and summaries
```

`runtime_config.json` duplicates `LEVEL2_CONFIG` in `src/scheduler.py`. The scheduler uses `LEVEL2_CONFIG`; edit that to change Level 2 parameters.

## Periodic Task Set Generation

`src/task_generator.py` draws `N = 6–10` tasks. The committed `output/task_set.json` (seed 2026) has `N = 6`, `DW ≈ 0.99` and 44 jobs over 72 hours.

- Execution times are `[1]*(N-4) + [2, 2, 3, 3]`. Only the two `e=2` tasks are non-preemptive; the two `e=3` tasks (`d=3`) stay preemptive. A non-preemptive `e=3, d=3` task has zero slack and needs one exact 3-hour block, which fragments the schedule and forces more DFS backtracking.
- `(N-2)//2` tasks use `p=6`, which gives a steady base load. The generator does not target a specific DW; `validate()` accepts only `0.7 ≤ DW ≤ 1.0`.
- The remaining first `N-2` tasks share one period from `{11, 12}`; the last two take two distinct periods from `{15, 18, 21, 24}`. The first `N-2` tasks draw `d` from `[6, p]`, which satisfies Frame visibility for `f=3` (`2f−gcd(f,p)≤d`).
- The last two tasks have `d=3` (`=e`), meeting the requirement that at least 20% of tasks have `d=e`. Earlier tasks keep relaxed deadlines (≥6), leaving room to shift energy and ramp generators.
- `RANDOM_SEED=2026`. `validate()` checks Frame visibility, `0.7 ≤ DW ≤ 1.0`, more than 30 jobs in 72 hours, at least 3 distinct periods, and `d ≥ e` for non-preemptive tasks. The generator draws once; `main()` writes `output/task_set.json` only if validation passes.

## Scheduling Engine

Periodic tasks get an offline plan. Everything else goes through online admission and dispatch.

### 1. Offline pre-scheduling
`src/engine/offline_planner.py`, for periodic tasks.

A DFS over 24 consecutive 3-hour Frames (72 hours):
1. For the jobs active in the current Frame, enumerate slot assignments for its three hours.
2. Try candidate patterns longest and earliest first, so tasks finish early. Prune patterns that leave more remaining work than hours left before the deadline.
3. Check each combination hour by hour: renewable forecast is absorbed first, then thermal units (output, ramp-rate and minimum up/down-time limits) and battery discharge (discharge and SOC limits) cover the load. If the Frame fails or no later Frame can be completed, restore the generator/battery snapshot and try the next combination (backtracking).

Output: a 72-hour base schedule and the hourly slack capacity (remaining thermal and battery headroom).

### 2. Online admission control
`src/engine/acceptance_tester.py`, for sporadic and aperiodic tasks.
- Sporadic tasks (hard deadline): check whether enough hours before the deadline (contiguous for non-preemptive tasks) have slack ≥ the task's demand. If so, reserve them; otherwise reject.
- Aperiodic tasks (soft): reject on arrival if the task fits nowhere in the remaining slack; otherwise queue it. Each hour the queue is scanned in order: tasks that fit run (preemptive one hour at a time, non-preemptive as a whole block), and tasks that do not fit are skipped so later ones can run (backfilling avoids head-of-line blocking). A queued task is dropped after waiting more than 24 hours, or when its remaining execution time exceeds the hours left in the 72-hour horizon.

### 3. Real-time dispatch and tracing
`src/engine/main_scheduler.py`, `src/engine/power_tracer.py`. Each hour:
1. Absorb all renewable energy first (marginal cost 0) to get the net load.
2. Starting from each generator's minimum output, raise generators in ascending `cost_variable` order to cover the net load. If thermal capacity is not enough, discharge the battery.
3. If the market price exceeds the variable cost of a unit that is already online, push it to the highest output its ramp limits allow and sell the surplus.
4. `power_tracer.py` maps each supply source (renewable, generator, battery) to each task with a water-filling algorithm, producing the $k_{j,i,t}$ matrix. A task left short raises an error; leftover supply is recorded as sold energy, so supply = consumption + sales every hour.

### 4. Level 2 dynamic rescheduling
`src/advanced_scheduler.py`, active only when `LEVEL2_ENABLED = True`; parameters are in `LEVEL2_CONFIG` in `src/scheduler.py`.
1. Actual output is the forecast scaled by a relative error drawn uniformly from ±20% (`forecast_error_ratio`, seed `random_seed = 2026`), clipped to unit capacity.
2. The system commits to sell a fixed amount each hour (`5.0 × commitment_ratio = 4.0` MWh). A shortfall is penalized at `penalty_rate` per MWh; sales above the commitment earn `(realtime_price_multiplier − 1) × price`.
3. The battery model has charge/discharge efficiencies, self-discharge and an SOC-dependent discharge limit. Each discharged MWh adds a degradation cost to the Level 2 adjusted objective. Surplus supply charges the batteries through a `<battery>_chg` pseudo-job.
4. When actual renewable output falls below the forecast, the system logs how much of the gap battery discharge and thermal ramp-up headroom could cover (estimates only); dispatch then covers the net load as in step 3 above. If the estimated headroom is not enough, aperiodic jobs running that hour (IDs starting with `a_`) are deferred back to the queue so periodic and sporadic jobs keep their power.
