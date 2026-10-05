# ADR-0008 – Consumption Profile Calculation (Baseline, Average, Deviation)

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-05 **Related ADRs:**
ADR-0001 §5/§6 (`window_days`, recorder access via `fetch_fn` /
`fetch_flag_fn`), ADR-0004 §4 (per-phase grid sensors, used for the correction
in §5), ADR-0006 (the consumer of this capability's output), ADR-0009 (holiday
mode: baseline reuse and history exclusion) **Related capability:** CAP-04
(`tasks/adr-capability.md`)

---

## Context

Chargy needs household consumption figures to feed the whole-house energy gap
calculation (ADR-0006). The source data is 3-phase grid-import history from the
Shelly 3EM grid meter (the same meter the BMS already reads for its own
self-regulation — see ADR-0004's context). The readings are **signed** per-phase
net grid power (import positive, export negative) and are taken from the Home
Assistant recorder (§7), not read live. Phases are kept statistically separate
during construction, since mixing genuinely distinct per-phase load patterns
together would blur the resulting statistics — but the figure ADR-0006
ultimately needs is a single whole-house one, so the per-phase results are
recombined for that purpose.

---

## Decision

### 1 — Per-phase histogram

For each of the 3 phases independently, build a **value-based** histogram of
historical wattage readings for that phase — after the §5 correction, so
non-negative — over the configured history window (`window_days`, ADR-0001 §5;
default 28 days), using **fixed-width bins** anchored at 0W (bin edges at
`0, bucket_width, 2×bucket_width, ...` up to the observed maximum) — not bins
scaled or normalized to the observed range. `bucket_width` is configurable,
default **5W**, validated against live data.

An earlier version of this design also considered a "by count" mode (bin edges
chosen so each bin holds a roughly equal number of samples). Dropped:
equal-count binning distorts the value axis so a bin's position no longer
corresponds to a stable wattage range, which defeats the point of using a
histogram to locate distinct load clusters (e.g. a fridge's idle vs. active
draw) in the first place — value-based binning is the only mode that actually
serves §2's peak-detection use.

### 2 — Baseline: one figure per phase

Baseline is a single value per phase (3 total), derived from §1's histogram via
a configurable cutoff, `cap`, **defaulting to 0 (auto mode)**:

- **`cap` = 0 (auto mode, default)** — detect the histogram's distinct local
  peaks (clusters) and take the mean of the **first** one's cluster as the
  baseline (see the algorithm below). More robust than a fixed percentage cutoff
  whenever the low-value cluster's actual share of total samples varies — e.g. a
  fridge that spends more or less than 20% of its time idle would otherwise pull
  a fixed-percentage cutoff away from the true idle-state value.
- **`cap` > 0 (percentage cutoff)** — mean of the histogram's lowest `cap`% of
  samples by value.

Pooled across the phase's entire history window regardless of weekday or time of
day, per the earlier reasoning (idle/always-on load isn't meaningfully weekday-
or time-of-day-dependent).

**Smoothing, applied before peak detection:** each bin's count is replaced with
a distance-weighted average of itself and its `smoothing_radius` nearest
neighbors on each side (config, default **2** — i.e. a 5-bin window by default;
`0` disables smoothing). Proposed weighting — flagged for confirmation, not yet
validated the way the core algorithm below is: weight at distance `d` =
`1 / (d + 1)` (so the bin itself counts most, each neighbor less the further
away), normalized to sum to 1 across the window, truncated at the histogram's
edges rather than wrapping or padding. Smoothing only affects *where* the
peak-detection algorithm below locates peaks and valleys; the final baseline
mean in step 4 is always computed from genuine, unsmoothed raw sample values.

**Peak-detection method for auto mode — validated against live data** (see the
Shelly/Solakon power-histogram chart reviewed 2026-09-20, which shows exactly
the two-cluster shape this method is designed for):

1. Scan §1's bins in ascending value order, starting from the 0W bin, using the
   smoothed counts above.
1. A bin is a **candidate peak** if its (smoothed) count is ≥ the preceding
   bin's (or it's the first bin) and > the following bin's — a local maximum,
   ties resolved toward the lower-value bin.
