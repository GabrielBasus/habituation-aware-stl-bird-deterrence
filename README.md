# Habituation-Aware Signal Temporal Logic for Predictive Multi-Robot Bird Deterrence in Vineyards

A simulation framework for multi-robot bird deterrence in vineyard environments. The system addresses a core limitation of predictive robot dispatch: when robots repeat the same deterrence cue in the same location, birds habituate and the cue becomes less effective over time. This framework reformulates predictive task value as the counterfactual robustness improvement of a Signal Temporal Logic (STL) specification that explicitly encodes anti-habituation objectives alongside exposure reduction and spatial coverage, so the planner actively selects cue sequences that preserve deterrence effectiveness.

See [`docs/abstract_GabrielBasus.pdf`](docs/abstract_GabrielBasus.pdf) for a full summary of the problem, approach, and results.

---

## The Problem

Static scare devices fail in vineyards because birds habituate to repeated stimuli. Mobile robot teams can continuously vary cue type and location, reducing habituation. However, a naive predictive planner — one that scores tasks by their immediate exposure reduction — provides no signal for choosing among cue types: if two cues suppress equally today but one has been overused and will be ignored tomorrow, the legacy value function rates them identically.

The existing scalar value function ΔJ captures only the immediate reduction from a single action, which greedy dispatch already optimizes. Because habituation makes action value depend on the full history of deployed cues and mode variety, ΔJ provides no planning signal for a history-aware planner to exploit.

---

## The System

The simulator runs a closed-loop environment where:

1. A **ground-truth bird process** generates arrival events using a self-exciting spatiotemporal point process (SESTPP) with suppression from recent deterrence events. When habituation is enabled, suppression from a deterrence event is scaled by the current effectiveness `η` of that cue in that cell.

2. Each robot carries an **online SESTPP forecast model** that it uses to identify hotspots and score candidate tasks.

3. **Task value** is computed in one of two modes:
   - **Legacy mode (ΔJ):** counterfactual exposure reduction from a single action.
   - **STL robustness mode:** counterfactual improvement in the robustness of a mission specification `Φ = φ_exp ∧ φ_cov ∧ φ_hab`, encoding exposure reduction, spatial coverage recency, and cue effectiveness.

4. A **reserved-capacity dispatcher** allocates predictive (planned) and reactive (event-driven) tasks to robots while enforcing a capacity reservation fraction `ρ_res` that keeps reactive slots available.

### Baselines

Six systems are evaluated in a ladder from reactive-only to full habituation-aware STL:

| System | Description |
| --- | --- |
| B0 | Reactive only: responds to detections, no predictive dispatch |
| B1 | Greedy predictive with fixed cue selection (ΔJ value, no reservation) |
| B2 | Reserved-capacity dispatch with ΔJ value and fixed cue selection |
| B3 | Reserved-capacity dispatch with STL value, no habituation clause (`φ_exp ∧ φ_cov` only) |
| B4 | Full system: reserved-capacity dispatch with STL value including habituation clause (`φ_exp ∧ φ_cov ∧ φ_hab`) |
| B5 | Greedy habituation-aware cue rotation (no STL, no reservation) |

B4 is the proposed system. Comparing B4 to B3 isolates the contribution of the habituation clause. Comparing B3 to B2 isolates the contribution of STL structure over ΔJ. Comparing B4 to B5 separates predictive STL planning from simple cue rotation.

### The STL Specification

For robot `r`, the task value of candidate action `a` is:

```
U(a, r) = ρ(Φ_r, ξ_with_action) − ρ(Φ_r, ξ_without_action)
```

where `ξ` is the predicted local signal trace and `ρ` is the smooth conjunctive STL robustness. The three clauses are:

- `φ_exp`: value-weighted exposure stays below threshold `E*`
- `φ_cov`: coverage age in each cell stays below `T_cov`
- `φ_hab`: cue effectiveness `η` stays above `η_min`

The habituation state `η ∈ (0, 1]` decays multiplicatively on each application (rate `κ`) and recovers exponentially (time constant `T_rec`). The same habituation field drives both the planner's value estimate and the truth-process suppression, so the planner's forward model is consistent with ground truth.

---

## Installation

Python 3.10 or later is required.

