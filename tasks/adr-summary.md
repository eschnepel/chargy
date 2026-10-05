# ADR Summary — architectural ground truth for all workers

Condensed from `adr/ADR-0000` … `ADR-0009` (all `Draft`, Phase 0 review
2026-10-05). The ADRs win on any detail not stated here.

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
  `config_flow.py` / `__init__.py`. Provisional module layout:
  `price_provider/`, `tier/`, `bms/`, `consumption.py`, `efficiency.py`,
  `gap.py`, `schedule.py`.
- **Pure tier** (zero HA imports, zero-mocking tests): `tier/`, `gap.py`,
  `schedule.py`, `efficiency.py`, `consumption.py`, base classes of
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
- Pluggable strategies behind base classes: price provider, tier strategy, BMS
  adapter. Exactly one active provider.
- No persistence: everything history-derived is rebuilt from the recorder at
  startup. Cadence (provisional): prices hourly + startup; 5-minute tick for
  live BMS read, gap and plan; history-derived stats at startup and daily after
  midnight; flipping the holiday switch recomputes gap and plan immediately.
- Shared config: `window_days` (default 28) = recorder history analysed by
  ADR-0005/0008 (ADR-0001 §5).

## Capability decisions

- **Price provider (0002):** `async get_prices(start, end)` → 5-minute
  `(timestamp, price)`; native resolution (hourly/15-min) is step-held flat,
  never interpolated. `test_availability()` ∈ `not_installed` / `not_available`
  / `available`; `supports_native_tiers()` is `False` for both providers.
  `FixedPriceProvider` (day/night, switch time); `TibberProvider`: prices from
  the `tibber.get_prices` action (`start_time`/`price` per home; home picker if
  > 1), plus a user-picked price entity for availability and recorder price
  > history.
- **Tier strategy (0003):** `classify(price_curve, history)` → ordered
  `(tier, start, end)` over exactly the price-curve horizon; tier = opaque label
  with integer rank (0 = cheapest); **every strategy yields ≥3 tiers**.
  `FixedBoundaryTier` (≥2 €/kWh cutoffs), `RollingWindowTier` (percentiles;
  independent backward/forward lengths; backward part is `history`). Recomputed
  on price update.
- **BMS read (0004):** read-only base class: `get_soc`, `get_max_soc` (live),
  `get_max_capacity` (live), cached `get_max_charge_power` /
  `get_max_discharge_power`, charge/discharge energy counters,
  `get_grid_consumed(phase)` / `get_grid_fed(phase)` (≤3 phases, user phase
  mapping), `test_availability()`, read-only mapped entity IDs. `SolakonBMS` via
  config-flow entity pickers.
- **Consumption (0008):** per-phase signed Shelly readings (entity picker per
  phase), corrected `max(0, shelly + bms_fed − bms_consumed)`; baseline per
  phase via histogram peak detection (diagnostic; also the holiday-mode
  expectation, ADR-0009); average/stddev per phase × weekday × 5-min slot,
  combined to whole-house (means add, variances add); thin-bucket smoothing is
  an implementation detail. Slots in which holiday mode was active are excluded
  from the average/stddev buckets (not from the baseline histogram).
- **Efficiency (0005):** from charge/discharge counter `sum` statistics between
  an early and a recent SoC valley (24 h windows, valley must lie in the middle
  18 h, 6 h shifts); SoC-difference corrected; static default **95 %** if no
  valleys ≥24 h apart or numbers unusable; clamp `(0, 1]`.
- **Gap (0006):** `effective_target = min(target, get_max_soc())`; aggregate
  over horizon `min(price end, PV end)`; per-slot W → kWh = W×5/60/1000;
  `gap_battery = min(headroom, max(0, E_target − E_now − E_net + margin))`;
  grid-side `gap = gap_battery / η`; margin from per-bucket confidence
  `1/(1+CV)` (proposed, unvalidated); in holiday mode the consumption input is
  the holiday figure with stddev 0, so the margin is 0.
- **Window planning (0007):** eligible = lowest `N` tiers by rank
  (`charge_eligible_tiers`, default `⌊count/2⌋`, highest never eligible);
  cheapest-first greedy up to `max_charge_power × 5 min` per slot; plan only, no
  actuation; no fixed deadline.
- **Holiday mode (0009):** `ChargyHolidaySwitch` (`switch.py`, off by default,
  restores state); while on, expected consumption = Σ phase baselines ×
  `HOLIDAY_BASELINE_FACTOR` (constant 2.0) with stddev 0, for every slot;
  holiday slots are kept out of the §3 statistics; if a configured phase has no
  baseline, the normal profile is used and a warning logged.

## Non-functional requirements

- Canonical resolution 5 minutes everywhere.
- Pure calculation modules raise no exceptions in normal range — they clamp;
  guard `mean ≈ 0`, empty history, non-positive denominators.
- Background failures (polls, action calls) are logged and swallowed, never
  raised; provider failure degrades to `not_available`.
- Tests: pure tier zero-mocking; HA-touching modules and HA-facing layer use a
  hand-written `homeassistant` stub in `sys.modules` + real (non-`Mock`) `Fake*`
  classes in `tests/support.py` / `tests/support_ha.py`; modules loaded by file
  path with the `TYPE_CHECKING` static-import pattern; each test class documents
  its scenario; invariants asserted explicitly.
- mypy suppressions are per-file (`mypy.ini`) with coded `# type: ignore[...]`
  on the exact reported line; never global.
- Rationale lives in ADRs, not long docstrings; `adr/INDEX.md` is updated in the
  same commit as any ADR status/structure change; diagrams are Mermaid, tables
  native Markdown.

## Explicit exclusions

- No BMS write/actuation methods — not even abstract — and no use of Solakon's
  built-in grid-charge logic (disabled).
- No tier-based dispatch/discharge policy; tiers are classified only.
- No PV forecasting or PV production history in Chargy; the forecast is an
  external input.
- No direct vendor API calls (Tibber only via its HA integration); no other
  providers (EPEX etc.) and no simultaneous providers in the MVP.
- `DerivedFromProviderTier` is not implemented (no provider has native tiers).
- Shelly is a recorder-history source only — no live reads, no control.
- No additional generation sources per phase in the MVP.
- No persistence layer/`Store`; no real HA instance or
  `pytest-homeassistant-custom-component` in tests; no interpolation of prices;
  no bare/global `type: ignore`.
- No automatic holiday detection or calendar; the holiday factor (2.0) is a
  constant, not a config option, in the MVP.
