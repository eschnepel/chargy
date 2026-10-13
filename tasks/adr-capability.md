# Chargy — Capability List (Phase 1, pending Gate 1 approval)

Derived thematically from the Phase-0-reviewed ADRs (`adr/ADR-0000` …
`ADR-0012`, all `Draft`). Capabilities are listed in rough dependency order; the
real dependency graph is finalized in Phase 2. Every capability is end-to-end
demonstrable; none is defined by files or modules. Terms follow `GLOSSARY.md`.

**Changed 2026-10-12 (glossary alignment):** CAP-10 added (standby power
estimation, ADR-0012) — new scope, to be confirmed at Gate 1; CAP-07 and CAP-09
renamed (no more "window planning").

**Enabling work (not a capability; becomes the first task in Phase 2):** project
scaffold — `custom_components/chargy/` package skeleton, `const.py`, coordinator
shell, config-flow shell, `tests/support.py` / `tests/support_ha.py` (ADR-0001
§7), `tasks/DEPENDENCIES.md`, and the config alignments that `uv sync` and CI
need (`hacs.json` domains, `mypy.ini` for `switch`).

---

## CAP-01 — Price Provider Package

**ADR:** [ADR-0002](../adr/ADR-0002-price-provider-architecture.md)

**Goal:** A normalized 5-minute **price curve** (€/kWh) from a pluggable price
provider.

**Scope:** Base class (`async get_prices()`, `test_availability()` →
`not_installed` / `not_available` / `available`, `supports_native_tiers()`,
`False` for both providers); shared step-hold normalization to 5-minute slots;
`FixedPriceProvider` (day/night tariff, configurable switch time);
`TibberProvider` (prices from the `tibber.get_prices` action, a user-picked
price entity for availability, a home picker when the response holds several
homes); a **price unit** dropdown per provider (€ / kWh × 1, default, or Cent /
kWh ÷ 100) applied to all prices the provider handles.

**Out of scope:** other providers (EPEX etc.); any tier logic.

**Demonstrable as:** select a provider in the config flow and see a 5-minute
price series exposed as a sensor attribute.

---

## CAP-02 — Tier Strategy Package

**ADR:** [ADR-0003](../adr/ADR-0003-tier-strategy-architecture.md)

**Goal:** Classify a price curve into the **tier timeline** — an ordered list of
`(tier, start, end)` tier spans over the price horizon; every strategy yields at
least 3 ranked tiers.

**Scope:** Base class `classify(price_curve, history)`; `FixedBoundaryTier` (≥2
€/kWh cutoffs); `RollingPercentileTier` (percentiles over a percentile window
with independent backward and forward lengths; backward part from the recorder
via `fetch_fn`); config-flow rule that charge-eligible N + discharge-eligible M
≤ tier count, re-validated when the strategy changes.

**Depends on:** CAP-01.

**Out of scope:** `DerivedFromProviderTier` — no provider reports native tiers,
so it is not implemented in the MVP.

**Demonstrable as:** config flow offers a tier strategy; a tier-timeline sensor
shows current and upcoming tier spans, rebuilt on price updates.

---

## CAP-03 — BMS Read Package

**ADR:** [ADR-0004](../adr/ADR-0004-bms-read-adapter.md)

**Goal:** Live, read-only battery state from a pluggable BMS adapter.

**Scope:** Base class (`get_soc`, `get_max_soc`, `get_min_soc`,
`get_standby_power`, `get_capacity`, cached `get_max_charge_power` /
`get_max_discharge_power` refreshed when the capacity changes, charge/discharge
energy counters (any sensor with a `state_class`, Effy's effective sensors
included), `get_grid_draw(phase)` / `get_grid_feed(phase)` for up to 3 phases
L1/L2/L3, `test_availability()`, read-only mapped entity IDs); `SolakonBMS` with
config-flow entity pickers. Min SoC, Max SoC, max charge power, max discharge
power and standby power each come from an entity picker or, when it is empty,
from a fixed value (% / W) entered in the config flow — one source per value,
picker wins; standby power may also stay on auto (`-1`, default, CAP-10).

**Out of scope:** any write/actuation method; Solakon's built-in grid-charge
logic (disabled).