```bash
pip install -r requirements.txt
```

---

## Reproducing Results

There are two paths: a quick start that regenerates figures from the included pre-computed data, and a full re-run that executes all simulations from scratch.

### Quick start: regenerate figures and tables

Pre-computed merged data is included in `results/testbench/acc_merged/`. To regenerate all figures and tables:

```bash
python -m experiments.plot_acc_results
```

Output is written to `results/acc_submission/figures/` (PDF, SVG, EPS) and `results/acc_submission/tables/` (CSV, Markdown, LaTeX).

### Full re-run from scratch

Run the steps below in order. Each step writes its output to `results/testbench/`. Steps 1 through 6 can be run independently (they write to separate subdirectories); Step 7 merges them; Step 8 generates figures.

**Step 1 — Confirmatory ladder at spare load (µ=2×10⁻⁵, n=30 seeds):**

This is the primary confirmatory experiment. It runs all six baselines (B0–B5) under both habituation-on and habituation-off conditions.

```bash
python -m experiments.run_habituation_stl_production_ladder \
    --outdir results/testbench/habituation_stl_revised_confirm \
    --duration-s 1800 --num-runs 30 --seed-start 125 \
    --nx 60 --ny 48 --nrobots 4 \
    --reservation-fraction 0.25 \
    --mu-true 2e-05 \
    --deterrence-beta-scale 4.0 \
    --deterrence-sigma-scale 4.0 \
    --deterrence-omega-scale 3.0 \
    --habituation-kappa 0.5 \
    --systems B1_greedy_fixedcue B2_res_deltaJ_fixedcue \
              B3_res_stl_nohab_fixedcue B4_res_stl_full_multicue B5_greedy_habcue \
    --max-workers 4
```

**Step 2 — Joint κ × ρ_res sweep:**

Sweeps habituation strength κ and reservation fraction ρ_res jointly to show the dose-response relationship and identify the sensitivity of the primary result.

```bash
python -m experiments.run_habituation_stl_joint_sweep
```

**Step 3 — H4 ρ_res sensitivity sweep:**

Sweeps ρ_res alone at the confirmatory load to characterize how the reservation fraction affects the reactive miss rate and overall exposure.

```bash
python -m experiments.run_h4_rho_res_sweep
```

**Step 4 — 2×2 cost decomposition:**

Runs fixed-cue and multi-cue variants of B3 and B4 to isolate whether overhead comes from the reservation mechanism or the multi-cue selection logic.

```bash
python -m experiments.run_2x2_overhead_isolation
```

**Step 5 — B2 and B3 seed top-up to n=30:**

Extends B2 and B3 (which ran n=10 in early confirmatory runs) to n=30 seeds at all three load levels for consistency.

```bash
python -m experiments.run_b2_b3_topup
```

**Step 6 — Preemption variant:**

Evaluates a policy variant in which an arriving urgent reactive task can preempt an in-progress predictive task, testing whether preemption improves reactive responsiveness at the cost of predictive throughput.

```bash
python -m experiments.run_preemption_variant
```

**Step 7 — Merge experiment outputs:**

Merges all per-run CSVs from the steps above into the unified `results/testbench/acc_merged/` directory used by the plotter. Run this after all experiments complete.

```bash
python -m experiments.build_acc_merged_data
```

**Step 8 — Generate figures and tables:**

```bash
python -m experiments.plot_acc_results
```

---

## Repository Structure

