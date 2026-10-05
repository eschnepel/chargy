# Chargy – Price-Optimized Grid-Charge Planning

**Status:** Brainstorming / concept phase – no working version yet. All ADRs are
currently `Draft`, pending Gate 1 approval; see [`adr/INDEX.md`](adr/INDEX.md)
for the authoritative, up-to-date status of every design decision.

Chargy is a Home Assistant integration that plans *when* to charge a home
battery from the grid, so it draws grid power during the cheapest/least
constrained price windows it needs — instead of whenever it happens to be low,
or on a fixed schedule you have to maintain by hand.

## Why this exists

A battery with a dynamic electricity tariff should only pull from the grid
during genuinely cheap windows, and only pull as much as it actually needs to
reach a sensible target state of charge — no more, no less. Doing that well
requires combining several things that usually live in separate places: the
price curve itself, how much energy the battery actually needs (which depends on
its current state, expected PV production, and how much the house is expected to
consume before prices are cheap again), and the battery's own hardware limits.
Chargy's job is to bring all of that together into one concrete, continuously
recomputed plan.

## How it works, in plain terms

- **A price provider** normalizes a dynamic tariff — currently Tibber, or a
  simple configurable day/night tariff — into a 5-minute price curve. The
  architecture is pluggable, so other tariffs can be added later without
  touching anything downstream.
- **A tier strategy** classifies that price curve into at least three tiers
  (cheapest … most expensive), using either fixed €/kWh cutoffs or a rolling
  percentile window that adapts to recent prices.
- **A BMS read adapter** polls the battery management system (currently Solakon)
  for live, read-only state: state of charge, capacity, charge-power limits,
  energy counters, and per-phase grid flow. Chargy never touches the BMS's own
  built-in grid-charge logic or issues control commands — this is observation
  only.
- **A consumption profile** is learned from historical per-phase grid readings
  (e.g. a Shelly 3EM) read from Home Assistant's recorder, per phase, weekday,
  and time-of-day slot, so Chargy has a realistic estimate of what the house
  will draw before the next cheap window arrives.
- **A holiday mode**, switched on by a Home Assistant switch, replaces the
  learned profile with the household's idle baseline (times a safety factor) and
  keeps holiday periods out of the learned averages.
- **A round-trip efficiency estimate** is derived over time from the BMS's own
  charge/discharge energy counters at two low-SoC points, starting from a
  configurable default (95 %).
- **An energy-gap calculation** combines current SoC, capacity, efficiency,
  expected consumption, and an existing PV forecast (from whatever forecast
  integration you already use — Chargy does not forecast PV itself) into how
  much grid energy is actually needed to reach a configurable, soft target SoC.
- **Grid-charge window planning** selects the cheapest allowed 5-minute slots
  that sum to that energy gap, respecting the BMS's own cached max charge power.

Chargy only plans and exposes the result — it does not yet have a mechanism to
instruct the BMS to actually charge; see `tasks/adr-capability.md`'s "Explicitly
Deferred" section for what is intentionally out of scope for now.

## Requirements

- A Home Assistant instance with the Tibber integration configured (for the
  Tibber price provider) or a willingness to use the built-in fixed day/night
  tariff instead.
- A Solakon battery management system, integrated into Home Assistant via its
  community integration, with its own built-in grid-charge logic disabled (so it
  doesn't fight Chargy's plan).
- A 3-phase grid meter (e.g. Shelly 3EM) whose per-phase power sensors are in
  Home Assistant, plus a recorder that keeps at least as many days of history as
  Chargy's history window (default 28) and statistics for those sensors and the
  battery's.
- An existing PV-forecast integration (e.g. Forecast.Solar, Solcast, or a
  sibling project such as [Shady](https://github.com/eschnepel/shady)) feeding
  Chargy's energy-gap calculation.

## Installation (HACS)

1. In HACS, add this repository as a custom repository (category: Integration).
1. Install "Chargy" and restart Home Assistant.
1. Go to **Settings → Devices & Services → Add Integration**, search for
   "Chargy", and follow the setup flow.

## Configuration

Setup is planned to be entirely through the Home Assistant UI — no YAML. The
config flow is expected to grow incrementally as each capability lands: price
provider choice, tier strategy and its parameters, SoC bounds and the
soft-target margin, Solakon entity mapping, the Shelly per-phase sensors, the
history window, and the efficiency default. See `tasks/adr-capability.md` for
exactly which fields belong to which capability.

## Entities (planned)

Nothing is implemented yet; the entities below are what each capability's ADR
describes as demonstrable once built:

- A 5-minute normalized price series, exposed as a sensor/attribute.
- A tier-plan sensor showing the current and upcoming price-tier windows.
- SoC, max-SoC, max capacity, and per-phase grid-consumed/grid-fed sensors,
  sourced live from the BMS.
- Per-phase consumption baseline sensors.
- A holiday-mode switch.
- An efficiency sensor, starting at its configured default.
- A "grid energy needed" sensor.
- A "planned charge windows" sensor listing the selected slots and total planned
  energy.

## For contributors

Chargy's design decisions are recorded as Architecture Decision Records, not in
this README:

- [`adr/INDEX.md`](adr/INDEX.md) — the full ADR list, status, and how they
  relate to one another.
- [`adr/ADR-0000-coding-standards.md`](adr/ADR-0000-coding-standards.md) —
  project-independent coding standards and module-boundary conventions (shared
  with sibling projects).
- [`adr/ADR-0001-project-conventions.md`](adr/ADR-0001-project-conventions.md) —
  Chargy's own identifiers, module layout, and version targets.
- [`tasks/adr-capability.md`](tasks/adr-capability.md) — the draft capability
  list every numbered ADR derives from.