1. From a candidate peak, scan forward through the following bins for as long as
   each is ≤ the previous one (i.e. still declining) — this is the peak's
   **valley region**, ending either where counts start rising again or the
   histogram ends. The candidate is **confirmed** as the peak if the valley
   region's lowest count falls below the peak's count by more than
   `fall_threshold`% (default 20%, configurable — a separate setting from `cap`
   above, despite the coincidentally equal default) at *any* point in that
   region, not only the bin immediately after the peak — a shallow 10% dip
   immediately following the peak, continuing into a 50% dip two bins further
   out, is correctly recognized this way, which a single-next-bin-only
   comparison would have missed. If not confirmed, continue scanning past the
   valley region for the next candidate and repeat.
1. Baseline = the mean of the *raw, unsmoothed* samples in every bin from 0W
   through the confirmed peak bin, inclusive — not the peak bin alone, and not a
   bin center — consistent with baseline representing the whole low-value
   cluster, not just its tallest point.

This deliberately takes the **first** confirmed peak scanning upward from zero,
not the tallest or the globally lowest-value peak found by enumerating all of
them first — the live chart shows why: Phase A has two clusters (an early one,
then a taller one further out), and it's the first, smaller one that represents
the true idle floor.

**No confirmed peak:** if auto mode confirms no peak (e.g. a monotonic or
single-plateau histogram), it falls back to the percentage-cutoff behaviour with
`cap` = 20, i.e. the mean of the lowest 20% of samples by value. A phase with no
samples at all reports its baseline sensor as unavailable.

**Role:** baseline is a per-phase sensor with two uses. (1) Diagnostic:
comparison against that phase's *current* live wattage to surface idle consumers
— a live reading persistently well above baseline, at a time nothing should be
actively running, is a signal something was left on. (2) Holiday mode
(ADR-0009): the whole-house baseline × a buffer factor replaces §3's average as
the expected consumption while the holiday switch is on. In normal mode it is
not an input to any other capability and not a floor/fallback value for §3's
buckets.

### 3 — Average and standard deviation: per phase × weekday × slot

