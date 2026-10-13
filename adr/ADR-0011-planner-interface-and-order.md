# ADR-0011 – Planner Interface and Execution Order

**Date:** 2026-10-05 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment, `planner/` package) **Related ADRs:** ADR-0006 (simulation the
planners evaluate against), ADR-0007 (charge planner), ADR-0010 (discharge
planner) **Related capability:** CAP-07, CAP-09 (`tasks/adr-capability.md`)

---

## Context

Two planners now exist: charging (ADR-0007) and discharging (ADR-0010). Both
read the same slot-wise picture, claim slots on it, and report what they
achieved. Their order — the **planner order** — changes the outcome. **Discharge
first** fixes the discharge slots, so the charge planner sizes its energy to the
slots that will actually exist. **Charge first** plans charging against the
BMS's default behavior (discharge allowed everywhere unclaimed) and lets the
discharge planner place its slots around the charge slots. The order is a user
choice, which calls for one uniform way to run a planner.

---

## Decision

### 1 — `PlanContext`

An immutable dataclass in `planner/context.py` (ADR-0001 §3), pure: the
**planning horizon**'s slots; per-slot prices and tiers; per-slot expected PV
energy, the **consumption band** (high, mean and low series, ADR-0006 §5) and
the standby power; the BMS-derived bounds and limits (`E_now`, `E_min`,
`E_target`, `E_max`, max charge/discharge energy per slot in Wh, η); the
**claims** so far (per slot `charge` with its grid energy, `discharge`, or
unclaimed); `unassigned_mode` — `discharge` until a discharge planner has run,
`hold` afterwards (ADR-0006 §3) — the **unassigned mode**; and a `report`
mapping planners append to (planned energy, **unmet gap**, spill-relief slots).

### 2 — `Planner`

An abstract base class, also in `planner/context.py`, with one method,
`plan(context) -> PlanContext`: pure, returning a new context with the planner's
own claims added. A planner never changes a slot claimed earlier, and may call
ADR-0006's `simulate(context, scenario)` (`planner/simulation.py`) as often as
it needs, choosing the scenario that is conservative for its goal.

### 3 — Execution order

The config option `planner_order` is `discharge_first` (default) or
`charge_first`. The coordinator builds the sequence from it and runs it once per
replan. `charge_first` is the less consistent order: restricting discharge
afterwards changes the trajectory the charge plan was sized against. It is
offered because the right order depends on the installation.

### 4 — Deliberately minimal

One method, one context type; no registry, no entry points, no user-selectable
planner implementations. A third planner means a new class and a new value of
`planner_order`.

There is deliberately **no PV-charging planner**: PV surplus charging is the
BMS's default behavior, modelled by the simulation (ADR-0006 §3), not a decision
Chargy can influence. What such a planner would otherwise do — leave room for
expected PV, avoid grid charging into spill, discharge early to make room — the
charge planner's spill check (ADR-0007 §2) and the discharge planner's
spill-relief slots (ADR-0010 §3) already cover. It becomes a planner when a
lever exists, i.e. once actuation does (ADR-0004 §3).

---

## Consequences

- **Pro:** The order is data, not code: the coordinator needs no branching per
  order.
- **Pro:** Each planner is tested alone with a hand-built `PlanContext` —
  zero-mocking tier (ADR-0000 §6).
- **Con:** Sequential greedy passes are not a global optimum.
- **Con:** An abstract class for two implementations is a small overhead; it is
  kept thin so that dropping it later costs little.
