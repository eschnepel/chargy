# ADR-0009 – Holiday Mode (Baseline-Only Consumption Expectation)

**Date:** 2026-10-05 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment) **Related ADRs:** ADR-0008 (baseline §2, average/deviation §3,
interface §6), ADR-0006 (consumes the expected consumption), ADR-0001 §6
(recorder access) **Related capability:** CAP-08 (`tasks/adr-capability.md`)

---

## Context

The learned weekday+slot consumption profile (ADR-0008 §3) describes a lived-in
household. While nobody is home it overstates what the house will draw, and the
gap (ADR-0006) would plan to charge for consumption that never happens. The
per-phase baseline (ADR-0008 §2) is exactly the idle floor such a household sits
at, but until now it was a purely diagnostic figure.

---

## Decision

### 1 — The holiday switch

The **holiday switch** (`ChargyHolidaySwitch`, `switch.py`) is a user-facing
Home Assistant switch, one per config entry, off by default, restoring its last
state across restarts. It is the only input: there is no calendar, date range or
automatic detection — an automation that wants one can flip the switch.
**Holiday mode** is the state in which the switch is on.

### 2 — Expected consumption in holiday mode

While the switch is on, `consumption.py` answers ADR-0008 §6's interface with:

- **average:** the whole-house baseline × the **holiday baseline factor**
  (`HOLIDAY_BASELINE_FACTOR`), for every slot regardless of weekday or time. The
  whole-house baseline is the sum of the configured phases' baselines (means add
  linearly, as in ADR-0008 §4).
- **standard deviation:** `0`.

`HOLIDAY_BASELINE_FACTOR = 2.0` is a constant in `const.py` — not a config-flow
option in the MVP.

Because the standard deviation is 0, ADR-0006 §5's consumption band collapses
(high = mean = low): the factor is the buffer, and `planner/simulation.py` needs
no special case. If any configured phase has no baseline (no samples), holiday
mode cannot be evaluated; the normal profile is used for that replan and a
warning is logged.

### 3 — Statistics follow the switch (weighting)

Every statistic of ADR-0008 §1–§3 — the baseline histogram and the
average/standard-deviation buckets — is **weighted by how well each slot's
holiday flag agrees with the current switch state**: weight `1.0` if the slot's
flag equals the switch state, `HOLIDAY_OTHER_FLAG_WEIGHT = 0.2` if it differs.
Slots with the other flag are down-weighted, not discarded, so there is never a
shortage of data to fall back from, and after a switch change the statistics
adapt within a day or two: with the 7-day baseline window, the new state's own
samples outweigh the five older days after about one day (5 × 0.2 = 1) and
dominate after two.

- The weights multiply with ADR-0008's kernel weights where both apply.
- A slot carries the **holiday flag** if the switch was `on` at any moment
  within it.
- The flags come from the switch's recorder state history (a switch has no
  statistics) through a second injected callable, `fetch_flag_fn` (ADR-0001 §6).
  `None` — no history for the slot, e.g. before the switch existed or in a
  recorder gap — is treated as *not* holiday: a recorder gap normally removes
  the consumption samples of that slot as well.
- `HOLIDAY_OTHER_FLAG_WEIGHT` is a constant in `const.py`, not a config option;
  a proposed default.

### 4 — Live behavior

The coordinator reads the switch state on every replan. Flipping the switch
changes the slot weights (§3), so the coordinator re-derives them from the
history it already holds (no refetch) and then immediately replans (gap and
plan) rather than waiting for the next 5-minute tick (ADR-0001 §6).

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
- **Con:** The weighting never fully discards the other state's history: right
  after a switch change the statistics are still a blend, and a very different
  holiday baseline takes a day or two to dominate.
