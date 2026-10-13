# ADR Summary — architectural ground truth for all workers

Condensed from `adr/ADR-0000` … `ADR-0012` (all `Draft`, glossary alignment
2026-10-12). The ADRs win on any detail not stated here. Domain terms are
defined in `GLOSSARY.md` (ADR-0001 §8) — use them, and avoid the words it lists
under _Avoid_. Units (ADR-0001 §9): energy **Wh**, power **W**, SoC **%**,
prices **€/kWh** after the price unit factor; sensors in other units are
normalised at the boundary.

## Tech stack

- Home Assistant custom integration, HACS-distributed: package
  `custom_components/chargy/`, entity-class prefix `Chargy` (ADR-0001 §1).
- Python ≥3.14, Home Assistant ≥2026.3.1 (ADR-0001 §2). `numpy>=1.26.0` is
  pre-declared in `pyproject.toml`.
- Gates (ADR-0000 §1): `ruff format`, `ruff check`, `mypy --strict`
  (`mypy custom_components/chargy tests --config-file mypy.ini`), `pytest`; CI
  runs pre-commit (incl. `mdformat`) + pytest. Config is UI-only (config flow,
  no YAML); user-facing errors are translation keys.
- Style: `from __future__ import annotations`, built-in generics, `X | None`,
  full signatures (incl. `-> None`), `@dataclass` for data containers, constants
  `UPPER_SNAKE` in `const.py`, private helpers `_prefixed`, one concept per
  module.

## Architecture

- Layering, dependencies point upward only (ADR-0000 §3, ADR-0001 §3): pure
  calculation modules → `coordinator.py` → `sensor.py` / `switch.py` /
  `config_flow.py` / `__init__.py`. Provisional layout: `price_provider/`,
  `tier/`, `bms/`, `consumption.py`, `efficiency.py`, `standby.py`, and the
  `planner/` package (`context.py`, `simulation.py`, `charge.py`,
  `discharge.py`; empty `__init__.py`, modules import each other by full
  submodule path to avoid an import cycle).
- **Pure tier** (zero HA imports, zero-mocking tests): `tier/`, `planner/`,
  `efficiency.py`, `standby.py`, `consumption.py`, base classes of
  `price_provider/` and `bms/`. Narrow exceptions that touch HA:
  `price_provider/tibber.py` (`hass.states` + the `tibber.get_prices` action)
  and `bms/solakon.py` (`hass.states`).
- **Recorder rule:** only `coordinator.py` touches the recorder. History reaches
  pure modules via an injected sync
  `fetch_fn(sensor_id, start, end, statistic) -> list[float | None]`, one entry
  per 5-minute slot (`statistic` = `"mean"` or `"sum"`; `"sum"` for
  `TOTAL_INCREASING` counters). Sensors need a `state_class`. A second injected
  `fetch_flag_fn(entity_id, start, end) -> list[bool | None]` (recorder *state*
  history; `True` if `on` at any moment in the slot, `None` = no history) serves
  on/off entities such as the holiday switch.
- **Planners (ADR-0011):** immutable `PlanContext` in `planner/context.py`
  (planning-horizon slots, prices, tiers, PV energy, consumption band
  (high/mean/low), standby power, BMS bounds, claims, `unassigned_mode`,
  `report`); abstract `Planner.plan(context) -> PlanContext`, pure, never
  changes earlier claims; config `planner_order` ∈ `discharge_first` (default) |
  `charge_first`. No registry or pluggable planner implementations. The
  coordinator resolves standby power (picked → fixed → ADR-0012 estimate) and
  passes it in.
- No persistence: everything history-derived is rebuilt from the recorder at
  startup. Cadence (provisional): prices hourly + startup; 5-minute tick for
  live BMS read and **replan** (gap + planners); history-derived stats
  (consumption, efficiency, standby) rebuilt at startup and daily after
  midnight; flipping the holiday switch re-derives the statistics from held
  history and replans immediately.
- Shared config (ADR-0001 §5): `window_days` (default 28) = the **history
  window** (ends now, today included) for consumption averages/deviation,
  efficiency and standby estimation; `baseline_days` (default 7) = the
  **baseline window**; `planner_order`; `soft_target_soc` (%, default 80,
  validated between Min SoC and Max SoC). A **day** is exactly 24 hours.

## Capability decisions

