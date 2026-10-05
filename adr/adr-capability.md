# Chargy — Draft Capability List (Phase 1, pending Gate 1 approval)

Derived from brainstorming, not yet from finalized ADRs (Chargy has no numbered
ADRs beyond ADR-0000 yet). This list is the input to the ADR set and, later, to
`tasks/<slug>.md` generation — it is expected to shift as ADRs are actually
written and as Phase 0 surfaces gaps or contradictions. Capabilities are listed
in rough dependency order; the real dependency graph is finalized in Phase 2.

---

## CAP-01 — Price Provider Package

**Derived ADR:** [ADR-0002](ADR-0002-price-provider-architecture.md) *(Draft)*

**Goal:** A normalized 5-minute price curve, sourced from a pluggable provider.

**Scope:** Base class (`get_prices()`, `test_availability()` → `not_installed` /
`not_available` / `available`, `supports_native_tiers()`); `FixedPriceProvider`
(two-level day/night tariff, configurable switch time); `TibberProvider` (reads
the official Tibber HA integration's entities — providers read from whatever HA
integration already exposes the data, never a direct external API).
Source-resolution prices are step-held flat into 5-minute slots (no
interpolation).

**Out of scope:** EPEX or other providers (architecture supports them,
implementation deferred); any tier logic.

**Demonstrable as:** select a provider in the config flow, see a 5-minute
normalized price series exposed as a sensor/attribute.

---

## CAP-02 — Tier Strategy Package

**Derived ADR:** [ADR-0003](ADR-0003-tier-strategy-architecture.md) *(Draft)*

**Goal:** Classify a price curve into a time-ordered array of tier windows
(`[(tier, start, end), ...]`), covering the horizon the price data actually
reaches.

