# ADR-0007 – Grid-Charge Window Planning

**Date:** 2026-09-20
**Status:** Draft
**Related ADRs:** ADR-0003 (tier plan input), ADR-0004 (max charge power),
ADR-0006 (energy gap input)
**Related capability:** CAP-07 (`adr/adr-capability.md`)

---

## Context

Once the outstanding energy gap (ADR-0006) is known, Chargy needs to
decide *when* to actually draw it from the grid, at minimum cost.
Grid-charging is restricted to inexpensive periods only — the same
"no battery-to-grid in cheap tiers" principle stated for the (separately
deferred) discharge/grid-support policy implies, symmetrically, that
charging should only happen in those same cheap tiers, never in the
middle-to-expensive tiers reserved for grid support.

---

## Decision

### 1 — Eligible slots

Only slots classified by the active tier strategy (ADR-0003) as the
cheapest tier(s) — cheap / very-cheap — are eligible for grid-charging.

### 2 — Slot selection

Among eligible slots within the horizon the current tier plan covers, the
planner selects slots to cover the outstanding gap (ADR-0006), **cheapest
first**, up to the cached max charge power (ADR-0004 §1) per slot.

### 3 — No fixed deadline

There is no fixed deadline for reaching the target. If the currently
available price/tier horizon doesn't contain enough cheap capacity to
close the gap, the plan simply reflects that the target won't be fully
reached yet, and is recomputed (and potentially extended) as new price
data arrives (ADR-0003 §3).

### 4 — Planning only, no actuation

This capability only computes and exposes the resulting plan (selected
slots and total planned energy). It does not command the BMS — no write
capability exists yet (ADR-0004 §3).

---

## Consequences

- **Pro:** Cheapest-first greedy selection is simple to reason about,
  test, and explain to the user.
- **Pro:** Decoupling planning from actuation means this capability is
  fully testable without touching real hardware.
- **Con:** A pure greedy cheapest-first selection may produce fragmented,
  non-contiguous charge windows. If the eventual dispatch mechanism turns
  out to prefer fewer, contiguous windows — unknown until ADR-0004's
  write extension is designed — this planner may need revision.
