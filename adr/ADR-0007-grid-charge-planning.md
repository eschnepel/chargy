# ADR-0007 – Grid-Charge Planning

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment, eligible-tier rule, file renamed from `…-window-planning`) **Related
ADRs:** ADR-0003 (tier timeline input), ADR-0004 (max charge power), ADR-0006
(trajectory and gap), ADR-0011 (`Planner` interface) **Related capability:**
CAP-07 (`tasks/adr-capability.md`)

---

## Context

Once the slot-wise trajectory (ADR-0006) shows where the battery would run
short, Chargy needs to decide *when* to draw that energy from the grid, at
minimum cost and early enough to help. Grid-charging is restricted to
inexpensive periods: the discharge planner (ADR-0010) confines discharging to
the **discharge-eligible tiers**, and symmetrically charging only happens in the
**charge-eligible tiers**, never in the middle-to-expensive ones. This is the
**charge planner**, one of the two planners behind the common interface of
ADR-0011; it lives in `planner/charge.py` (ADR-0001 §3).

---

## Decision

### 1 — Eligible slots

Only slots whose tier (ADR-0003) is a **charge-eligible tier** — one of the
**lowest N tiers** — are eligible for grid-charging. `N` is the configurable
`charge_eligible_tiers`, default `⌊tier_count / 2⌋` (3 tiers → the lowest only;
5 tiers → the two cheapest), valid range `1 … tier_count − 1` — the highest tier
is never charge-eligible. Because ADR-0003 §2 guarantees at least three tiers, a
neutral middle band always exists, and the default keeps it out of charging. `N`
and the discharge planner's `M` (`discharge_eligible_tiers`, ADR-0010 §1) must
satisfy `N + M ≤ tier_count`, so no tier is both charge- and discharge-eligible.
The config flow enforces this and re-validates it whenever the tier strategy or
its parameters change (ADR-0003 §2). Slots already claimed by the discharge
planner are not available.

### 2 — Slot selection against the trajectory

The planner works on the simulation (ADR-0006 §3):

1. Simulate under the **high-consumption scenario** (ADR-0006 §3/§5) and take
   the earliest slot `j` with a shortfall.
1. Candidates are the unclaimed eligible slots **before** `j`; pick the cheapest
   (ties: the earliest).
1. Claim it as `charge` with grid energy up to `max_charge_power × 5/60` Wh
   (ADR-0004 §1), limited so the battery does not exceed `E_target` (the soft
   target SoC, ADR-0006 §2) after that slot, and rejected if the
   **low-consumption** simulation shows more total spill than before (then the
   next candidate is tried).
1. Re-simulate and repeat for `j` until its shortfall is covered or no candidate
   remains, then continue with the next shortfall.

The grid energy planned is therefore bounded by the gap (ADR-0006 §3, a
**grid-side** figure already divided by efficiency) and by what can still reach
the shortfall in time. Using the high scenario for the shortfall and the low one
for the spill check errs on the side of enough energy without charging into
**spill**.

### 3 — No fixed deadline

There is no fixed deadline for reaching the soft target SoC. If the currently
available price horizon doesn't contain enough charge-eligible capacity to close
the gap, the plan reports the **unmet gap** (`unmet_gap`) and is replanned (and
potentially extended) as new price data arrives (ADR-0003 §3).

### 4 — Planning only, no actuation

This capability only computes and exposes the resulting plan (charge blocks,
total planned energy, unmet gap). It does not command the BMS — no write
capability exists yet (ADR-0004 §3).

---

## Consequences

- **Pro:** Cheapest-first greedy selection is simple to reason about, test, and
  explain to the user.
- **Pro:** Charging is anchored to the shortfall it serves, so cheap slots
  *after* a shortfall are never planned.
- **Pro:** Decoupling planning from actuation means this capability is fully
  testable without touching real hardware.
- **Con:** A per-shortfall greedy pass is not a global optimum, and may produce
  fragmented charge blocks. If the eventual dispatch mechanism turns out to
  prefer fewer, longer blocks — unknown until ADR-0004's write extension is
  designed — this planner may need revision.
