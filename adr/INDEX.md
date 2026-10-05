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
| [0006](ADR-0006-energy-gap-calculation.md) | Draft | Energy Gap Calculation *(CAP-06)* |
| [0007](ADR-0007-grid-charge-window-planning.md) | Draft | Grid-Charge Window Planning *(CAP-07)* |
| [0008](ADR-0008-consumption-profile-calculation.md) | Draft | Consumption Profile Calculation (Baseline, Average, Deviation) *(CAP-04)* |

See [`adr-capability.md`](adr-capability.md) for the full capability list these
ADRs derive from, pending Gate 1 approval.
