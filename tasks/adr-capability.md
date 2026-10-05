# Chargy — Capability List (Phase 1, pending Gate 1 approval)

Derived thematically from the Phase-0-reviewed ADRs (`adr/ADR-0000` …
`ADR-0009`, all `Draft`). Capabilities are listed in rough dependency order; the
real dependency graph is finalized in Phase 2. Every capability is end-to-end
demonstrable; none is defined by files or modules.

**Enabling work (not a capability; becomes the first task in Phase 2):** project
scaffold — `custom_components/chargy/` package skeleton, `const.py`, coordinator
shell, config-flow shell, `tests/support.py` / `tests/support_ha.py` (ADR-0001
§7), `tasks/DEPENDENCIES.md`, and the config alignments that `uv sync` and CI
need.

---

## CAP-01 — Price Provider Package

**ADR:** [ADR-0002](../adr/ADR-0002-price-provider-architecture.md)

**Goal:** A normalized 5-minute price curve from a pluggable provider.

**Scope:** Base class (`async get_prices()`, `test_availability()` →
`not_installed` / `not_available` / `available`, `supports_native_tiers()`,
`False` for both providers); shared step-hold normalization to 5-minute slots;
`FixedPriceProvider` (day/night tariff, configurable switch time);
`TibberProvider` (prices from the `tibber.get_prices` action, a user-picked
price entity for availability, a home picker when the response holds several
homes).

**Out of scope:** other providers (EPEX etc.); any tier logic.

**Demonstrable as:** select a provider in the config flow and see a 5-minute
price series exposed as a sensor attribute.

---

## CAP-02 — Tier Strategy Package

**ADR:** [ADR-0003](../adr/ADR-0003-tier-strategy-architecture.md)

**Goal:** Classify a price curve into an ordered list of `(tier, start, end)`
windows over the horizon the price data reaches; every strategy yields at least
3 ranked tiers.

**Scope:** Base class `classify(price_curve, history)`; `FixedBoundaryTier` (≥2
€/kWh cutoffs); `RollingWindowTier` (percentiles, independent backward and
forward lengths; backward part from the recorder via `fetch_fn`).

**Depends on:** CAP-01.

**Out of scope:** what each tier triggers (dispatch policy);
`DerivedFromProviderTier` — no provider reports native tiers, so it is not
implemented in the MVP.

**Demonstrable as:** config flow offers a tier strategy; a tier-plan sensor
shows current and upcoming tier windows, recomputed on price updates.

---

## CAP-03 — BMS Read Package

**ADR:** [ADR-0004](../adr/ADR-0004-bms-read-adapter.md)

**Goal:** Live, read-only battery state from a pluggable BMS adapter.

**Scope:** Base class (`get_soc`, `get_max_soc`, `get_max_capacity`, cached
`get_max_charge_power` / `get_max_discharge_power`, charge/discharge energy
counters, `get_grid_consumed(phase)` / `get_grid_fed(phase)` for up to 3 phases,
`test_availability()`, read-only mapped entity IDs); `SolakonBMS` with
config-flow entity pickers.

**Out of scope:** any write/actuation method; Solakon's built-in grid-charge
logic (disabled).

**Demonstrable as:** SoC, max-SoC, max capacity and per-phase grid-consumed /
grid-fed sensors, sourced live from Solakon.

---

## CAP-04 — Consumption Profile Calculation

**ADR:** [ADR-0008](../adr/ADR-0008-consumption-profile-calculation.md)

**Goal:** Per-phase baseline, and per-phase × weekday × 5-minute-slot average
and standard deviation, from recorder history of signed per-phase Shelly
readings, combined into a whole-house figure for CAP-06.

**Scope:** per-phase Shelly entity pickers; history window `window_days`
(default 28) via the injected `fetch_fn`; gross-consumption correction
`max(0, shelly + bms_fed − bms_consumed)`; value-based histograms with peak
detection (auto mode default, no-peak fallback); per-bucket average/stddev with
the whole-house recombination (variances add); per-phase baseline sensors.