**Demonstrable as:** SoC, Min/Max SoC, capacity and per-phase grid-draw /
grid-feed sensors, sourced live from Solakon (or from the fixed fallback
values).

---

## CAP-04 — Consumption Profile Calculation

**ADR:** [ADR-0008](../adr/ADR-0008-consumption-profile-calculation.md)

**Goal:** Per-phase baseline, and per-phase × weekday × 5-minute-slot average
and standard deviation (**expected consumption** per bucket), from recorder
history of signed per-phase grid meter readings, combined into a whole-house
figure for CAP-06.

**Scope:** per-phase grid meter entity pickers; the baseline window
(`baseline_days`, default 7) and the history window (`window_days`, default 28)
via the injected `fetch_fn`; gross-consumption correction
`max(0, grid_meter_reading + bms_grid_feed − bms_grid_draw)`; kernel smoothing
of the raw values over neighbor slots (`smoothing_radius`) before the
value-based histogram (`bin_width`); peak detection with a falloff region (auto
mode default, no-peak fallback); per-bucket weighted average/stddev (buckets
keyed by local weekday and local time of day, DST by local clock) with
neighbor-slot kernel borrowing and the whole-house recombination (variances
add); per-phase baseline sensors.

**Depends on:** CAP-03 (mapped entity IDs, per-phase grid sensors).

**Out of scope:** PV production history; the holiday weighting (CAP-08).

**Demonstrable as:** per-phase baseline sensors; a query/sensor returning the
whole-house average and standard deviation for a weekday + slot.

---

## CAP-05 — Battery Efficiency Estimation

**ADR:** [ADR-0005](../adr/ADR-0005-battery-efficiency-estimation.md)

**Goal:** Round-trip efficiency from the counters' statistics between two SoC
valleys (found in 24-hour valley search windows; start and end capacity applied
separately), with a static default (95 %) until two valleys at least 24 hours
apart exist.

**Depends on:** CAP-03; the effective standby power (CAP-10, or the fixed or
placeholder value until CAP-10 lands).

**Demonstrable as:** an efficiency sensor starting at the default, switching to
a measured value (attribute `measured` / `default`) as history accumulates.

---

## CAP-06 — Slot-wise Energy Trajectory and Gap

**ADR:** [ADR-0006](../adr/ADR-0006-energy-gap-calculation.md)

**Goal:** A 5-minute simulation of the battery over the planning horizon (the
earlier of price horizon and PV forecast horizon) — the trajectory of stored
energy, PV spill and shortfall per slot, with standby power (the SoC may fall
below Min SoC overnight) — for a higher and a lower scenario (a
confidence-derived consumption band), and from the high scenario the grid-side
gap ("grid energy needed"). Energy in Wh; PV-side conversion losses are out of
scope (Effy).

**Depends on:** CAP-03, CAP-04, CAP-05, CAP-10 (effective standby power), and
the 5-minute PV forecast (external).

**Demonstrable as:** a "grid energy needed" sensor and a trajectory attribute
that react correctly to changing SoC, forecast and consumption inputs.

---

## CAP-07 — Grid-Charge Planning

**ADRs:** [ADR-0007](../adr/ADR-0007-grid-charge-planning.md),
[ADR-0011](../adr/ADR-0011-planner-interface-and-order.md)

**Goal:** Place charge slots in the charge-eligible tiers
(`charge_eligible_tiers`), cheapest first, each before the shortfall it serves,
within the cached max charge power and the soft target SoC (`soft_target_soc`,
default 80 %), without increasing PV spill. Shortfalls are evaluated under the
high-consumption scenario, the spill check under the low one.

**Scope:** the charge planner (`planner/charge.py`); the `Planner` base class
and `PlanContext` (`planner/context.py`) and the simulation
(`planner/simulation.py`) shared with CAP-09.

**Depends on:** CAP-02, CAP-06.

**Out of scope:** commanding the BMS; the plan is computed and exposed only.

**Demonstrable as:** a "planned charge blocks" sensor listing the charge blocks,
planned energy and any unmet gap, replanned as price or trajectory change.

---

## CAP-08 — Holiday Mode

**ADR:** [ADR-0009](../adr/ADR-0009-holiday-mode.md)

**Goal:** While nobody is home, plan against the idle baseline instead of the
learned household profile, and let the statistics adapt quickly to the switch
state.

