# ADR-0004 – BMS Read Adapter Architecture

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment, value sources) **Related capability:** CAP-03
(`tasks/adr-capability.md`)

---

## Context

Chargy needs live battery state — state of charge, capacity, power limits,
energy counters — from the **BMS** to compute its plans. The first deployment's
hardware is a Solakon ONE, integrated into Home Assistant via a community
integration that exposes both read-only sensors and writable config entities.
Solakon's own built-in "charge from grid when cheap" logic was found
insufficient for price-optimized charging and has been switched off by the user;
consequently, any future command/write capability will be Chargy's own
responsibility, not Solakon's built-in scheduler's.

The concrete write mechanism and the dispatch policy that would drive it are
explicitly **not yet decided**. This ADR governs only the read side and the
package/base-class skeleton, so it can be implemented and tested now without
speculatively designing a write contract that would likely need revision once
policy is defined.

Separately, Solakon also exposes its own **grid sensors** — up to 3, one per
phase, each reporting bidirectional flow at the BMS's own grid connection point
(its **grid draw** and its **grid feed**). These are not something Chargy
computes or infers; they are a direct hardware read, on the same footing as SoC
or the energy counters in §1, and are what "BMS support" concretely refers to.

The BMS also regulates its own output to the **grid meter reading** (a Shelly in
the first deployment): it serves the house load by itself, and Chargy never
commands amounts — it can at most decide in which slots discharging is allowed
(ADR-0010).

---

## Decision

### 1 — Base class contract (read-only)

Units follow ADR-0001 §9: SoC in %, power in W, energy and capacity in Wh.

- `get_soc()`
- `get_max_soc()` — **Max SoC**, the BMS's ceiling. See ADR-0006 for how the
  **soft target SoC** relates to it.
- `get_min_soc()` — **Min SoC**, the BMS's discharge reserve (the lowest SoC it
  will discharge to). It is the floor of the slot-wise simulation (ADR-0006).
  Chargy imposes no reserve of its own: where the BMS does not expose it, the
  user enters the value the BMS is set to (*Value sources* below).
- `get_max_charge_power()` / `get_max_discharge_power()` — **max charge power**
  and **max discharge power**. A value read from an entity is read once and
  **cached**, treated as effectively fixed for planning purposes. Because a
  battery extension can change them, the cache is refreshed whenever the live
  `get_capacity` reading differs from the previous one. A fixed value (see
  *Value sources* below) needs no cache.
- `get_standby_power()` — **standby power** in W, the power the battery system
  itself draws to keep running, in every mode and even below Min SoC. Returns
  the picked or fixed value, or `None` when the user left it on **auto** (stored
  as `-1`, the default): the coordinator then uses the estimate of ADR-0012. It
  feeds ADR-0006's simulation and is subtracted in ADR-0005 §4.
- `get_capacity()` — **capacity**, read live, not cached, since usable capacity
  can drift with degradation over time or change step-wise when the battery is
  extended (ADR-0005 §4 reads its history at the two SoC valleys). Always a
  required entity picker; it has no fixed-value fallback.
- `get_charge_energy_counter()` / `get_discharge_energy_counter()` — cumulative
  counters, consumed by ADR-0005's efficiency estimation. Any sensor with a
  `state_class` and a convertible unit may be picked, including the
  loss-corrected sensors of the sibling project Effy; which one is picked
  decides whether the round-trip efficiency includes the conversion losses Effy
  models (ADR-0005 §2). Chargy has no dependency on Effy.
- `test_availability()` — mirrors the Price Provider pattern (ADR-0002 §1):
  `not_installed` / `not_available` / `available`.
- Mapped entity IDs, read-only — the adapter exposes the entity IDs it was
  configured with (SoC, capacity, both energy counters, the per-phase grid
  sensors of §4) so history consumers (ADR-0005, ADR-0008, ADR-0012) can query
  the recorder for them through the injected `fetch_fn` (ADR-0001 §6). The
  adapter itself never reads history.

**Value sources.** Five values — Min SoC, Max SoC, max charge power, max
discharge power and standby power — each come from either an entity picker (read
live, or cached as above) or, when the picker is left empty, a fixed value
entered in the config flow (% for SoC, W for power), for BMS hardware or
integrations that do not expose them. Per value exactly one source is required;
if both are set, the picker wins. Standby power additionally allows neither
(auto, `-1`, the default). Config keys: `min_soc`, `max_soc`,
`max_charge_power`, `max_discharge_power`, `standby_power` hold the fixed value;
`<key>_entity` holds the picked entity.

### 2 — Entity mapping

The concrete `SolakonBMS` implementation maps to entity IDs chosen by the user
in the config flow (entity pickers, plus the fixed-value fields of §1), rather
than hardcoded `entity_id` strings — this is a community integration whose
entity IDs are not guaranteed stable across installs or versions.

### 3 — Write/actuation explicitly excluded

No write or actuation methods are declared on the base class, even as
unimplemented abstracts. They will be added once a future ADR defines the
dispatch mechanism and policy — adding them speculatively now risks guessing at
a shape that policy decisions would likely invalidate.

### 4 — Per-phase grid sensors (bidirectional)

- `get_grid_draw(phase)` / `get_grid_feed(phase)` — the BMS's own measurement,
  at its grid connection point, of its **grid draw** (power taken from the grid)
  and its **grid feed** (power given to the grid) on a given phase. Not
  household consumption (GLOSSARY.md).
- Exposed for however many phases the concrete implementation actually has
  mapped — up to 3 — not a requirement that all 3 exist; a
  single-phase-connected system like Solakon ONE populates one.
- **Phase mapping:** the config flow lets the user map each of these sensors to
  the corresponding **phase** (the same L1/L2/L3 labeling used by the grid meter
  in ADR-0008's per-phase histograms), following the same entity-picker pattern
  as §2 — Solakon's own entity labels are not assumed to already agree with the
  grid meter's.
- **Relationship to §1's counters:** these are a separate, complementary data
  source, measured at the grid connection point rather than at the battery
  itself. They do not replace §1's whole-battery charge/discharge energy
  counters, which remain the basis for ADR-0005's efficiency estimation.
  Historical values of these sensors (ADR-0008 §5), like SoC and the counters
  (ADR-0005), are read from the recorder's statistics, so all of them must carry
  a `state_class`.

---

## Consequences

- **Pro:** Live battery state can be exposed and the gap calculation and charge
  planning (ADR-0006/ADR-0007) can be developed and tested against real data
  immediately, without waiting on dispatch-policy decisions.
- **Pro:** Not speculatively designing the write contract avoids committing to
  an interface shape before the actual triggering rules exist.
- **Pro:** §4's per-phase grid sensors give an exact, direct correction for the
  gross-consumption bias the BMS's discharge introduces into the grid meter
  readings on its wired phase (see ADR-0008 §5), rather than needing to infer it
  indirectly from the whole-battery counters.
- **Con:** Because writes are absent, this capability alone cannot close the
  loop (compute a plan and act on it) — a later, currently undesigned extension
  of this same package is required for that.
