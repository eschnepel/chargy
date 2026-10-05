# ADR-0002 – Price Provider Architecture

**Date:** 2026-09-20
**Status:** Draft
**Related capability:** CAP-01 (`adr/adr-capability.md`)

---

## Context

Chargy needs a normalized dynamic-price curve to plan grid-charging
against. Several tariff types exist (fixed day/night, Tibber, EPEX, …),
and the user's own tariff (Tibber) should not be hardcoded into the rest
of the system — the project must stay portable to other tariffs without
touching downstream logic. Providers read from whichever HA integration
already exposes the data ("take what is there") rather than calling a
vendor's API directly, so Chargy inherits whatever authentication/setup
the user already has for that integration.

---

## Decision

### 1 — Base class contract

- `get_prices(start, end)` → an ordered series of `(timestamp, price)`,
  normalized to Chargy's canonical 5-minute resolution.
- `test_availability()` → one of three states:
  - `not_installed` — the underlying HA integration this provider wraps
    is not present at all.
  - `not_available` — the integration is present but not currently
    yielding usable data (not authenticated, entity missing/unavailable,
    or a startup race before the source integration has populated its
    own state).
  - `available` — usable price data can be read right now.
- `supports_native_tiers()` → whether this provider's underlying source
  also exposes its own price-tier/level classification (e.g. Tibber's
  price levels), consumable directly by a `DerivedFromProviderTier`
  strategy (ADR-0003).

### 2 — Single active provider

Exactly one provider is active per Chargy install, selected in the config
flow. Multiple simultaneous providers are not supported.

### 3 — Resolution normalization

A provider's native resolution (e.g. hourly) is **step-held flat** down
into Chargy's canonical 5-minute slots — no interpolation between price
points.

### 4 — Config-flow availability behavior

- At initial setup, only providers currently reporting `available` are
  offered as selectable choices.
- During reconfiguration (options flow), a provider that has since become
  unavailable is still shown, accompanied by an explanatory message,
  rather than silently hidden or removed from the choice list.

### 5 — Implementations in scope

- **`FixedPriceProvider`** — a two-level day/night tariff with a
  configurable time-of-day switch point. Always reports `available` (no
  external dependency).
- **`TibberProvider`** — reads price data from the official Tibber Home
  Assistant integration's own entities/attributes; does not call Tibber's
  API directly.

---

## Out of Scope / Deferred

- Other providers (EPEX or similar) — the base class supports them, but
  no further implementation is part of this ADR.
- Any tier/price-level classification logic — see ADR-0003.

---

## Consequences

- **Pro:** Adding a new tariff later means writing one new base-class
  implementation; no downstream capability (tier classification, gap
  calculation, charge-window planning) needs to change.
- **Pro:** Reusing whichever integration is already installed means
  Chargy never stores or manages its own third-party credentials.
- **Con:** Step-hold normalization discards any true intra-hour price
  shape a finer-grained future tariff might expose — an accepted
  trade-off given the project's canonical 5-minute resolution.
