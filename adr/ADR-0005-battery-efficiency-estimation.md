# ADR-0005 – Battery Round-Trip Efficiency Estimation

**Date:** 2026-09-20 **Status:** Draft **Revised:** 2026-10-12 (glossary
alignment, standby input) **Related ADRs:** ADR-0004 (SoC, capacity, energy
counters, mapped entity IDs), ADR-0001 §5/§6 (`window_days`, `fetch_fn`),
ADR-0012 (standby power estimate) **Related capability:** CAP-05
(`tasks/adr-capability.md`)

---

## Context

The slot-wise trajectory and gap calculation (ADR-0006) needs a **round-trip
efficiency** figure to convert between grid energy drawn and usable battery
energy gained. The BMS does not expose this figure directly, but it exposes
cumulative charge and discharge energy counters (ADR-0004 §1) and the state of
charge.

A plain ratio of the counters' deltas over an arbitrary interval is biased by
whatever energy was added to or removed from storage in between. Comparing two
points at which the battery sits at (approximately) the same low SoC cancels
that term — this is what the design below exploits, by anchoring the interval on
two SoC *valleys*.

---

## Decision

### 1 — Principle

Round-trip efficiency is derived from the installation's own history: energy
delivered out of the battery divided by energy put in, between an early **SoC
valley** and a recent SoC valley within the **history window** (`window_days`,
ADR-0001 §5), with a small correction for any remaining SoC difference (§4). It
covers the battery path only; PV-side conversion losses are outside Chargy's
scope (the sibling project Effy handles them).

### 2 — Inputs

SoC, capacity, the charge-energy counter and the discharge-energy counter are
read from the recorder through the injected `fetch_fn` (ADR-0001 §6), using the
entity IDs the BMS adapter exposes (ADR-0004 §1). The counters are
`TOTAL_INCREASING`: they are read as statistics `sum` at the valley timestamps
(not as raw states), so meter resets inside the interval do not matter and need
no special handling. SoC and capacity are read as 5-minute series (`mean`).
Energy values are normalised to Wh (ADR-0001 §9). Which sensors the user picked
for the two counters decides what the efficiency includes (ADR-0004 §1): raw BMS
counters include the conversion losses, the loss-corrected sensors of Effy
exclude them. The **standby power** used in §4 is the effective value the
coordinator supplies: the picked or fixed value, or the ADR-0012 estimate when
set to auto.

### 3 — Valley search

Two searches run, each over a **valley search window** of 24 hours:

- **Start search** — first valley search window `[t0, t0 + 24 h]`, where `t0` is
  the later of `now − window_days` (the start of the history window) and the
  earliest available SoC history.
- **End search** — first valley search window `[now − 24 h, now]`.

In each valley search window the minimum SoC sample is taken (ties: the sample
nearest the valley search window's centre). If that minimum lies within the
**middle 18 hours** of the valley search window — i.e. not in its first or last
3 hours — it is the SoC valley. Otherwise the valley search window is shifted by
6 hours and the step repeats: the start search shifts **forward**, the end
search shifts **backward**. A minimum at the edge of a valley search window is
presumed to be the slope of a valley that lies outside it, not the valley
itself.

A search ends without a valley if its valley search window would overlap the
other search's valley search window or leave the available data.

### 4 — Efficiency from the two valleys

With `t1` the start valley and `t2` the end valley:

```
out    = discharge_sum(t2) − discharge_sum(t1)
in     = charge_sum(t2)    − charge_sum(t1)
stored = SoC(t2)/100 × capacity(t2) − SoC(t1)/100 × capacity(t1)      (usually ≈ 0)

standby = standby_power × (t2 − t1)                  (Wh = W × h; effective standby power, ADR-0004 §1 / ADR-0012)

efficiency = out / (in − stored − standby)
```

`capacity(t)` is the battery's capacity at the valley's timestamp, read from its
history (ADR-0004 §1, mapped entity ID). Start and end capacity are applied
separately, so a battery extension between the valleys is reflected; the small
residual — the added module's own stored energy — is accepted, since the valleys
sit at low SoC and the effect on the efficiency is small. If no capacity history
is available, the current capacity is used for both valleys. The discharge
counter does not count the energy standby power consumes, yet that energy leaves
the battery, so it is subtracted here; ADR-0006's simulation models it
separately and it must not be absorbed into the efficiency as well. If `out ≤ 0`
or `in − stored − standby ≤ 0` the result is treated as unusable (§5). The
result is clamped to `(0, 1]`.

### 5 — Fallback

The configurable static default — **default 95 %** — is used whenever no usable
measurement exists: either search found no valley, the two valleys are **less
than 24 hours apart**, or §4's guards reject the numbers. This is a start-up
condition: once the recorder holds enough battery cycling history for two
valleys at least a day apart, the fallback stops applying.

The efficiency sensor exposes the value and an attribute stating whether it is
`measured` or the `default`. It is rebuilt at startup and once daily (ADR-0001
§6).

---

## Consequences

- **Pro:** The estimate reflects this specific installation's actual behavior
  rather than a nameplate figure, and anchoring on equal-SoC valleys keeps it
  unbiased without modelling the battery's stored energy.
- **Pro:** Using the counters' statistics `sum` makes counter resets a
  non-issue; no outlier or reset handling is needed in Chargy.
- **Pro:** Everything is rebuilt from the recorder; nothing is persisted.
- **Con:** Early operation relies on the default until two valleys at least 24
  hours apart exist.
- **Con:** The valley heuristic (24-hour valley search windows, middle-18-hours
  rule, 6-hour shifts) is a pragmatic default, not validated against long-term
  data; a battery that rarely cycles deeply yields few usable valleys and stays
  on the default longer.
- **Con:** Requires recorder statistics for SoC, capacity and both counters
  (`state_class` set), and recorder retention of at least `window_days`.
