# ADR-0000 – Code Quality Standards, Programming Style & Core Concepts

**Date:** 2026-09-20 **Status:** Draft **Split note (2026-09-20):** the module
dependency diagram and every project-specific identifier (package name,
entity-class prefix, minimum Python/HA version) that originally lived in this
document have been moved out into a dedicated project-conventions ADR, so this
document stays genuinely project-independent. See the project's `adr/INDEX.md`
for that ADR's number and title.

---

## Context

This ADR documents overarching engineering conventions for a Home Assistant
custom-component project, independent of which project adopts it. It exists so
that contributors and future maintainers can understand *why* the code looks the
way it does without having to infer it from individual diffs. Domain-specific
decisions belong in the project's other numbered ADRs; this one covers
everything that applies uniformly across all files, regardless of what the
integration actually does.

This document is itself the product of exactly that separation: it descends from
two sibling projects, [Effy](https://github.com/eschnepel/effy) and
[Shady](https://github.com/eschnepel/shady), whose own ADR-000 files each mixed
generic conventions with that project's specific module names and identifiers.
Shady's version is the more mature of the two, having absorbed several process
lessons (the testing-tier definition in §6, the `INDEX.md` sync rule in §7, the
diagramming rule in §9) worth keeping. Rather than repeat their pattern of
hardcoding one project's names into a "generic" document, this version keeps
every project-specific fact (package name, entity-class prefix, concrete module
diagram, version targets) out entirely — a project adopting this ADR records
those in its own companion conventions ADR instead of editing this one.

Throughout this document, `<project>` denotes an integration's package name
(lowercase, e.g. what would sit under `custom_components/<project>/`) and
`<Project>` its PascalCase entity-class prefix (e.g. `<Project>Sensor`,
`<Project>Coordinator`). A project adopting this ADR fixes both, along with a
concrete module diagram and its minimum supported Python/HA version, in its own
project-conventions ADR rather than by editing this file — see §3 and §4 below
for exactly what that companion ADR needs to supply.

---

## Decision

### 1 — Tooling: ruff, mypy strict, pytest

Three tools gate every change, run via `.github/workflows/ci.yml`:

| Tool | Purpose | Invocation |
| -- | -- | -- |
| `ruff format` | Code formatting (replaces black) | `ruff format custom_components/` |
| `ruff check` | Linting (replaces flake8/isort/pyupgrade) | `ruff check custom_components/` |
| `mypy --strict` | Static type checking | `mypy custom_components/<project> tests --config-file mypy.ini` |
| `pytest` | Unit tests | `pytest tests/` |

All four must pass with zero errors before a change is considered complete.
`mypy --strict` is non-negotiable: every function signature carries full type
annotations, including return types on methods that return `None`.

### 2 — Handling Home Assistant's untyped surface

Home Assistant ships without inline type stubs for many of its base classes and
decorators (`SensorEntity`, `ConfigFlow`, `@callback`, etc.). Under
`mypy --strict` this produces two categories of unavoidable noise:

- `misc` errors when subclassing a base class typed as `Any`.
- `untyped-decorator` errors when `@callback` wraps a method.

These are suppressed **per-file**, not globally, via `mypy.ini`
(`warn_unused_ignores = False` on the specific modules that subclass HA entities
or use `@callback`), combined with targeted `# type: ignore[<code>]` comments at
the exact line mypy flags. Suppression is never broad (no bare `# type: ignore`
without a code, no `disable_error_code` at the global `[mypy]` level) — the goal
is to silence exactly the HA-stub gap, not to weaken type checking elsewhere.

The comment goes on the exact source line mypy reports, not on a nearby line:
`misc` errors on subclassing are attached to the base-class entry in the class
statement (e.g. `class <Project>Sensor(SensorEntity):  # type: ignore[misc]`),
and `untyped-decorator` errors are attached to the `@callback` line itself, not
the `def` line below it — mypy's reported line number is authoritative and
should never be guessed at.

The modules held to the full, unsuppressed strict standard are exactly the
zero-Home-Assistant-import, pure-Python tier — see §6 for what defines that
tier. A module belongs to this typing tier if and only if it is also in §6's
zero-mocking test tier; the two are the same boundary described from two
different angles.

### 3 — Module boundaries and dependency direction

The general principle: **pure calculation logic sits at the bottom, has zero
Home Assistant imports, and never imports upward; HA entity glue sits at the
top; a coordinator orchestrates between them; dependencies point upward only.**
This is what allows the domain math to be unit-tested in complete isolation from
Home Assistant (§6) and, incidentally, to be reusable outside HA entirely (e.g.
a CLI tool or a test harness) if ever useful.

A module that necessarily talks to `hass.states` by design — because its whole
job is reading another integration's entities — is an explicit, narrow exception
to "zero HA imports." It still never touches the coordinator's orchestration
concerns and never writes state at this layer.

This document intentionally contains no concrete module diagram: the actual
file/package layout is domain-specific and, before any code exists, usually
still unknown (file structure should emerge during implementation rather than
being imposed upfront). A project adopting this ADR records its own current
diagram — updated by amendment as real modules land and diverge from any earlier
sketch — in its companion project-conventions ADR.

### 4 — Type-hinting conventions

- `from __future__ import annotations` at the top of every module — allows
  modern `list[str] | None` syntax without runtime evaluation cost.
- Built-in generics (`list[str]`, `dict[str, float]`) are used directly;
  `typing.List`/`typing.Dict` are never imported.
- `X | None` is used instead of `Optional[X]`.
- Every function and method has a complete signature: parameter types and a
  return type, including `-> None`. This applies to private helpers exactly as
  it does to public HA-facing methods — `mypy --strict` does not distinguish,
  and a single unannotated parameter triggers `no-untyped-def` just as an
  entirely bare signature does.
- `@dataclass` is used for plain data containers instead of dicts or named
  tuples — gives attribute access, auto-generated
  `__init__`/`__repr__`/`__eq__`, and a single place to add validation later if
  needed.
- **Minimum supported Python version:** always set to match Home Assistant's own
  bundled minimum for whichever HA release the project targets
  (`pyproject.toml`'s `requires-python`, `[tool.mypy] python_version`, and
  `hacs.json`'s `homeassistant` key must all agree). Both sibling projects had
  to correct a stale hardcoded version after HA's own minimum moved past their
  original assumption — record the actual chosen value in the
  project-conventions ADR, not here, precisely so this document never goes stale
  the same way.

### 5 — Naming and structure

- Module-level constants are `UPPER_SNAKE_CASE` and live in `const.py`
  (cross-module) or at the top of the module that owns them (single-use).
- Private helpers are prefixed with a single underscore and are not exported.
- HA-facing entities (`<Project>Sensor`, `<Project>Coordinator`,
  `<Project>ConfigFlow`, `<Project>OptionsFlow`, …) are all prefixed with the
  project's fixed entity-class prefix for discoverability when grepping or
  reading stack traces.
- One concept per module: each module or package does exactly one thing. A
  module that starts doing two unrelated things is a signal to split it.

### 6 — Testing philosophy

- Every module in the zero-HA-import, pure-Python tier (as currently enumerated
  in the project-conventions ADR) is unit-tested with **zero mocking** — no
  `unittest.mock`, no fake `hass` object. Because these modules have no Home
  Assistant dependency, tests call the real functions with real dataclass
  instances and assert on real return values. This is only possible *because* of
  the module boundary in §3; it is the practical payoff of that design choice.
  The narrow exception is any module that reads `hass.states` directly by design
  (§3's exception) — those are tested against a real `hass` fixture instead.
- Tests are loaded via direct file-path import
  (`importlib.util.spec_from_file_location`) rather than package import,
  specifically to avoid pulling in `custom_components/<project>/__init__.py`
  (which imports `homeassistant.*`) just to test a dependency-free module. This
  keeps the test environment lightweight (`pytest` only — no
  `pytest-homeassistant-custom-component` needed).
- Because file-path loading yields plain `ModuleType` objects, mypy cannot see
  the real classes on attributes such as a dynamically-loaded module's own data
  classes — it only sees `Any`. Test files still get full static typing for
  these names via a `TYPE_CHECKING`-only static import that mirrors the runtime
  path (e.g.
  `if TYPE_CHECKING: from <project>.<module> import <ClassName> as <ClassName>`).
  This import is never executed (it runs only under static analysis), so it does
  not reintroduce the `homeassistant` dependency the file-path loading was
  designed to avoid; the runtime assignment
  (`<ClassName> = _module.<ClassName>`) is correspondingly guarded with
  `if not TYPE_CHECKING:` so the two bindings never conflict. This is mandatory
  for any test module that binds a dynamically-loaded class to a name used later
  as a type annotation.
- Every test class documents the scenario it covers in a docstring or comment,
  so the intent behind a fixture is clear without having to re-derive it from
  the numbers alone.
- Invariant checks specific to each capability's math (conservation properties,
  bounds clamps, coverage of a time horizon with no gaps or overlaps, etc.) are
  asserted explicitly in tests, not just spot-checked values — each capability's
  own ADR is the source of truth for what its invariants actually are; this
  section only mandates that they be tested as first-class assertions, not
  spot-checks.

### 7 — Documentation: ADRs over inline essays

Design rationale lives in `adr/`, not in large module docstrings or inline
comment blocks. Module docstrings stay short (what the module does, 1–3
sentences); the *why* behind non-obvious decisions is captured once in an ADR
and referenced by number from the code (e.g. `# Clamp to zero (ADR-000X §2)`).
This avoids rationale drifting out of sync with the code, since an ADR is
versioned independently and can be marked `Superseded` if a decision changes,
without having to hunt down every comment that explained it.

**[`adr/INDEX.md`](INDEX.md) is the single, authoritative list of every ADR and
its current status** — not this document, and not the project README (which only
links to it). Updating it is a **mandatory** part of any structural change to
the ADR set, in the same commit as the change itself, not a follow-up: adding a
new ADR, splitting one, marking one `Superseded`, or otherwise changing an ADR's
`Status` header all require a matching edit to `adr/INDEX.md`.

### 8 — Error handling

- User-facing errors (config flow validation, e.g. an unreachable source entity
  or an invalid configuration value) return error keys resolved via
  `translations/*.json`, never raw exception text — keeps the UI translatable
  and avoids leaking internals.
- Background failures (a periodic refresh, an external-data poll) are logged via
  `_LOGGER.exception`/`_LOGGER.warning` and swallowed rather than raised, since
  these run outside a request/response cycle where there is no caller to
  propagate the exception to.
- Pure calculation modules raise no exceptions in their normal operating range;
  they use `min`/`max` clamps instead of validation errors, because their inputs
  are derived from live sensor/forecast data that is expected to occasionally be
  noisy rather than invalid.

### 9 — Diagrams and tables: Markdown/Mermaid-native, not ASCII art

Structural diagrams (module dependency graphs, data-flow pipelines, state
machines) are written as Mermaid code blocks (```` ```mermaid ````), not as
hand-drawn ASCII boxes/arrows inside a plain code fence — Mermaid renders
natively on GitHub and in most Markdown viewers, where ASCII art frequently
misaligns once line-wrapped, font-substituted, or viewed on a narrow screen.
Tabular data is written as a native Markdown table (`| ... | ... |` with a
header separator row), not hand-aligned with manual spacing in a code fence, for
the same reason. Per-node/per-module explanatory prose belongs in an adjacent
bullet list next to the diagram, not crammed inside the diagram's own node
labels — a diagram should show *structure*, prose explains *why*. Plain code
fences remain the right tool for formulas, type/data-shape sketches, and short
illustrative config examples — none of these are themselves diagrams or tables,
so this rule does not apply to them.

---

## Consequences

- **Pro:** A new contributor can run four commands (`ruff format --check`,
  `ruff check`, `mypy --strict`, `pytest`) and know immediately whether their
  change meets the bar.
- **Pro:** The pure-logic/HA-glue split makes a project's core domain logic
  trivially testable and reusable independent of Home Assistant.
- **Pro:** ADRs prevent "tribal knowledge" about *why* a clamp, a config bound,
  or a suppression exists from living only in a pull request that gets buried.
- **Pro:** Keeping this document free of any project's concrete names means it
  can be copy-pasted into a new project unchanged, with only the companion
  project-conventions ADR needing to be written — no risk of a stale module
  diagram or version number living inside the "standards" document itself.
- **Con:** Two documents (this one plus the project-conventions ADR) must be
  read together to get the full picture, instead of one self-contained file — a
  small navigation cost traded for this document never needing an edit just
  because the project's module layout changed.
- **Con:** Strict mypy plus per-file suppression configuration is more upfront
  setup than "just ignore HA imports everywhere" — but it means type errors in
  actual business logic are never silently masked by a blanket ignore.
