# ADR-0006 – Slot-wise Energy Trajectory and Gap

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment, Wh, soft target SoC) **Related ADRs:** ADR-0004 (SoC, min/max SoC,
capacity, power limits, standby power), ADR-0005 (round-trip efficiency input),
ADR-0008 (consumption input), ADR-0009 (holiday-mode consumption), ADR-0012
(standby power estimate), ADR-0011 (`PlanContext`, slot claims) **Related
capability:** CAP-06 (`tasks/adr-capability.md`)

---

## Context

Chargy's core purpose is determining how much grid energy to buy, and when, so
that the planners (ADR-0007, ADR-0010) can place charging and discharging slots.
The **soft target SoC** is intentionally a configurable bound for *grid
charging* that sits at or below the BMS's own **Max SoC** (ADR-0004 §1's
`get_max_soc`), deliberately leaving headroom for PV production the forecast
under-predicted.

An earlier draft computed one aggregate over the whole planning horizon. That
cannot represent four things that matter: the battery can run empty tonight even
if tomorrow's PV balances the 24-hour total; PV beyond Max SoC is **spill**, not
banked; charging only helps if it happens *before* the shortfall it serves; and
planned discharging changes the trajectory. The gap is therefore a by-product of
a forward simulation over 5-minute slots.

Two physical facts shape the simulation. The BMS regulates its output to the
**grid meter reading** on its own (ADR-0004): it serves the house load and
Chargy never commands amounts, so the simulation models only *whether
discharging is allowed* in a slot. And the battery system keeps itself running
from the battery (**standby power**), so the SoC can fall below Min SoC
overnight.

Consumption is uncertain, and the uncertainty cuts differently for each planner:
charging must not fall short if consumption turns out higher, discharge slots
must make enough room if it turns out lower. The simulation is therefore run for
a higher and a lower **scenario**.

---

## Decision

### 1 — Inputs

All carried in the `PlanContext` (ADR-0011). Energy is in Wh, power in W
(ADR-0001 §9).

- Current SoC, capacity, Min SoC and Max SoC (`get_min_soc()`, `get_max_soc()`),
  the effective **standby power** (`get_standby_power()`, or the ADR-0012
  estimate when it is on auto) and the cached max charge/discharge power
  (ADR-0004).
- Round-trip efficiency η (ADR-0005).
- **Expected consumption** per 5-minute slot as a **consumption band**:
  `cons_mean` — the combined whole-house average for that weekday+slot
  **bucket** (ADR-0008 §4/§6), or while holiday mode is on the baseline-based
  figure (ADR-0009 §2) — plus `cons_high` and `cons_low` derived from it (§5).
  All are powers (W); like every per-slot figure below they are converted to
  energy per slot, Wh = W × 5/60.
