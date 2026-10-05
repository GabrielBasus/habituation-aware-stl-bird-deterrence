# Habituation-Aware Signal Temporal Logic for Predictive Multi-Robot Bird Deterrence in Vineyards

Code for the ACC 2027 paper by Gabriel Basus (Cal Poly).

## Abstract

See [`docs/abstract_GabrielBasus.pdf`](docs/abstract_GabrielBasus.pdf).

## Requirements

Python 3.10+

```bash
pip install -r requirements.txt
```

## Reproducing results

There are two paths: **quick start** (regenerate figures from pre-computed data) and **full re-run** (run all simulations from scratch).

### Quick start: regenerate figures and tables

Pre-computed merged data is included in `results/testbench/acc_merged/`. To regenerate all 7 figures and 4 tables from the paper:

```bash
python -m experiments.plot_acc_results
```

Output is written to `results/acc_submission/figures/` (PDF, SVG, EPS) and `results/acc_submission/tables/` (CSV, Markdown, LaTeX).

### Full re-run from scratch

Run experiments in order. Each step writes its output to `results/testbench/`.

**Step 1 — Confirmatory ladder (mu=2e-5, n=30 seeds):**
```bash
python -m experiments.run_habituation_stl_production_ladder
```

**Step 2 — Joint kappa x rho sweep:**
```bash
python -m experiments.run_habituation_stl_joint_sweep
```

**Step 3 — H4 rho_res sweep:**
```bash
python -m experiments.run_h4_rho_res_sweep
```

**Step 4 — 2x2 cost decomposition:**
```bash
python -m experiments.run_2x2_overhead_isolation
```

**Step 5 — B2/B3 seed top-up to n=30:**
```bash
python -m experiments.run_b2_b3_topup
```

**Step 6 — Preemption variant:**
```bash
python -m experiments.run_preemption_variant
```

**Step 7 — Merge into acc_merged/:**
```bash
python -m experiments.build_acc_merged_data
```

**Step 8 — Generate figures and tables:**
```bash
python -m experiments.plot_acc_results
```

## Repository structure

```
.
├── DeterrentSystem.py          # Core simulation: ground-truth bird process, dispatch, metrics
├── Robot.py                    # Robot agent: zone partitioning, neighbor coordination
├── SESTPP.py                   # Online spatiotemporal intensity model
├── TaskGenerator.py            # Task candidate generation and STL-based scoring
├── ZonePartitioner.py          # Zone partitioning utilities
├── planner_*.py                # Dispatch planner components
├── system_*.py                 # System wiring and stage helpers
├── habituation_stl/            # STL specification, habituation model, task value
├── experiments/
│   ├── run_habituation_stl_production_ladder.py   # Main B0-B4 ladder experiment
│   ├── run_habituation_stl_joint_sweep.py         # Joint kappa x rho sweep
│   ├── run_h4_rho_res_sweep.py                    # H4: rho_res sensitivity
│   ├── run_2x2_overhead_isolation.py              # 2x2 cost decomposition
│   ├── run_b2_b3_topup.py                         # B2/B3 seed top-up to n=30
│   ├── run_preemption_variant.py                  # Preemption policy variant
│   ├── build_acc_merged_data.py                   # Merge experiment outputs
│   └── plot_acc_results.py                        # Generate ACC figures and tables
├── results/
│   ├── testbench/acc_merged/   # Pre-computed merged data (included)
│   └── sestpp_calibration_sweep/  # SESTPP model calibration (included)
└── docs/
    └── abstract_GabrielBasus.pdf
```

## Citation

If you use this code, please cite:

```
Gabriel Basus. Habituation-Aware Signal Temporal Logic for Predictive
Multi-Robot Bird Deterrence in Vineyards. ACC 2027.
```