Average and standard deviation are computed per **phase × weekday × 5-minute
slot** (the project's canonical resolution) from the historical (§5-corrected)
readings of the last `window_days` days falling into that specific bucket,
**excluding slots in which holiday mode was active** (ADR-0009 §3):

- Average = mean of the bucket's samples.
- Standard deviation = sample standard deviation of the same set.

**Role of standard deviation:** feeds a per-bucket *confidence* figure, which
ADR-0006 uses to size its margin gap buffer (ADR-0006 §5) — a bucket with high
relative variability contributes a wider buffer than a stable one. The
confidence/margin formula itself lives in ADR-0006, not here, since it's
specific to how the gap calculation consumes it.

**Minimum-history reliability:** resolved by the same smoothing principle
already established in §2, extended across this section's weekday+slot grid —
borrowing from neighboring slots/weekdays when a given bucket's own sample count
is thin, rather than needing a separate minimum-history threshold or fallback
mechanism. The exact extension (which neighboring buckets, what weighting) is
left as an implementation detail, analogous to §2's own flagged-but-unvalidated
weighting choice. Excluding holiday slots thins buckets in the same way and is
covered by the same mechanism.

### 4 — Whole-house combination for downstream consumers

ADR-0006's gap calculation needs a single whole-house per-slot figure, not three
separate per-phase ones. For a given weekday+slot bucket, the three phases' §3
figures are combined as:

- **Combined average** = sum of the three phases' averages (means add linearly).
- **Combined standard deviation** = √(sum of the three phases' variances) — i.e.
  combined as independent variances, not by summing standard deviations directly
  — under the assumption that the three phases' loads are statistically
  independent. This is a simplification: if a specific appliance genuinely
  straddles phases or its load correlates across them, the combined figure
  understates the true combined variance. Accepted for the MVP; revisit if it
  proves wrong in practice.
- Baseline (§2) is **not** part of this combination in normal mode — it stays a
  per-phase figure (§2's "Role"). In holiday mode ADR-0009 §2 sums the phases'
  baselines the same way (means add linearly).

### 5 — Gross-consumption correction using the BMS's own grid sensors

Solakon's grid activity affects the Shelly reading on whichever phase it's wired
to — historical grid-import readings on that phase, during periods the BMS was
actively drawing from or feeding to the grid, don't reflect gross household
load. ADR-0004 §4's per-phase grid sensors make the correction exact rather than
inferred: for a given historical interval and phase,

```
corrected_consumption = max(0, shelly_reading
                                + bms_grid_fed(phase)
                                − bms_grid_consumed(phase))
```

— the BMS feeding *to* the grid depressed what Shelly saw, so it's added back;
the BMS drawing *from* the grid on that phase isn't household load at all, so
it's subtracted out. The result is **clamped to a minimum of zero** —
measurement/timing noise between the asynchronously-polled Shelly and BMS
sensors can otherwise push the corrected figure slightly negative, which isn't
meaningful for a consumption reading (this clamp is a plain floor operation,
unrelated to the `cap` config parameter in §2 despite the similar wording). This
correction is applied before a reading enters §1's histogram or §3's per-slot
buckets, using the phase mapping ADR-0004 §4 establishes. Superseded from an
earlier, less precise version of this idea that proposed inferring the
correction from the whole-battery discharge-energy counter (§1's data source)
instead of a direct per-phase measurement.

The Shelly reading is signed, so a phase exporting more than it imports (the BMS
feeding the grid) arrives as a negative value; the formula above relies on that.
Any further generation source on a phase — a second inverter, say — is not part
of the MVP: it would have to be configured **per phase** (its own injection
sensor for that phase) and added back in the same way.

### 6 — Interface exposed to ADR-0006

The combined average (§4) for a given weekday+slot is the "expected consumption
per slot" value ADR-0006 §1 treats as an opaque input. The combined standard
deviation is additionally exposed and is now used — ADR-0006 §5 derives a
confidence figure from it, which in turn sizes a margin gap buffer. See ADR-0006
for the actual formula.

In holiday mode (ADR-0009 §2) the same interface returns the whole-house
baseline × `HOLIDAY_BASELINE_FACTOR` as the average and a standard deviation of
0\.

### 7 — History access and configuration

- **Entities:** each phase's Shelly power sensor is chosen by the user in the
  config flow (entity picker per phase, L1/L2/L3 — the same pattern as ADR-0004
  §2). The BMS's per-phase grid sensors come from the BMS adapter's mapped
  entity IDs (ADR-0004 §1/§4). A phase with no Shelly sensor configured
  contributes nothing to the whole-house sums.
- **Source:** the recorder's 5-minute statistics (`mean`), read through the
  injected `fetch_fn` (ADR-0001 §6), so `consumption.py` stays in the
  zero-mocking tier and never imports the recorder API. All sensors involved
  must carry a `state_class` so statistics exist, and recorder retention must
  cover `window_days`.
- **Holiday flags:** which historical slots were holiday slots comes from the
  holiday switch's recorder state history through the injected `fetch_flag_fn`
  (ADR-0001 §6, ADR-0009 §3); `consumption.py` stays pure.
- **Cadence and persistence:** recomputed at startup and once per day (ADR-0001
  §6); nothing is persisted — the profile is rebuilt from the recorder.

---

## Consequences

- **Pro:** Keeping phases separate during construction avoids one phase's load
  pattern contaminating another's statistics.
- **Pro:** Baseline's phase-only granularity converges quickly, without needing
  to wait for a full weekday+slot history to mature.
- **Pro:** §5's correction using ADR-0004 §4's direct per-phase grid sensors is
  exact, not inferred — no need to guess at how much of a historical reading was
  attributable to the BMS.
- **Pro:** Standard deviation isn't just exposed for possible future use —
  ADR-0006 already consumes it via a confidence-derived margin buffer.
- **Pro:** The baseline gains a second use (ADR-0009) without any new data
  source.
- **Pro:** Every input is rebuildable from the recorder, so no cache or
  persistence layer is needed.
- **Con:** Depends on recorder retention of at least `window_days` and on
  statistics existing for the Shelly and BMS sensors.
- **Con:** All three of this ADR's original open questions are now resolved
  (baseline's role, confidence/margin, minimum-history via smoothing), but two
  of the resolutions (§3's smoothing extension, ADR-0006's confidence/margin
  formula) are explicitly proposed defaults rather than validated the way
  §1/§2's core algorithm was against live data — worth confirming once
  implemented.