- The externally sourced PV forecast at 5-minute resolution (already available;
  producing it is outside Chargy's scope), converted to energy per slot the same
  way if it is a power series.
- **Planning horizon:** from the current slot to the earlier of the **price
  horizon** (end of the price curve) and the **PV forecast horizon** (end of the
  PV forecast) — the slots for which all inputs exist.

### 2 — Bounds

`E_min = min_soc × capacity` and `E_max = max_soc × capacity`, both from the BMS
adapter (live when picked from an entity, constant when entered as a fixed
value, ADR-0004 §1). `E_min` is the floor for *discharging to cover load*; it is
**not a hard floor for the SoC**: standby power (§3) continues in every mode and
can take the SoC below `E_min`, down to 0, overnight. The soft target
`E_target = clamp(soft_target_soc, min_soc, max_soc) × capacity` —
`soft_target_soc` is a Chargy setting in % (ADR-0001 §5, default 80), the config
flow constrains it to lie between Min SoC and Max SoC, and every replan
re-applies the clamp because the BMS values can be live. `E_target` caps **grid
charging** (enforced by the charge planner, ADR-0007); PV may still fill the
battery up to `E_max`. The BMS's own limits always win.

### 3 — Slot-wise simulation and gap

`simulate(context, scenario)` takes a scenario — `high`, `mean` or `low`,
selecting `cons_high`, `cons_mean` or `cons_low` as `cons_i`. Per slot `i` (5
minutes): `net_i = pv_i − cons_i` (Wh). Each slot has a mode — `charge` (grid
charging planned, `g_i` Wh grid-side, at most `max_charge_power × 5/60`),
`discharge` (the battery may cover load) or `hold`. Slots no planner has claimed
take the **unassigned mode** `context.unassigned_mode` (ADR-0011): `discharge`,
the BMS's default behavior, until a discharge planner has run, `hold`
afterwards.

```
E ← E_now
for each slot i:
    if mode == charge:   E ← min(E + η·g_i, E_max)
    if net_i ≥ 0:        E ← E + net_i;  spill_i ← max(0, E − E_max);  E ← min(E, E_max)
    elif mode == discharge:
        supplied    ← min(−net_i, max_discharge_power·5/60, max(0, E − E_min))
        E           ← E − supplied
        shortfall_i ← −net_i − supplied
    else:  the load is imported from the grid          (planned, not a shortfall)
    E ← max(0, E − standby_i)                          (standby power)
    E_i ← E
```

PV surplus charges the battery in every mode, up to `E_max`; whatever does not
fit is `spill_i`. In `hold` and `charge` slots the battery does not discharge.
In `discharge` slots it covers the load until it reaches `E_min`; what it cannot
cover is the `shortfall_i`. Standby power `standby_i = standby_power × 5/60`
(Wh) is drawn in every slot and is not a shortfall. Simplifications: η applies
to grid charging only, PV-side conversion losses are out of scope (the sibling
project Effy handles them), out-of-range values are clamped, never raised
(ADR-0000). The result is the **trajectory**: per-slot `E_i`, `spill_i`,
`shortfall_i`.

**Gap**, measured under the `high` scenario on the trajectory *before any charge
slot is claimed*:

```
gap = Σ shortfall_i / η            (grid-side Wh)
```

It is a requirement, not a promise: feasibility (soft-target cap, power cap,
timing before the shortfall) is the charge planner's job, and it reports any
**unmet gap**. `gap` is the "grid energy needed" figure exposed to the user.

**Scenario per planner:** each planner evaluates the scenario that is
conservative for its goal — the charge planner the `high` scenario (enough
energy even if consumption is higher), the discharge planner the `low` scenario
(the most PV spill, so enough room is made even if consumption is lower). Both
choices widen the buffer. `mean` is for display only.

### 4 — Continuous replanning

The trajectory and gap are re-derived as SoC, PV forecast, or expected
consumption change, at the project's canonical 5-minute resolution. There is no
fixed daily reset or single point-in-time computation — the gap is always the
current, live figure. A change of the holiday switch (ADR-0009) triggers an
immediate replan rather than waiting for the next slot.

### 5 — Confidence-derived consumption band

The `cons_mean` input (§1) is ADR-0008's combined whole-house average per
weekday+slot. ADR-0008 also exposes a combined standard deviation for the same
bucket (ADR-0008 §3/§4), which sizes the band around it rather than planning
against the bare mean alone:

```
CV[bucket]         = stddev[bucket] / mean[bucket]
confidence[bucket] = 1 / (1 + CV[bucket])
band[bucket]       = (1 − confidence[bucket]) × stddev[bucket]

cons_high = mean + band
cons_low  = max(0, mean − band)
```

Confidence is bounded in `(0, 1]` and falls as a bucket's variability grows
relative to its own mean; the band scales from `0` (a fully stable bucket needs
no buffer) toward a full standard deviation (a highly volatile bucket is widened
by roughly its own spread). `stddev` and `mean` are per-slot power (W,
ADR-0008). Guarded against `mean ≈ 0`: treated as `CV → ∞`, i.e.
`confidence → 0`, the most cautious case, rather than dividing by zero.

This specific formula is **proposed, not validated** the way ADR-0008 §1/§2's
peak-detection algorithm was against live data — worth confirming once
implemented.

In holiday mode ADR-0008's interface reports a standard deviation of 0 (ADR-0009
§2), so the band is 0 and the three scenarios coincide without any special case
here; the holiday baseline factor is the buffer.

---

## Consequences

- **Pro:** Timing and bounds are respected: shortfalls are placed in time, PV
  spill is visible, and charging is asked for only where it can still help.
- **Pro:** Standby power is modelled where it happens, so overnight SoC (and the
  energy needed to reach a discharge slot) is realistic; ADR-0005 subtracts the
  same energy, so it is never counted twice.
- **Pro:** One simulation serves both planners (ADR-0007, ADR-0010) and the
  exposed trajectory, so charge and discharge decisions can never disagree about
  the battery's state.
- **Pro:** Choosing the conservative scenario per planner replaces a single
  added buffer: charging errs on the side of enough energy, discharging on the
  side of enough room.
- **Pro:** Bounding everything by the BMS's own floor and ceiling means Chargy
  can never ask for more than the hardware itself permits, regardless of a
  misconfiguration.
- **Con:** The result depends on the per-slot quality of the PV forecast and the
  consumption profile, not only on their totals, and on a configured standby
  power.
- **Con:** §5's confidence/band formula is a proposed default, not yet validated
  against live data — may need adjustment once real consumption history is
  behind it.
- **Con:** η applies to grid charging only; PV-side conversion losses are out of
  scope (the sibling project Effy handles them).
