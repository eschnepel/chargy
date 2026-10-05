# ADR-0005 – Battery Round-Trip Efficiency Estimation

**Date:** 2026-09-20 **Status:** Draft **Related ADRs:** ADR-0004 (energy
counters as input) **Related capability:** CAP-05 (`adr/adr-capability.md`)

---

## Context

The energy gap calculation (ADR-0006) needs a round-trip charge efficiency
figure to convert between grid energy drawn and usable battery energy gained.
The BMS does not expose this figure directly, but does expose cumulative charge
and discharge energy counters (ADR-0004 §1).

---

## Decision

1. Efficiency is **derived from the BMS's charge and discharge energy counters**
   — actual energy delivered out, compared with actual energy put in, observed
   over the installation's own operating history — rather than a fixed nameplate
   assumption.
1. Until enough counter history has accumulated to produce a stable estimate, a
   **configurable static default** value is used in its place.
1. Left open for the implementing worker, to be confirmed at review rather than
   blocking this ADR: the precise definition of "enough history" (e.g. a minimum
   number of observed full charge/discharge cycles) and how counter resets or
   obvious outliers are handled.

---

## Consequences

- **Pro:** The estimate reflects this specific installation's actual behavior
  rather than a generic nameplate figure that may not hold in practice.
- **Con:** Early operation, before sufficient counter history exists, relies on
  a configured default that may not match the real installation until enough
  data accumulates.