```
.
├── DeterrentSystem.py          # Core simulation: ground-truth bird process, closed-loop
│                               # intervention feedback, habituation in truth process, metrics
├── Robot.py                    # Robot agent: health-weighted zone partitioning,
│                               # neighbor sharing, boundary event exchange
├── SESTPP.py                   # Online spatiotemporal intensity model
│                               # (self-exciting trigger + inhibitory intervention mass)
├── TaskGenerator.py            # Task candidate generation and scoring;
│                               # STL robustness value when predictive_utility_mode="stl_robustness"
├── ZonePartitioner.py          # Zone geometry and spatial utilities
├── planner_dispatch.py         # Dispatch queue logic and capacity enforcement
├── planner_profiles.py         # Named planner configuration profiles (B0–B5 definitions)
├── planner_task_estimation.py  # Counterfactual deterrence estimator; converts SESTPP fields
│                               # and coverage memory into CellState inputs for STL value
├── planner_task_extraction.py  # Task candidate buffer management
├── planner_task_generation.py  # Task generation pipeline
├── planner_task_selection.py   # Pre-assignment candidate selection
├── system_structure.py         # System-level data structures and metric containers
├── system_stage_helpers.py     # Per-step stage execution helpers
├── graph_motion.py             # Graph-based robot motion model
├── calibration_config.py       # SESTPP calibration loader
├── config_loader.py            # Configuration file utilities
├── action_schema.py            # Task action type definitions
├── telemetry_sim.py            # Live telemetry CSV output
├── tracking_export.py          # State tracking export utilities
│
├── habituation_stl/            # STL and habituation package (numpy-only, self-contained)
│   ├── stl.py                  # Quantitative STL robustness operators and smooth aggregators
│   ├── habituation.py          # HabituationField: per-cell/per-mode effectiveness with recovery
│   ├── mission_spec.py         # SpecParams and mission clauses (exp, cov, hab)
│   ├── task_value.py           # Counterfactual task value U(a,r)
│   ├── signals.py              # RobotMonitor for rolling STL robustness telemetry
│   ├── metrics.py              # Exposure, cue variety, and habituation diagnostics
│   ├── dispatch.py             # Reference reserved-capacity dispatcher
│   └── reference_sim.py       # Standalone B0–B4 smoke-test simulation
│
├── experiments/
│   ├── run_habituation_stl_production_ladder.py   # Main B0–B5 ladder experiment runner
│   ├── run_habituation_stl_joint_sweep.py         # Joint κ × ρ_res sweep
│   ├── run_h4_rho_res_sweep.py                    # H4: ρ_res sensitivity analysis
│   ├── run_2x2_overhead_isolation.py              # 2×2 cost decomposition experiment
│   ├── run_b2_b3_topup.py                         # B2/B3 seed top-up to n=30
│   ├── run_preemption_variant.py                  # Preemption policy variant
│   ├── build_acc_merged_data.py                   # Merge experiment outputs into acc_merged/
│   ├── plot_acc_results.py                        # Generate figures and tables
│   └── habituation_stl_selected_configs.py        # Fair-tuning config loader
│
├── results/
│   ├── testbench/acc_merged/   # Pre-computed merged data (included for quick start)
│   └── sestpp_calibration_sweep/  # SESTPP calibration parameters
│
├── docs/
│   └── abstract_GabrielBasus.pdf
│
├── requirements.txt
└── .gitignore
```

---

## Key Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `--mu-true` | `2e-5` | Background bird arrival rate (spare load) |
| `--habituation-kappa` | `0.5` | Habituation decay rate per cue application |
| `--reservation-fraction` | `0.25` | Fraction of robot capacity reserved for reactive tasks (ρ_res) |
| `--duration-s` | `1800` | Simulation duration in seconds |
| `--num-runs` | `10` | Number of independent seeds |
| `--nx`, `--ny` | `60`, `48` | Grid resolution |
| `--nrobots` | `4` | Number of robots |

Load regimes used in experiments:

| Load | `--mu-true` | Description |
| --- | --- | --- |
| Spare | `2e-5` | Primary confirmatory load |
| Heavy spare | `1e-4` | Intermediate load |
| Overloaded | `4e-4` | High-demand stress test |

---

## Output Files

Each experiment run produces a `per_run_metrics.csv` with one row per (seed, system, habituation condition). Key columns:

| Column | Description |
| --- | --- |
| `value_weighted_exposure` | Primary metric: sum of value-weighted accepted truth events |
| `reactive_completed_frac` | Fraction of reactive tasks completed (1 − miss rate) |
| `override_count` | Number of reactive override events |
| `stl_robustness_mean` | Mean STL robustness over the run |
| `variety_index` | Cue variety index (higher = more diverse cue deployment) |
| `eta_at_apply_mean` | Mean cue effectiveness at task execution time |

After running `build_acc_merged_data.py`, merged CSVs appear in `results/testbench/acc_merged/`. After running `plot_acc_results.py`, figures appear in `results/acc_submission/figures/` and tables in `results/acc_submission/tables/`.