**Scope:** Base class, tier-count-agnostic; `DerivedFromProviderTier` (uses a
provider's own native tier classification, where supported); `FixedBoundaryTier`
(configurable €/kWh cutoffs); `RollingWindowTier` (percentile-based,
provider-agnostic, with independently configurable backward and forward window
lengths — the default/fallback for providers without native tiers).

**Depends on:** CAP-01 (price curve). `DerivedFromProviderTier` is only offered
in the config flow when the selected provider's `supports_native_tiers()` is
true.

**Out of scope:** what each tier triggers (dispatch policy — deferred
indefinitely, to be defined later).

**Demonstrable as:** config flow offers a tier strategy (filtered by provider
capability), a tier-plan sensor shows the current and upcoming tier windows,
recomputed whenever price data updates.

---

## CAP-03 — BMS Read Package

**Derived ADR:** [ADR-0004](ADR-0004-bms-read-adapter.md) *(Draft)*

**Goal:** Live, read-only battery state from a pluggable BMS adapter.

**Scope:** Base class (`get_soc`, `get_max_soc` \[the BMS's own configured
ceiling\], `get_max_capacity`, `get_max_charge_power` /
`get_max_discharge_power` [cached once, not re-polled per cycle],
`get_charge_energy_counter`, `get_discharge_energy_counter`,
`get_grid_consumed(phase)` / `get_grid_fed(phase)` for up to 3 phases \[the
BMS's own bidirectional grid-connection sensors, phase-mapped via config flow\],
`test_availability()`); `SolakonBMS` implementation with user-mapped entity IDs
(config-flow entity pickers, not hardcoded entity_ids, since this is a community
integration where IDs can drift between installs).

**Out of scope:** any write/actuation method. Solakon's own built-in grid-charge
logic is not used (disabled) because it isn't flexible enough for
price-optimized charging — but the mechanism and policy for Chargy to issue
write commands itself is a future capability, not this one.

**Demonstrable as:** SoC, the BMS's own max-SoC ceiling, max capacity, and
per-phase grid-consumed/grid-fed sensors, exposed as Chargy sensors, sourced
live from Solakon.

---

## CAP-04 — Consumption Profile Calculation

**Derived ADR:** [ADR-0008](ADR-0008-consumption-profile-calculation.md) *(Draft
— all originally-flagged Open Questions now resolved; baseline's role and the
confidence/margin extension to ADR-0006 are proposed, unvalidated defaults worth
confirming at implementation)*

**Goal:** Per-phase baseline, and per-phase/weekday/slot average and standard
deviation, of grid-import consumption — derived from historical 3-phase Shelly
readings and combined into a whole-house figure for CAP-06.

**Scope:** Per-phase, value-based histograms of historical readings, using
fixed-width bins anchored at 0W (default 5W, configurable), smoothed by a
configurable distance-weighted neighbor average (default radius 2) before peak
detection; baseline **defaults to auto mode** (`cap = 0`): the mean of
everything from 0W through the first confirmed peak — a local maximum whose
following valley region dips more than `fall_threshold`% below it anywhere in
that region, not just the immediately next bin (default 20%) — with `cap > 0`
available as a plain percentage-cutoff alternative. Validated against live
per-phase power histograms. Pooled across the phase's whole history regardless
of weekday/slot; average and standard deviation = computed per phase × weekday ×
5-minute slot; the three phases' average and standard deviation are recombined
into a single whole-house figure per weekday+slot for CAP-06 to consume
(variances summed under an independence assumption, not standard deviations
summed directly); gross-consumption correction on the BMS's wired phase clamped
to a minimum of zero.

**Out of scope:** PV production history (Solakon's PV string counters are
explicitly out of scope for Chargy — PV forecasting is handled by an existing,
separate integration plus the user's own calculation layer on top of it).

**Demonstrable as:** per-phase baseline sensors; a query/sensor returning the
combined whole-house average and standard deviation for any given weekday +
slot.

---

## CAP-05 — Battery Efficiency Estimation

**Derived ADR:** [ADR-0005](ADR-0005-battery-efficiency-estimation.md) *(Draft)*

**Goal:** Round-trip charge efficiency, derived over time from the BMS's
charge/discharge energy counters, with a static configurable default used until
enough counter history has accumulated.

**Depends on:** CAP-03.

**Demonstrable as:** an efficiency sensor starting at the configured default,
updating as counter history grows.

---

## CAP-06 — Energy Gap Calculation

**Derived ADR:** [ADR-0006](ADR-0006-energy-gap-calculation.md) *(Draft —
consumes ADR-0008's combined whole-house average and, via a confidence figure,
its standard deviation too, see ADR-0006 §5)*

**Goal:** How much grid energy is needed to reach Chargy's target SoC, given
current SoC, capacity, efficiency, expected consumption (plus a
confidence-derived margin buffer), and the PV forecast — recomputed continuously
across the 5-minute horizon. The target is a **soft upper bound**, configurable,
intentionally allowed to sit below the BMS's own hard max-SoC ceiling (CAP-03's
`get_max_soc`) so there is headroom left for PV production the forecast didn't
capture.

**Depends on:** CAP-03, CAP-04, CAP-05, plus the (already available, externally
sourced) 5-minute PV forecast.

**Demonstrable as:** a "grid energy needed" sensor that reacts correctly to
changing SoC, forecast, and consumption inputs.

---

## CAP-07 — Grid-Charge Window Planning

**Derived ADR:** [ADR-0007](ADR-0007-grid-charge-window-planning.md) *(Draft)*

**Goal:** Given the energy gap (CAP-06) and the tier plan (CAP-02), select the
cheapest allowed 5-minute slots — cheap/very-cheap tiers only — that sum to the
required energy, respecting the cached max charge power (CAP-03).

**Depends on:** CAP-02, CAP-06.

**Out of scope:** actually commanding the BMS to charge. No BMS write capability
exists yet (see CAP-03's exclusions) — this capability only computes and exposes
the plan.

**Demonstrable as:** a "planned charge windows" sensor listing the selected
slots and total planned energy, recomputed as price or gap changes.

---

## Explicitly Deferred (flagged so they are not designed prematurely)

- **BMS write/actuation capability** — issuing real charge/discharge commands to
  Solakon. Mechanism (which config entities to write, what domain/service) and
  policy (when/how much) are both undefined.
- **Tier-based grid-support/discharge dispatch policy** — the actual rule set
  for what each price tier triggers. Explicitly "rules will be defined later"
  per project brainstorming.
- **Shelly** — confirmed no role in Chargy. It is a grid meter the BMS already
  reads on its own for self-regulation; Chargy has no reason to touch it.
- **Non-Tibber price providers** (EPEX via Nordpool or similar) — architecture
  supports them (CAP-01's base class), implementation deferred.

Config-flow fields (provider choice, tier strategy + its parameters, SoC bounds
and the soft-target margin, Solakon entity mapping, day/night prices, the
efficiency default) are not a capability of their own — they accrete onto the
config flow as each capability above is implemented.

---

## Open items to resolve before/while Phase 0 formally begins

No capability-blocking open items remain — every capability above now has a
derived ADR. Several individual ADRs still carry their own flagged open
questions or proposed-but-unvalidated defaults that don't block their
architecture but should be settled/confirmed before or shortly after
implementation:

- **ADR-0008 / ADR-0006** — all of ADR-0008's originally-flagged open questions
  are resolved, but two resolutions are proposed defaults, not validated against
  live data the way §1/§2's peak-detection algorithm was: the exact smoothing
  extension for thin weekday+slot buckets (ADR-0008 §3), and the
  confidence/margin formula itself (ADR-0006 §5).
- **ADR-0005** (Battery Efficiency Estimation) — the precise definition of
  "enough history" and how counter resets/outliers are handled.
- **ADR-0007** (Grid-Charge Window Planning) — whether greedy cheapest-first
  slot selection is acceptable if the eventual BMS write mechanism turns out to
  prefer contiguous windows.
