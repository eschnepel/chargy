# ADR Index

This is the authoritative list of all ADRs in this project, their current
status, and how they relate to one another. See
[`ADR-0000-coding-standards.md`](ADR-0000-coding-standards.md) §7 for what
belongs in an ADR versus a code comment.

**This file must be updated whenever an ADR undergoes a structural change** — a
new ADR is added, one is split, superseded, or its status otherwise changes —
see ADR-0000 §7 for the mandatory-update rule this enforces.

**All ADRs below are `Draft`** — none have been reviewed/confirmed by a human
yet. Per this project's Phase 0 convention, a new ADR starts in draft state
until review confirms it; none of these should be treated as settled
architecture until their status changes.

| ADR | Status | Title |
| -- | -- | -- |
| [0000](ADR-0000-coding-standards.md) | Draft | Code Quality Standards, Programming Style & Core Concepts (project-independent) |
| [0001](ADR-0001-project-conventions.md) | Draft | Chargy Project Conventions: Identifiers, Module Layout, Version Targets |
| [0002](ADR-0002-price-provider-architecture.md) | Draft | Price Provider Architecture *(CAP-01)* |
| [0003](ADR-0003-tier-strategy-architecture.md) | Draft | Tier Strategy Architecture *(CAP-02)* |
| [0004](ADR-0004-bms-read-adapter.md) | Draft | BMS Read Adapter Architecture *(CAP-03)* |
| [0005](ADR-0005-battery-efficiency-estimation.md) | Draft | Battery Round-Trip Efficiency Estimation *(CAP-05)* |
| [0006](ADR-0006-energy-gap-calculation.md) | Draft | Slot-wise Energy Trajectory and Gap *(CAP-06)* |
| [0007](ADR-0007-grid-charge-planning.md) | Draft | Grid-Charge Planning *(CAP-07)* |
| [0008](ADR-0008-consumption-profile-calculation.md) | Draft | Consumption Profile Calculation (Baseline, Average, Deviation) *(CAP-04)* |
| [0009](ADR-0009-holiday-mode.md) | Draft | Holiday Mode: Baseline-Only Consumption Expectation *(CAP-08)* |
| [0010](ADR-0010-battery-discharge-planning.md) | Draft | Battery-Discharge Planning *(CAP-09)* |
| [0011](ADR-0011-planner-interface-and-order.md) | Draft | Planner Interface and Execution Order *(CAP-07, CAP-09)* |
| [0012](ADR-0012-standby-power-estimation.md) | Draft | Standby Power Estimation *(CAP-10, pending Gate 1)* |

Domain vocabulary is defined in [`GLOSSARY.md`](../GLOSSARY.md) (ADR-0001 §8).
ADR-0007 and ADR-0010 were renamed on 2026-10-12 (the word "window" now only
appears qualified, see the glossary); their numbers are unchanged.

See [`tasks/adr-capability.md`](../tasks/adr-capability.md) for the full
capability list these ADRs derive from, pending Gate 1 approval.
