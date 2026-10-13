# ADR-0010 – Battery-Discharge Planning

**Date:** 2026-10-05 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment, eligible-tier rule, spill-relief slots, file renamed from
`…-window-planning`) **Related ADRs:** ADR-0003 (tier timeline input), ADR-0004
(min SoC, discharge power, no write capability), ADR-0006 (trajectory
simulation), ADR-0011 (`Planner` interface, execution order) **Related
capability:** CAP-09 (`tasks/adr-capability.md`)

---

## Context

The BMS regulates its output to the **grid meter reading** on its own: it serves
the house load and does not export deliberately. A small negative reading is a
coincidence — a race between the meter reading and the BMS's reaction. There is
therefore no export price to plan against and no amount to plan: Chargy only
decides *in which slots the battery is allowed to discharge*. The goals are to
spend stored energy where grid energy is expensive, keep it elsewhere, and avoid
PV that a full battery could not store. Like ADR-0007 this is a plan only;
actuation does not exist yet (ADR-0004 §3). This is the **discharge planner**,
the second planner behind ADR-0011's interface; it lives in
`planner/discharge.py` (ADR-0001 §3).

---

## Decision

### 1 — Discharge-eligible tiers

Slots whose tier (ADR-0003) is a **discharge-eligible tier** — one of the
**highest M tiers** — are claimed as `discharge`, covering every slot of each
such **tier span**. `M` is the configurable `discharge_eligible_tiers`, default
`⌊tier_count / 2⌋`, valid range `1 … tier_count − 1` — the lowest tier is never
discharge-eligible. `M` and the charge planner's `N` (`charge_eligible_tiers`,
ADR-0007 §1) must satisfy `N + M ≤ tier_count`, so no tier is both; the config
flow enforces this and re-validates it whenever the tier strategy or its
parameters change (ADR-0003 §2). Slots already claimed as `charge` are skipped.

### 2 — Hold everywhere else

Once this planner has run, unclaimed slots are `hold` (`unassigned_mode`,
ADR-0011): the load is imported from the grid and the stored energy is kept for
the discharge-eligible tier spans. PV surplus still charges the battery.

### 3 — Spill-relief slots

The planner simulates the **low-consumption scenario** (ADR-0006 §3/§5) — the
one with the most PV spill. If the trajectory shows PV spill, it adds
**spill-relief slots** — discharge slots **before** the earliest spill slot `s`,
taken from the unclaimed slots, **highest price first**, re-simulating after
each, until the spill at `s` is gone or no candidate frees any energy. A
candidate frees `min(load, max_discharge_power × 5/60, E − E_min)` (Wh); one
with no load or a battery at the floor is skipped. The same repeats for later
spill slots. This realizes "also discharge in the cheaper hours of the ramp-up
toward the discharge-eligible tier span, ranked by price, but only if PV would
otherwise spill".

### 4 — Floor and amounts

The floor is the BMS's own `get_min_soc()` (ADR-0004 §1), applied by the
simulation — Chargy imposes no reserve of its own (a fixed value entered in the
config flow only mirrors the BMS's setting, ADR-0004 §1). The output is only a
set of discharge slots; how much the battery delivers in them is the BMS's
decision, following the grid meter reading.

### 5 — Planning only, no actuation

The plan is computed and exposed (discharge blocks, and which slots are
spill-relief slots). Nothing is commanded.

---

## Consequences

- **Pro:** Matches how the hardware actually behaves — slots, not amounts — so
  no export-price or power-setpoint model is needed.
- **Pro:** Simple, explainable rule: discharge when it is expensive, make room
  when PV would spill.
- **Con:** Holding the battery outside the discharge-eligible tier spans means
  importing in cheap and middle slots even while the battery has energy. That is
  the intent, but surprising to anyone who expects plain self-consumption.
- **Con:** The spill-relief slots depend on the PV forecast; a wrong forecast
  causes needless early discharge or missed spill.
- **Con:** Highest-price-first ordering does not guarantee the fewest or most
  contiguous extra slots.
