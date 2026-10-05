# ADR-0002 – Price Provider Architecture

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-05 **Related ADRs:**
ADR-0001 §6 (`fetch_fn` for recorder history) **Related capability:** CAP-01
(`tasks/adr-capability.md`)

---

## Context

Chargy needs a normalized dynamic-price curve to plan grid-charging against.
Several tariff types exist (fixed day/night, Tibber, EPEX, …), and the user's
own tariff (Tibber) should not be hardcoded into the rest of the system — the
project must stay portable to other tariffs without touching downstream logic.
Providers read from whichever HA integration already exposes the data ("take
what is there") rather than calling a vendor's API directly, so Chargy inherits
whatever authentication/setup the user already has for that integration.

For Tibber specifically, Home Assistant's integration exposes the *current*
price as a sensor (with `min_price`/`max_price` attributes for the day) and the
price curve through the response-returning action `tibber.get_prices`: per
Tibber home, a list of `start_time` (local time) / `price` (energy + taxes)
entries for a requested date range, hourly or — as the action's own example
shows — 15-minute. It documents no price-level or tier data.

---

## Decision

### 1 — Base class contract

- `async get_prices(start, end)` → an ordered series of `(timestamp, price)`,
  normalized to Chargy's canonical 5-minute resolution. Asynchronous because a
  provider may have to await a Home Assistant action call (§5); the step-hold
  normalization (§3) itself is a shared pure function.
- `test_availability()` → one of three states:
  - `not_installed` — the underlying HA integration (or, for an action-based
    provider, the action it calls) is not present at all.
  - `not_available` — present but not currently yielding usable data (not
    authenticated, entity missing/unavailable, empty response, or a startup race
    before the source integration has populated its own state).
  - `available` — usable price data can be read right now.
- `supports_native_tiers()` → whether the source also exposes its own price-tier
  classification covering the planning horizon, consumable directly by a
  `DerivedFromProviderTier` strategy (ADR-0003). Both providers in scope (§5)
  return `False`.

### 2 — Single active provider

Exactly one provider is active per Chargy install, selected in the config flow.
Multiple simultaneous providers are not supported.

### 3 — Resolution normalization

A provider's native resolution (hourly, or 15-minute as Tibber's action may
return) is **step-held flat** down into Chargy's canonical 5-minute slots — no
interpolation between price points.

### 4 — Config-flow availability behavior

- At initial setup, only providers currently reporting `available` are offered
  as selectable choices.
- During reconfiguration (options flow), a provider that has since become
  unavailable is still shown, accompanied by an explanatory message, rather than
  silently hidden or removed from the choice list.
- A provider may bring its own config-flow fields: `FixedPriceProvider` its two
  prices and switch time; `TibberProvider` a price entity (entity picker) and,
  when the action's response holds more than one Tibber home, which home to use
  (§5).

### 5 — Implementations in scope

- **`FixedPriceProvider`** — a two-level day/night tariff with a configurable
  time-of-day switch point. Always reports `available` (no external dependency).
- **`TibberProvider`** — built on the official Tibber Home Assistant
  integration; never calls Tibber's API directly.
  - *Forward and current prices* come from the action `tibber.get_prices`,
    called with a start/end range from today through as far ahead as Tibber has
    published. The provider returns exactly what the response contains, so the
    price horizon (ADR-0003 §1) is whatever Tibber currently offers. The `price`
    field is used as reported (energy + taxes, per kWh).
  - *The picked price entity* (config-flow entity picker, Tibber's current-price
    sensor) serves two purposes: it is part of the availability check, and it is
    the recorder-history source for backward-looking price statistics
    (`RollingWindowTier`'s backward window, ADR-0003 §4), read via the injected
    `fetch_fn` (ADR-0001 §6). `tibber.get_prices` also accepts a past `start`,
    which is the documented fallback if the entity turns out to hold no recorder
    statistics — to be verified against the real installation at implementation.
  - *Availability:* `not_installed` if the `tibber.get_prices` action is not
    registered; `not_available` if the picked entity is missing/unavailable or
    the action returns no price data; `available` otherwise.
  - Calling an action for its response is covered by ADR-0000 §3's narrow
    Home-Assistant exception for adapter modules.

---

## Out of Scope / Deferred

- Other providers (EPEX or similar) — the base class supports them, but no
  further implementation is part of this ADR.
- Any tier/price-level classification logic — see ADR-0003.

---

## Consequences

- **Pro:** Adding a new tariff later means writing one new base-class
  implementation; no downstream capability (tier classification, gap
  calculation, charge-window planning) needs to change.
- **Pro:** Reusing whichever integration is already installed means Chargy never
  stores or manages its own third-party credentials.
- **Con:** `TibberProvider` depends on a cloud-polling integration; a failed or
  empty action call must degrade to `not_available` (logged, never raised —
  ADR-0000 §8), so availability can flicker with Tibber's own connectivity.
- **Con:** Step-hold normalization discards any true intra-hour price shape a
  finer-grained future tariff might expose — an accepted trade-off given the
  project's canonical 5-minute resolution.
