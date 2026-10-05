# ADR-0009 – Holiday Mode (Baseline-Only Consumption Expectation)

**Date:** 2026-10-05 **Status:** Draft **Related ADRs:** ADR-0008 (baseline §2,
average/deviation §3, interface §6), ADR-0006 (consumes the expected
consumption), ADR-0001 §6 (recorder access) **Related capability:** CAP-08
(`tasks/adr-capability.md`)

---

## Context

The learned weekday+slot consumption profile (ADR-0008 §3) describes a lived-in
household. While nobody is home it overstates what the house will draw, and the
energy gap (ADR-0006) would plan to charge for consumption that never happens.
The per-phase baseline (ADR-0008 §2) is exactly the idle floor such a household
sits at, but until now it was a purely diagnostic figure.

---

## Decision

### 1 — The holiday switch

`ChargyHolidaySwitch` (`switch.py`) is a user-facing Home Assistant switch, one
per config entry, off by default, restoring its last state across restarts. It
is the only input: there is no calendar, date range or automatic detection — an
automation that wants one can flip the switch.

### 2 — Expected consumption in holiday mode

While the switch is on, `consumption.py` answers ADR-0008 §6's interface with:

- **average:** the whole-house baseline × `HOLIDAY_BASELINE_FACTOR`, for every
  slot regardless of weekday or time. The whole-house baseline is the sum of the
  configured phases' baselines (means add linearly, as in ADR-0008 §4).
- **standard deviation:** `0`.

`HOLIDAY_BASELINE_FACTOR = 2.0` is a constant in `const.py` — the "baseline
buffer" — not a config-flow option in the MVP.

Because the standard deviation is 0, ADR-0006 §5's margin is 0 for every slot:
the factor is the buffer, and `gap.py` needs no special case. If any configured
phase has no baseline (no samples), holiday mode cannot be evaluated; the normal
profile is used for that recompute and a warning is logged.

### 3 — Holiday periods stay out of the learned statistics

Slots in which holiday mode was active are **excluded from the average and
standard-deviation buckets** of ADR-0008 §3, for all phases, so a bucket
reflects normal days only. The baseline histogram (ADR-0008 §1/§2) is
deliberately **not** filtered: it measures idle load, which holiday periods
sample particularly cleanly.

- A slot counts as holiday if the switch was `on` at any moment within it.
- The flags come from the switch's recorder state history (a switch has no
  statistics) through a second injected callable, `fetch_flag_fn` (ADR-0001 §6).
  `None` — no history for the slot, e.g. before the switch existed or in a
  recorder gap — is treated as *not* holiday: a recorder gap normally removes
  the consumption samples of that slot as well.
- Excluding slots thins the affected buckets; ADR-0008 §3's smoothing covers
  this.

### 4 — Live behavior

The coordinator reads the switch state on every recompute. Flipping the switch
triggers an immediate recompute of the gap and the charge plan rather than
waiting for the next 5-minute tick (ADR-0001 §6).

---

## Consequences

- **Pro:** Reuses the baseline already computed for the diagnostic sensor; no
  new data source.
- **Pro:** The gap shrinks to the idle need while away, with no change to the
  gap formula.
- **Con:** A fixed factor of 2 is a guess; a house with heavy standby loads or a
  lived-in pattern even when empty may want another value (a later config
  option).
- **Con:** The switch is manual; forgetting to turn it off after returning makes
  Chargy under-plan until noticed.
- **Con:** Excluded slots thin the buckets, so long holidays within the history
  window weaken the normal-mode profile for a while.
