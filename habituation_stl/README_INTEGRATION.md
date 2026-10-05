# habituation_stl Package

`habituation_stl` is the self-contained STL and habituation implementation used by the production vineyard simulator. It is intentionally small and numpy-only so the production system can keep its existing SESTPP model, zone partitioning, robot fleet logic, task admission, and dispatch policies unchanged.

The package replaces the predictive deterrence value when the simulator is configured with:

```
predictive_utility_mode = "stl_robustness"
```

---

## Module Map

| Module | Responsibility |
| --- | --- |
| `stl.py` | Quantitative STL robustness operators, temporal operators, smooth aggregators, and clause scaling |
| `habituation.py` | `HabituationField`: per-cell/per-mode cue-effectiveness model with multiplicative decay and exponential recovery |
| `mission_spec.py` | `SpecParams` and mission clauses for exposure (`φ_exp`), coverage (`φ_cov`), and habituation (`φ_hab`) |
| `task_value.py` | Counterfactual task value `U(a,r)` computed from robustness improvement |
| `signals.py` | `RobotMonitor` for rolling STL robustness telemetry during a run |
| `metrics.py` | Utility metrics for exposure, cue variety, and habituation diagnostics |
| `dispatch.py` | Reference reserved-capacity dispatcher used by the standalone harness |
| `reference_sim.py` | Standalone smoke-test simulation (B0–B4 reference ladder, numpy-only) |
| `experiment.py` | Experiment helpers for the standalone harness |

---

## How It Integrates With the Production System

The package is responsible only for the STL specification and habituation state. The production adapters outside this package handle the rest:

- `DeterrentSystem.py` owns runtime habituation state, ground-truth suppression scaling (truth events suppressed by `η × β_mode × kernel × decay`), coverage service time tracking, and final metrics collection.
- `planner_task_estimation.py` converts SESTPP fields and coverage memory into `CellState` inputs for `counterfactual_value(...)`.
- `TaskGenerator.py` scores model-scored predictive deterrence candidates using the STL value and selects the best action mode.
- `planner_task_extraction.py` preserves STL utility fields through the dispatch pipeline.
- `system_structure.py` exports STL and habituation config and metric sections.

The dispatchers need no special STL logic. STL mode writes `predictive_stl_U` and mirrors it into `utility`, `score`, `predicted_deltaJ`, and `deltaJ_per_cost` so existing dispatch policies work without change.

---

## STL Task Value

For candidate action `a` and robot `r`, the task value is:

```
U(a, r) = ρ(Φ_r, ξ_with_action) − ρ(Φ_r, ξ_without_action)
```

where `ξ` is the predicted local signal trace and `ρ` is the smooth conjunctive STL robustness. The trace contains local-cell exposure, coverage age, and cue effectiveness.

B3 and B4 differ only in their active clause set:

- B3: `("exp", "cov")` — exposure and coverage, no habituation awareness
- B4: `("exp", "cov", "hab")` — adds the habituation clause

When the habituation clause is inactive, the planner treats `η = 1.0` for all cues. When the clause is active, `η` is read from the live `HabituationField`.

---

## Habituation in Ground Truth

The truth process uses the same `HabituationField` concept as the planner.

For each completed deterrence event:

1. Resolve the cell and physical cue mode.
2. Read current effectiveness `η_at_apply` from the field.
3. Record the deterrence event with that `η`.
4. Apply habituation: `η ← η × (1 − κ)`, then schedule exponential recovery.
5. Update coverage service time for the cell.

During truth-event filtering, suppression from a recent deterrence event is scaled by its stored effectiveness:

```
suppression = η_at_apply × β_mode × spatial_kernel × temporal_decay
```

This makes repeated cue use less effective in the simulated world, consistent with the planner's forward model.

---

## Running the Standalone Reference Simulation

The package includes a standalone smoke-test harness that runs the B0–B4 ladder without the full production system. From the repo root:

```bash
python -m habituation_stl.reference_sim
```

---

## Running Tests

From the repo root:

```bash
python habituation_stl/run_all_tests.py
```