**Depends on:** CAP-03 (mapped entity IDs, per-phase grid sensors).

**Out of scope:** PV production history; holiday handling (CAP-08).

**Demonstrable as:** per-phase baseline sensors; a query/sensor returning the
whole-house average and standard deviation for a weekday + slot.

---

## CAP-05 — Battery Efficiency Estimation

**ADR:** [ADR-0005](../adr/ADR-0005-battery-efficiency-estimation.md)

**Goal:** Round-trip efficiency from the counters' statistics between two SoC
valleys, with a static default (95 %) until two valleys at least 24 hours apart
exist.

**Depends on:** CAP-03.

**Demonstrable as:** an efficiency sensor starting at the default, switching to
a measured value (attribute `measured` / `default`) as history accumulates.

---

## CAP-06 — Energy Gap Calculation

**ADR:** [ADR-0006](../adr/ADR-0006-energy-gap-calculation.md)

**Goal:** Grid-side energy needed to reach the soft target SoC, from current
SoC, capacity, efficiency, expected consumption (plus a confidence-derived
margin) and the externally sourced PV forecast, over the horizon
`min(price end, PV end)`; never above the headroom to the target.

**Depends on:** CAP-03, CAP-04, CAP-05, and the 5-minute PV forecast (external).

**Demonstrable as:** a "grid energy needed" sensor that reacts correctly to
changing SoC, forecast and consumption inputs.

---

## CAP-07 — Grid-Charge Window Planning

**ADR:** [ADR-0007](../adr/ADR-0007-grid-charge-window-planning.md)

**Goal:** Select the cheapest eligible 5-minute slots (lowest `N` tiers,
`charge_eligible_tiers`) that sum to the gap, within the cached max charge
power.

**Depends on:** CAP-02, CAP-06.

**Out of scope:** commanding the BMS; the plan is computed and exposed only.

**Demonstrable as:** a "planned charge windows" sensor listing the selected
slots and total planned energy, recomputed as price or gap changes.

---

## CAP-08 — Holiday Mode

**ADR:** [ADR-0009](../adr/ADR-0009-holiday-mode.md)

**Goal:** While nobody is home, plan against the idle baseline instead of the
learned household profile, and keep holiday periods out of the learned averages.

**Scope:** `ChargyHolidaySwitch` (restores state); in holiday mode expected
consumption = whole-house baseline × 2.0 with standard deviation 0; holiday
slots (from the switch's recorder state history via `fetch_flag_fn`) excluded
from the average/stddev buckets; immediate recompute when the switch flips.

**Depends on:** CAP-04, CAP-06.

**Out of scope:** calendar or automatic detection; a configurable factor.

**Demonstrable as:** toggling the switch changes the "grid energy needed" sensor
immediately; excluding recorded holiday slots measurably leaves the bucket
statistics unchanged by holiday consumption.

---

## Explicitly Deferred

- **BMS write/actuation** — real charge/discharge commands to Solakon (mechanism
  and policy undefined).
- **Tier-based dispatch/discharge policy** — what each tier triggers.
- **Non-Tibber providers** (EPEX etc.) and **`DerivedFromProviderTier`** — until
  a provider with native tiers exists.
- **Additional generation sources per phase**, **automatic holiday detection**,
  **configurable holiday factor**.
- **PV forecasting and PV production history** — external input only.

Config-flow fields (provider, tier strategy + parameters, SoC bounds and soft
target, Solakon entity mapping, Shelly per-phase sensors, `window_days`,
`charge_eligible_tiers`, price entity / home, efficiency default) are not a
capability of their own; they accrete onto the config flow as each capability
lands.

---

## Unvalidated defaults worth confirming at implementation

ADR-0008's thin-bucket smoothing, ADR-0006's confidence/margin formula,
ADR-0005's valley heuristic, ADR-0007's greedy slot fragmentation, and the
holiday factor of 2.0.
