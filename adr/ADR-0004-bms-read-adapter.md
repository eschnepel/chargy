# ADR-0004 – BMS Read Adapter Architecture

**Date:** 2026-09-20 **Status:** Draft **Related capability:** CAP-03
(`adr/adr-capability.md`)

---

## Context

Chargy needs live battery state — state of charge, capacity, power limits,
energy counters — from the physical battery management system to compute its
plans. The first deployment's hardware is a Solakon ONE, integrated into Home
Assistant via a community integration that exposes both read-only sensors and
writable config entities. Solakon's own built-in "charge from grid when cheap"
logic was found insufficient for price-optimized charging and has been switched
off by the user; consequently, any future command/write capability will be
Chargy's own responsibility, not Solakon's built-in scheduler's.

The concrete write mechanism and the dispatch policy that would drive it are
explicitly **not yet decided**. This ADR governs only the read side and the
package/base-class skeleton, so it can be implemented and tested now without
speculatively designing a write contract that would likely need revision once
policy is defined.

Separately, Solakon also exposes its own **grid sensors** — up to 3, one per
phase, each reporting bidirectional flow at the BMS's own grid connection point
(how much it is drawing from the grid, and how much it is feeding back). These
are not something Chargy computes or infers; they are a direct hardware read, on
the same footing as SoC or the energy counters in §1, and are what "BMS support"
concretely refers to.

---

## Decision

### 1 — Base class contract (read-only)

- `get_soc()`
- `get_max_soc()` — the BMS's own configured ceiling, read live as a sensor (not
  a Chargy config value). See ADR-0006 for how Chargy's own target relates to
  this.
- `get_max_capacity()` — read live, not cached, since usable capacity can drift
  with degradation over time.
- `get_max_charge_power()` / `get_max_discharge_power()` — read once and
  **cached**, treated as effectively fixed for planning purposes.
- `get_charge_energy_counter()` / `get_discharge_energy_counter()` — cumulative
  counters, consumed by ADR-0005's efficiency estimation.
- `test_availability()` — mirrors the Price Provider pattern (ADR-0002 §1):
  `not_installed` / `not_available` / `available`.

### 2 — Entity mapping

The concrete `SolakonBMS` implementation maps to entity IDs chosen by the user
in the config flow (entity pickers), rather than hardcoded `entity_id` strings —
this is a community integration whose entity IDs are not guaranteed stable
across installs or versions.

### 3 — Write/actuation explicitly excluded

No write or actuation methods are declared on the base class, even as
unimplemented abstracts. They will be added once a future ADR defines the
dispatch mechanism and policy — adding them speculatively now risks guessing at
a shape that policy decisions would likely invalidate.

### 4 — Per-phase grid sensors (bidirectional)

- `get_grid_consumed(phase)` / `get_grid_fed(phase)` — the BMS's own
  measurement, at its grid connection point, of power currently drawn from the
  grid ("consumed") and fed back to the grid ("fed") on a given phase.
- Exposed for however many phases the concrete implementation actually has
  mapped — up to 3 — not a requirement that all 3 exist; a
  single-phase-connected system like Solakon ONE populates one.
- **Phase mapping:** the config flow lets the user map each of these sensors to
  the corresponding physical phase (the same L1/L2/L3 labeling used by the
  Shelly meter in ADR-0008's per-phase histograms), following the same
  entity-picker pattern as §2 — Solakon's own entity labels are not assumed to
  already agree with the Shelly's.
- **Relationship to §1's counters:** these are a separate, complementary data
  source, measured at the grid connection point rather than at the battery
  itself. They do not replace §1's whole-battery charge/discharge energy
  counters, which remain the basis for ADR-0005's efficiency estimation.

---

## Consequences

- **Pro:** Live battery state can be exposed and the gap/schedule calculations
  (ADR-0006/ADR-0007) can be developed and tested against real data immediately,
  without waiting on dispatch-policy decisions.
- **Pro:** Not speculatively designing the write contract avoids committing to
  an interface shape before the actual triggering rules exist.
- **Pro:** §4's per-phase grid sensors give an exact, direct correction for the
  gross-consumption bias the BMS's discharge introduces into the Shelly readings
  on its wired phase (see ADR-0008 §5), rather than needing to infer it
  indirectly from the whole-battery counters.
- **Con:** Because writes are absent, this capability alone cannot close the
  loop (compute a plan and act on it) — a later, currently undesigned extension
  of this same package is required for that.