- **Price provider (0002):** `async get_prices(start, end)` → the price curve:
  5-minute `(timestamp, price)` in €/kWh; native resolution (hourly/15-min) is
  step-held flat, never interpolated. Every provider step has a **price unit**
  dropdown (`price_factor`: € / kWh × 1 = default, Cent / kWh ÷ 100) applied to
  all prices the provider handles. `test_availability()` ∈ `not_installed` /
  `not_available` / `available`; `supports_native_tiers()` is `False` for both
  providers. `FixedPriceProvider` (day/night, switch time); `TibberProvider`:
  prices from the `tibber.get_prices` action (`start_time`/`price` per home;
  home picker if >1), plus a user-picked price entity for availability and
  recorder price history.
- **Tier strategy (0003):** `classify(price_curve, history)` → the tier
  timeline: ordered `(tier, start, end)` tier spans over exactly the price
  horizon; tier = opaque label with integer rank (0 = cheapest); **every
  strategy yields ≥3 tiers**; charge-eligible N + discharge-eligible M ≤ tier
  count (config flow enforces and re-validates). `FixedBoundaryTier` (≥2 €/kWh
  cutoffs, always in €/kWh), `RollingPercentileTier` (percentile window with
  independent backward/forward lengths; backward part is `history`). Rebuilt on
  price update.
- **BMS read (0004):** read-only base class: `get_soc`, `get_max_soc`,
  `get_min_soc` (the BMS's discharge reserve), `get_capacity` (live, required
  picker), `get_standby_power`, cached `get_max_charge_power` /
  `get_max_discharge_power` (cache refreshed when capacity changes),
  charge/discharge energy counters (any sensor with a `state_class`, including
  Effy's effective sensors), `get_grid_draw(phase)` / `get_grid_feed(phase)` (≤3
  phases L1/L2/L3, user phase mapping), `test_availability()`, read-only mapped
  entity IDs. **Value sources:** Min SoC, Max SoC, max charge power, max
  discharge power and standby power each come from an entity picker, or when it
  is empty from a fixed value (% / W) in the config flow; exactly one source per
  value, picker wins; standby power may also be **auto** (`-1`, default →
  ADR-0012). `SolakonBMS` via config-flow entity pickers. The BMS regulates its
  output to the grid meter reading by itself.
- **Consumption (0008):** per-phase signed grid meter readings (entity picker
  per phase), corrected
  `max(0, grid_meter_reading + bms_grid_feed − bms_grid_draw)`; raw values are
  kernel-smoothed over neighbor slots (`smoothing_radius`, default 2), then
  assigned to value-based bins (`bin_width`, default 5 W); baseline per phase
  over the baseline window via histogram peak detection with a falloff region
  (diagnostic; also the holiday-mode expectation); average/stddev per bucket
  (phase × local weekday × local 5-min slot; the local clock decides on DST
  days) over the history window, each reading also contributing to neighbor
  slots by kernel weight, stddev from the raw values; combined to whole-house
  (means add, variances add).
- **Efficiency (0005):** round-trip efficiency from charge/discharge counter
  `sum` statistics between an early and a recent SoC valley (24 h valley search
  windows, valley must lie in the middle 18 h, 6 h shifts); stored-energy
  difference uses the capacity at each valley separately; the energy standby
  power consumes (`standby_power × Δt`, effective value) is subtracted from the
  energy in; static default **95 %** if no valleys ≥24 h apart or numbers
  unusable; clamp `(0, 1]`. Battery path only; PV-side conversion losses are out
  of scope (Effy).
- **Standby power (0012):** only in auto (`-1`): standby power = Σ energy drop
  over idle runs / Σ run duration within the history window. Idle run =
  consecutive slots with grid draw and grid feed ≈ 0 on every phase (tolerance 2
  W) and SoC never rising, ≥3 h and ≥2 SoC points (proposed, unvalidated); until
  one qualifies the 10 W placeholder is used; sensor exposes
  `picked`/`fixed`/`measured`/`default`; `standby.py` pure.
