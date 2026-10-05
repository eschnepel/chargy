# ADR-0003 – Tier Strategy Architecture

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-05 **Related ADRs:**
ADR-0002 (price curve input), ADR-0001 §6 (history via `fetch_fn`) **Related
capability:** CAP-02 (`adr/adr-capability.md`)

---

## Context

Eventual battery behavior (grid-support/discharge, and grid-charging — ADR-0007)
needs to know which price "tier" (very cheap … very expensive) a given moment
falls into, and needs to know it *ahead of time*, not only at the current
instant, so that future capabilities can plan windows rather than only react.
The number of tiers and the exact rules for what each tier triggers are
deliberately undecided and deferred — this ADR covers only the mechanism for
classifying prices into an ordered, time-windowed tier plan, independent of what
that plan is later used for.

---

## Decision

### 1 — Base class contract

`classify(price_curve, history)` → an ordered list of `(tier, start, end)`
windows, spanning exactly the horizon the input `price_curve` (ADR-0002) covers
— no independent planning horizon is introduced at this layer. `history` is an
optional earlier price series (possibly empty) used only as statistical context
by strategies that need it (`RollingWindowTier`); it is never classified itself
and never extends the output.

### 2 — Tier-count-agnostic, with a floor of three

A tier is an opaque label with an integer **rank** (0 = cheapest, ascending in
price). Strategies decide how many tiers exist and what they are called; nothing
downstream assumes a fixed number such as five.

The one hard rule: **every strategy must produce at least 3 tiers** — a lowest,
a highest and at least one middle tier — so a neutral band always exists between
the cheapest and the most expensive. Config-flow validation enforces this per
strategy (e.g. `FixedBoundaryTier` needs at least two cutoffs,
`RollingWindowTier` at least two percentile boundaries).

### 3 — Recomputation trigger

The tier plan is recomputed whenever the underlying price curve updates (i.e. on
a price-provider data update), not on a separate fixed polling schedule.

### 4 — Implementations in scope

- **`DerivedFromProviderTier`** — would use a provider's own native tier
  classification, where one is reported for the planning horizon with at least
  three levels. Only offered as a config-flow choice when the active provider's
  `supports_native_tiers()` (ADR-0002 §1) is true. **No provider in this MVP
  supports it** (ADR-0002 §5: the Tibber integration documents only price
  values), so the class is defined by the architecture but **not implemented in
  the MVP**; it becomes a task once a provider with native tiers is added.
- **`FixedBoundaryTier`** — user-configured absolute €/kWh cutoffs between tiers
  (at least two, per §2).
- **`RollingWindowTier`** — percentile-based classification computed over a
  window of price data spanning a **backward (historical) length** and a
  separate **forward (known-future) length**, each independently configurable
  (e.g. 3 days backward, 1 day forward) rather than a single symmetric window —
  a provider that has already published tomorrow's prices should be able to fold
  them into the percentile distribution without that changing how far back
  history is drawn from, and vice versa. The forward part is the `price_curve`
  itself; the backward part is the `history` argument, which the coordinator
  assembles from the recorder (ADR-0002 §5, ADR-0001 §6). Independent of any
  specific provider, so it is the strategy that works for every provider,
  including a future EPEX one.

### 5 — Classification vs. policy

Tier **classification** (this ADR) and tier **policy** (what action a given tier
should trigger) are separate concerns. This ADR governs only classification;
policy is explicitly out of scope and left for a future ADR once the underlying
rules are defined.

---

## Consequences

- **Pro:** A forward-looking, windowed tier plan lets any future dispatch policy
  reason about upcoming price behavior, not just the current instant.
- **Pro:** `RollingWindowTier`'s provider-independence makes it immediately
  reusable for a future EPEX (or other) provider without modification.
- **Con:** Because policy is undefined, this capability alone produces no
  user-visible battery behavior — it only exposes classified data for later
  capabilities to consume.
