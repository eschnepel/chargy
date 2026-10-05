# ADR-0006 – Energy Gap Calculation

**Date:** 2026-09-20
**Status:** Draft
**Related ADRs:** ADR-0004 (SoC/capacity input), ADR-0005 (efficiency
input), ADR-0008 (consumption input)
**Related capability:** CAP-06 (`adr/adr-capability.md`)

---

## Context

Chargy's core purpose is determining how much grid energy is needed to
bring the battery to its target level, so that grid-charge window
planning (ADR-0007) can decide when to actually draw it. The target
itself is intentionally a **soft**, configurable bound that sits at or
below the BMS's own hard max-SoC ceiling (ADR-0004 §1's `get_max_soc`),
deliberately leaving headroom for PV production the forecast
under-predicted.

---

## Decision

### 1 — Inputs

- Current SoC and max capacity (ADR-0004).
- Round-trip efficiency (ADR-0005).
- Expected consumption per 5-minute slot — the combined whole-house
  average for that weekday+slot bucket (ADR-0008 §4/§6).
- The externally sourced PV forecast at 5-minute resolution (already
  available; producing it is outside Chargy's scope).

### 2 — Target definition

The target is Chargy's configured soft upper-bound SoC. The config flow
constrains it to be **≤ the BMS's own `get_max_soc`** (ADR-0004 §1) —
never higher — so the BMS's own ceiling always wins if the two would
otherwise conflict.

### 3 — Gap formula

```
raw_gap = energy needed to reach the target from current SoC
          − the portion of that gap which expected PV production
            (net of expected consumption) is forecast to cover on its own,
            over the remaining horizon

gap = raw_gap + margin_buffer   (§5)
```

Grid energy only fills what PV production plus the existing charge won't
cover; the margin buffer (§5) then pads that figure for consumption
uncertainty.

### 4 — Continuous recomputation

The gap is recomputed as SoC, PV forecast, or expected consumption
change, at the project's canonical 5-minute resolution. There is no fixed
daily reset or single point-in-time computation — the gap is always the
current, live figure.

### 5 — Confidence-derived margin gap buffer

The consumption input (§1) is ADR-0008's combined whole-house average per
weekday+slot. ADR-0008 also exposes a combined standard deviation for the
same bucket (ADR-0008 §3/§4), which this ADR now uses to size §3's
`margin_buffer`, rather than planning against the bare mean alone:

```
CV[bucket]         = stddev[bucket] / mean[bucket]
confidence[bucket] = 1 / (1 + CV[bucket])
margin[bucket]     = (1 − confidence[bucket]) × stddev[bucket]

margin_buffer = Σ margin[bucket], over every weekday+slot bucket
                corresponding to a slot remaining in the horizon
```

Confidence is bounded in `(0, 1]` and falls as a bucket's variability
grows relative to its own mean; margin scales from `0` (a fully stable
bucket needs no buffer) toward a full standard deviation (a highly
volatile bucket gets padded by roughly its own spread). Guarded against
`mean ≈ 0`: treated as `CV → ∞`, i.e. `confidence → 0`, the most
cautious case, rather than dividing by zero.

This specific formula is **proposed, not validated** the way ADR-0008
§1/§2's peak-detection algorithm was against live data — worth
confirming once implemented.

---

## Consequences

- **Pro:** Decoupling the gap calculation from the exact
  consumption-statistic definition lets the two be developed, reviewed,
  and tested independently.
- **Pro:** Treating the target as strictly bounded by the BMS's own
  ceiling means Chargy can never ask for more than the hardware itself
  permits, regardless of a misconfiguration.
- **Pro:** The margin buffer (§5) means the gap calculation no longer
  silently assumes average conditions will hold — a volatile bucket
  contributes more buffer than a stable one, automatically.
- **Con:** §5's confidence/margin formula is a proposed default, not yet
  validated against live data — may need adjustment once real consumption
  history is behind it.