- **Trajectory and gap (0006):** 5-minute simulation
  `simulate(context, scenario)` (`planner/simulation.py`) over the planning
  horizon = earlier of price horizon and PV forecast horizon; slot modes
  `charge` / `discharge` / `hold` (unclaimed slots: `discharge` until the
  discharge planner ran, then `hold`); PV surplus charges in every mode up to
  `E_max = max_soc × capacity`, excess is `spill`; in `discharge` slots the
  battery covers load down to `E_min = min_soc × capacity`, the rest is
  `shortfall`; standby power is drawn every slot, so SoC may fall below `E_min`
  (clamped at 0); `E_target` (soft target SoC) caps grid charging only; per-slot
  W → Wh = W×5/60; PV-side conversion losses out of scope (Effy); consumption
  band from per-bucket confidence `1/(1+CV)`: `cons_high = mean + band`,
  `cons_low = max(0, mean − band)` (proposed, unvalidated);
  `gap = Σ shortfall / η` under the high scenario (grid-side, before any charge
  claim; a requirement, not a promise). Charge planner uses high, discharge
  planner low.
- **Charge planner (0007, `planner/charge.py`):** charge-eligible tiers = the
  lowest `N` by rank (`charge_eligible_tiers`, default `⌊count/2⌋`, highest
  never eligible); for the earliest shortfall pick the cheapest unclaimed
  eligible slot *before* it, up to `max_charge_power × 5/60` Wh, keeping
  `E ≤ E_target` and not increasing spill; repeat; shortfalls under the
  high-consumption scenario, the spill check under the low one; report
  `unmet_gap`; plan only.
- **Discharge planner (0010, `planner/discharge.py`):** discharge-eligible tiers
  = the highest `M` (`discharge_eligible_tiers`, default `⌊count/2⌋`, lowest
  never eligible) are claimed as `discharge`; everything else `hold`; if the
  low-consumption simulation shows PV spill, add spill-relief slots before the
  earliest spill slot, highest price first, until the spill is gone; floor = BMS
  `get_min_soc`; slots only, no amounts, no export price; plan only.
- **Holiday mode (0009):** `ChargyHolidaySwitch` (`switch.py`, off by default,
  restores state); while on, expected consumption = Σ phase baselines × the
  holiday baseline factor `HOLIDAY_BASELINE_FACTOR` (constant 2.0) with stddev
  0, for every slot; all consumption statistics weight slots whose holiday flag
  differs from the switch state by `HOLIDAY_OTHER_FLAG_WEIGHT` (constant 0.2)
  instead of dropping them; if a configured phase has no baseline, the normal
  profile is used and a warning logged.

## Non-functional requirements

- Canonical resolution 5 minutes everywhere.
- Pure calculation modules raise no exceptions in normal range — they clamp;
  guard `mean ≈ 0`, empty history, non-positive denominators.
- Background failures (polls, action calls) are logged and swallowed, never
  raised; provider failure degrades to `not_available`.
- Tests: pure tier zero-mocking; HA-touching modules and the HA-facing layer use
  a hand-written `homeassistant` stub in `sys.modules` + real (non-`Mock`)
  `Fake*` classes in `tests/support.py` / `tests/support_ha.py`; modules loaded
  by file path with the `TYPE_CHECKING` static-import pattern; each test class
  documents its scenario; invariants asserted explicitly; each planner tested
  alone with a hand-built `PlanContext`.
- mypy suppressions are per-file (`mypy.ini`) with coded `# type: ignore[...]`
  on the exact reported line; never global.
- Rationale lives in ADRs, not long docstrings; `adr/INDEX.md` is updated in the
  same commit as any ADR status/structure change; diagrams are Mermaid, tables
  native Markdown.

## Explicit exclusions

- No BMS write/actuation methods — not even abstract — and no use of Solakon's
  built-in grid-charge logic (disabled). Both planners only plan.
- No deliberate grid export, no export price, no discharge amounts: the BMS
  follows the grid meter reading itself.
- No PV forecasting or PV production history in Chargy; the forecast is an
  external input.
- No direct vendor API calls (Tibber only via its HA integration); no other
  providers (EPEX etc.) and no simultaneous providers in the MVP.
- `DerivedFromProviderTier` is not implemented (no provider has native tiers).
- The grid meter (Shelly) is a recorder-history source only — no live reads, no
  control.
- No additional generation sources per phase in the MVP.
- PV-side conversion losses are out of scope (the sibling project Effy handles
  them); Chargy has no dependency on Effy.
- No automatic holiday detection or calendar; the holiday factor (2.0) and
  weight (0.2) are constants, not config options, in the MVP.
- No planner registry or pluggable planner implementations; no persistence
  layer/`Store`.
- No real HA instance or `pytest-homeassistant-custom-component` in tests; no
  interpolation of prices; no bare/global `type: ignore`.
- No PV-charging planner (PV charging is BMS default behavior and has no lever
  until actuation exists).
