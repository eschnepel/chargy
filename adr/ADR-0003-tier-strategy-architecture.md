# ADR-0003 – Tier Strategy Architecture

**Date:** 2026-09-20
**Status:** Draft
**Related ADRs:** ADR-0002 (price curve input)
**Related capability:** CAP-02 (`adr/adr-capability.md`)

---

## Context

Eventual battery behavior (grid-support/discharge, and grid-charging —
ADR-0007) needs to know which price "tier" (very cheap … very expensive)
a given moment falls into, and needs to know it *ahead of time*, not only
at the current instant, so that future capabilities can plan windows
rather than only react. The number of tiers and the exact rules for what
each tier triggers are deliberately undecided and deferred — this ADR
covers only the mechanism for classifying prices into an ordered,
time-windowed tier plan, independent of what that plan is later used for.

---

## Decision

### 1 — Base class contract

`classify(price_curve)` → an ordered list of `(tier, start, end)` windows,
spanning exactly the horizon the input price curve (ADR-0002) covers — no
independent planning horizon is introduced at this layer.

### 2 — Tier-count-agnostic

The contract does not assume any fixed number of tiers (not hardcoded to
five). A tier is an opaque, ordered label; strategies decide how many
exist and their names.

### 3 — Recomputation trigger

The tier plan is recomputed whenever the underlying price curve updates
(i.e. on a price-provider data update), not on a separate fixed polling
schedule.

### 4 — Implementations in scope

- **`DerivedFromProviderTier`** — uses the active provider's own native
  tier classification, where present. Only offered as a config-flow
  choice when the active provider's `supports_native_tiers()` (ADR-0002
  §1) is true.
- **`FixedBoundaryTier`** — user-configured absolute €/kWh cutoffs between
  tiers.
- **`RollingWindowTier`** — percentile-based classification computed over
  a window of price data spanning a **backward (historical) length** and
  a separate **forward (known-future) length**, each independently
  configurable (e.g. 3 days backward, 1 day forward) rather than a single
  symmetric window — a provider that has already published tomorrow's
  prices should be able to fold them into the percentile distribution
  without that changing how far back history is drawn from, and vice
  versa. Independent of any specific provider, so it remains usable for a
  provider without native tier support (e.g. a future EPEX provider).

### 5 — Classification vs. policy

Tier **classification** (this ADR) and tier **policy** (what action a
given tier should trigger) are separate concerns. This ADR governs only
classification; policy is explicitly out of scope and left for a future
ADR once the underlying rules are defined.

---

## Consequences

- **Pro:** A forward-looking, windowed tier plan lets any future dispatch
  policy reason about upcoming price behavior, not just the current
  instant.
- **Pro:** `RollingWindowTier`'s provider-independence makes it
  immediately reusable for a future EPEX (or other) provider without
  modification.
- **Con:** Because policy is undefined, this capability alone produces no
  user-visible battery behavior — it only exposes classified data for
  later capabilities to consume.
