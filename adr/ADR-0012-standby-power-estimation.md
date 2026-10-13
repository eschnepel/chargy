# ADR-0012 – Standby Power Estimation

**Date:** 2026-10-12 **Status:** Draft **Related ADRs:** ADR-0004 §1 (value
sources, `get_standby_power()`), ADR-0001 §5/§6/§9 (`window_days`, `fetch_fn`,
units), ADR-0005 §4 (subtracts standby), ADR-0006 §3 (simulates standby)
**Related capability:** CAP-10 (`tasks/adr-capability.md`, pending Gate 1
confirmation)

---

## Context

**Standby power** is the power the battery system itself draws to keep running,
in every mode and even below Min SoC. Two consumers need it: ADR-0006's
simulation takes it in every slot, and ADR-0005 §4 subtracts the energy it
consumes from the efficiency measurement so it is never counted twice.

ADR-0004 §1 lets the user pick a sensor or enter a fixed value. Many BMS
integrations expose neither, and a fixed figure is a guess the user would have
to measure by hand. The installation's own history can answer the question:
while the BMS neither draws from nor feeds the grid, and the battery is not
being charged, the SoC can only fall because of standby power.

---

## Decision

### 1 — When the estimate applies

The setting `standby_power` (ADR-0004 §1) is resolved by the coordinator in this
order: the picked entity; else the fixed value (W, ≥ 0); else **auto**, stored
as `-1` and the default. Only in auto does this ADR's estimator run. The adapter
returns `None` for auto (ADR-0004 §1) so that the coordinator knows to ask
`standby.py`.

### 2 — Principle

Standby power is the energy the battery loses during **idle runs**, divided by
the total duration of those idle runs, over the **history window**
(`window_days`, ending now, ADR-0001 §5).

### 3 — Inputs

Read from the recorder through the injected `fetch_fn` (ADR-0001 §6), using the
entity IDs the BMS adapter exposes (ADR-0004 §1/§4): SoC and capacity as
5-minute series (`mean`), and **grid draw** and **grid feed** of every mapped
**phase** as 5-minute series (`mean`, W). All sensors must carry a
`state_class`. A slot for which any needed value is `None` is not idle and
breaks a run.

### 4 — Idle run

An **idle run** is a stretch of consecutive slots in which

- grid draw and grid feed both read zero on every mapped phase — below a small
  tolerance `STANDBY_IDLE_TOLERANCE_W` (proposed 2 W, absorbs sensor noise) —
  and
- the SoC never rises from one slot to the next.

The first condition says the BMS is neither supplying the house nor charging
from the grid; the second excludes PV charging, which Chargy has no direct
sensor for. Idle runs arise overnight at low SoC (the BMS stops discharging at
Min SoC) and in `hold` slots without PV.

A run counts only if it lasts at least `STANDBY_MIN_RUN_HOURS` (proposed 3
hours) and its SoC falls by at least `STANDBY_MIN_SOC_DROP` percentage points
(proposed 2), which guards against coarse SoC resolution: a drop of one step in
a short run would otherwise dominate the estimate.

### 5 — Estimate

```
energy(run)   = (SoC(start) − SoC(end)) / 100 × capacity            (Wh; capacity = mean over the run)
standby_power = Σ energy(run) / Σ duration(run in hours)            (W)
```

Duration-weighted over all qualifying runs, so long runs count more. The result
is clamped to ≥ 0 (a negative value means the runs were polluted by charging and
is rejected as unusable, see §6).

### 6 — Fallback and exposure

Until at least one run qualifies, or if §5 yields a rejected value, the
placeholder `DEFAULT_STANDBY_POWER = 10` W is used. The standby-power sensor
exposes the effective value and an attribute stating where it came from:
`picked`, `fixed`, `measured` (this estimator) or `default` (the placeholder).
The estimate is rebuilt at startup and once per day (ADR-0001 §6).

### 7 — Relationship to ADR-0005

The estimator reads only SoC, capacity, grid draw and grid feed, never the
efficiency, so there is no circular dependency: the coordinator resolves standby
power first and passes the effective value to ADR-0005 §4 and to the planners
(ADR-0001 §3).

### 8 — Module

`standby.py` is pure (zero-mocking tier, ADR-0000 §6): it takes the fetched
series as arguments and imports neither the recorder nor Home Assistant.

---

## Consequences

- **Pro:** No manual measurement: the figure comes from the installation's own
  behavior and adapts as the battery ages.
- **Pro:** A picked sensor or fixed value always wins, so nothing changes for
  hardware that exposes the figure.
- **Pro:** Everything is rebuilt from the recorder; nothing is persisted.
- **Con:** Early operation uses the 10 W placeholder until a qualifying idle run
  exists, which needs a night with an idle battery.
- **Con:** The idle-run thresholds (2 W, 3 hours, 2 percentage points) are
  proposed defaults, not validated against live data. SoC resolution is unknown
  until implemented; a coarse sensor may need longer runs.
- **Con:** A PV-charged slot that happens to show a flat SoC could slip into a
  run and bias the estimate slightly low.