**Scope:** `ChargyHolidaySwitch` (restores state); in holiday mode expected
consumption = whole-house baseline × the holiday baseline factor (2.0) with
standard deviation 0; all consumption statistics weight slots whose holiday flag
differs from the switch state by 0.2 (flags from the switch's recorder state
history via `fetch_flag_fn`); immediate replan when the switch flips.

**Depends on:** CAP-04, CAP-06.

**Out of scope:** calendar or automatic detection; a configurable factor or
weight.

**Demonstrable as:** toggling the switch changes the "grid energy needed" sensor
immediately, and the baseline adapts within a day or two of holiday data.

---

## CAP-09 — Battery-Discharge Planning

**ADRs:** [ADR-0010](../adr/ADR-0010-battery-discharge-planning.md),
[ADR-0011](../adr/ADR-0011-planner-interface-and-order.md)

**Goal:** Mark when the battery may discharge: the discharge-eligible-tier slots
(`discharge_eligible_tiers`), plus spill-relief slots (cheaper ramp-up slots,
highest price first) only where the low-consumption simulation shows PV spill;
hold elsewhere. Make the order of the charge and discharge planners
configurable.

**Scope:** the discharge planner (`planner/discharge.py`); the `planner_order`
option (`discharge_first` default, `charge_first`).

**Depends on:** CAP-02, CAP-03, CAP-06, CAP-07.

**Out of scope:** commanding the BMS; export prices and discharge amounts (the
BMS follows the grid meter reading itself).

**Demonstrable as:** a "planned discharge blocks" sensor showing the discharge
blocks (spill-relief slots marked), changing with price, PV forecast and the
planner order.

---

## CAP-10 — Standby Power Estimation *(added 2026-10-12, pending Gate 1)*

**ADR:** [ADR-0012](../adr/ADR-0012-standby-power-estimation.md)

**Goal:** Derive standby power from the installation's own history when the user
leaves it on auto (`-1`, default): the SoC drop during idle runs (grid draw and
grid feed ≈ 0 on every phase, SoC never rising), within the history window.

**Scope:** `standby.py` (pure), the coordinator's resolution order (picked →
fixed → auto), the 10 W placeholder until a qualifying idle run exists, and a
standby-power sensor with a `picked` / `fixed` / `measured` / `default`
attribute.

**Depends on:** CAP-03. Consumed by CAP-05 and CAP-06.

**Out of scope:** a configurable idle-run threshold (constants in the MVP).

**Demonstrable as:** the standby-power sensor starting at the placeholder and
switching to `measured` after a qualifying idle night.

---

## Explicitly Deferred

- **BMS write/actuation** — real charge/discharge commands to Solakon (mechanism
  undefined).
- **Non-Tibber providers** (EPEX etc.) and **`DerivedFromProviderTier`** — until
  a provider with native tiers exists.
- **Additional generation sources per phase**, **automatic holiday detection**,
  **configurable holiday factor and weight**.
- **PV forecasting and PV production history** — external input only.
- **A PV-charging planner** — PV surplus charging is the BMS's default, modelled
  by the simulation; it becomes a planner when a lever exists (actuation).
- **Deliberate grid export** and export pricing — the BMS regulates to the grid
  meter reading itself.
- **PV-side conversion losses** — handled by the sibling project Effy.

Config-flow fields (provider and price unit, tier strategy + parameters, soft
target SoC, Solakon entity mapping with the picker-or-fixed-value fields for Min
SoC, Max SoC, max charge/discharge power and standby power (incl. auto), grid
meter per-phase sensors, `window_days`, `baseline_days`,
`charge_eligible_tiers`, `discharge_eligible_tiers` (N + M ≤ tier count),
`planner_order`, price entity / home, efficiency default) are not a capability
of their own; they accrete onto the config flow as each capability lands.

---

## Unvalidated defaults worth confirming at implementation

ADR-0008's kernel and peak detection on smoothed raw values, ADR-0006's
confidence/band formula, the standby power placeholder (10 W) and ADR-0012's
idle-run thresholds (2 W, 3 h, 2 SoC points), the soft target SoC default (80
%), ADR-0005's valley heuristic, the planners' greedy selection, and the holiday
baseline factor (2.0) and weight (0.2).
