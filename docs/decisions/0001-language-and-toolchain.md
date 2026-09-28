# ADR-0001: Implementation language, toolchain, test runner, CI

**Status:** proposed
**Issue:** #2
**Date:** 2026-09-28

## Context

The PRD does not say what Neeti is written in. Nothing in Phase 1 of
[docs/ROADMAP.md](../ROADMAP.md) can start until this is fixed, and
[AGENTS.md § Working an issue](../../AGENTS.md#working-an-issue) step 6 refers
to "the test suite named in `docs/decisions/0001-*.md`", so this ADR must name
a concrete test command.

Constraints from the PRD and the roadmap:

- Facts and rules are "in a form a machine can reason over and a person can
  read". The on-disk form is text; the host language must read it plainly.
- Every answer carries a derivation a person can replay by hand. The reasoner
  test literally replays it, so tests are table-driven over facts, rules, and
  expected derivations, and failures must print readable diffs.
- "Unknown" is a first-class answer. The language must make "no answer" a
  distinct value, not an exception or a null that gets lost.
- The language edge is optional and a stand-in in Phase 1. Calling a real model
  later must be easy, and the model must stay at the edge.
- ADR-0002 (#3) chooses the formalism and solver from at least: Datalog
  (Soufflé or in-process), answer set programming (clingo), Prolog, a
  description-logic reasoner, or a purpose-built minimal engine. This ADR must
  not pre-empt that choice, so the host language must be able to bind to
  **every** candidate, not just one.
- The work is done by agents in short loops. Setup, test, and CI must be fast
  and boring; a slow or fragile toolchain is paid on every issue.
- AGENTS.md forbids runtime dependencies an issue did not name. Development
  tooling is named here so a follow-up issue can add it; the runtime dependency
  set stays empty until ADR-0002 names a solver.

## Options

### A. Python (3.12+), `uv`, `pytest`, `ruff`, `mypy`, GitHub Actions

How it works: a `src/` layout package `neeti` with one subpackage per module.
[uv](https://docs.astral.sh/uv/) manages the interpreter, virtualenv, and a
lockfile; [pytest](https://docs.pytest.org/) runs the suite;
[ruff](https://docs.astral.sh/ruff/) formats and lints;
[mypy](https://mypy.readthedocs.io/) `--strict` checks type hints; one GitHub
Actions workflow ([setup-uv](https://github.com/astral-sh/setup-uv)) runs all
four on pull requests.

What it buys: every ADR-0002 candidate has a maintained Python route —
[clingo](https://potassco.org/clingo/python-api/current/clingo/) ships official
wheels with a first-class API; SWI-Prolog exposes
[janus-swi](https://www.swi-prolog.org/pldoc/man?section=janus); Soufflé runs
as a subprocess over text files; description logic has
[owlready2](https://owlready2.readthedocs.io/); a purpose-built engine is plain
Python. Every model provider ships a Python SDK, so the edge is cheap to add
later. Agents produce correct Python with the least churn. `Optional[T]` plus a
dedicated `Unknown` type, checked by `mypy`, makes "no answer" a visible value.

What it costs: dynamic typing, so the derivation data structures need type
hints and a checker in CI or they drift. Slower than a compiled language;
irrelevant for a hand-checkable slice, and the heavy lifting moves into the
solver ADR-0002 picks anyway. `owlready2`'s bundled reasoners need a JVM, a
cost that lands on ADR-0002 only if it picks that option.

What it rules out: nothing in ADR-0002's candidate list.

### B. Rust, `cargo test`, `rustfmt`, `clippy`

Strong types make a derivation a value the compiler checks, and `Option` makes
"unknown" unavoidable. Zero-decision toolchain: `cargo` is build, test, format,
and lint. Costs: bindings to the ADR-0002 candidates are thin or absent —
`clingo-rs` exists but lags upstream, there is no maintained Prolog or OWL
route, and Soufflé is subprocess-only — so choosing Rust leans ADR-0002 toward
a purpose-built engine or Datalog crates such as `ascent`, which this ADR is not
allowed to do. Compile times are paid on every CI run, and agents spend
iterations on the borrow checker for code whose cost is not performance.

### C. TypeScript on Node, `vitest`, `biome`

Fast iteration, good LLM SDKs, agents are fluent. Costs: the solver story is
weak — clingo only via a WASM build, Prolog only via pure-JS ports, no
description-logic reasoner — so this too would pre-empt ADR-0002. Two package
managers, a bundler question, and `tsconfig` drift are toolchain churn the
project does not need.

### D. Go, `go test`, `gofmt`, `staticcheck`

The most boring toolchain: one binary, built-in test runner and formatter, fast
CI. Costs: no maintained clingo, Prolog, or OWL bindings, so Go forces the
purpose-built engine before ADR-0002 is written. Zero values and `nil` make
"unknown" easy to lose without discipline.

### E. Write Neeti in a logic language (SWI-Prolog with `plunit`)

The formalism is the implementation: rules are code, derivations fall out of
the proof tree. Costs: this *is* the ADR-0002 decision made by the back door.
It also makes the store's contradiction check, the constraints module, and the
language edge harder to keep separate, and the test, lint, and CI story is
thinner than the options above.

## Recommendation

**Option A: Python 3.12+, managed by `uv`, tested with `pytest`, formatted and
linted with `ruff`, type-checked with `mypy --strict`, run in one GitHub
Actions workflow.**

It is the only option that keeps every ADR-0002 candidate open with a
maintained binding, and the only one where the language edge is a later
addition rather than a later fight. Rust's type safety is real but the
derivation structures are small; type hints plus `mypy --strict` recover most
of it at a fraction of the iteration cost. Go and TypeScript are boring in the
right way but each quietly decides the solver. Option E decides it outright.

Concretely, the follow-up scaffolding issue adds:

- **Layout.** `pyproject.toml` and `uv.lock` at the root; `src/neeti/` with
  subpackages `store/`, `reasoner/`, `constraints/`, `edge/` and a
  `__main__.py` for the Phase 1 "one command" exit; `tests/` mirroring the four
  modules (`tests/store/`, `tests/reasoner/`, `tests/constraints/`,
  `tests/edge/`); `knowledge/` reserved for the hand-written fact files, whose
  format ADR-0002 decides.
- **Runtime dependencies.** None. ADR-0002 names the solver; no other runtime
  dependency is added without an issue that names it.
- **Development dependencies** (a `dev` dependency group): `pytest`, `ruff`,
  `mypy`. Pinned in `uv.lock`.
- **The test command.** `uv run pytest`. This is the suite AGENTS.md § Working
  an issue step 6 refers to. It must pass on a clean checkout after
  `uv sync --locked`, with no network access beyond the initial sync.
- **Format and lint.** `uv run ruff format --check .` and `uv run ruff check .`.
  Type check: `uv run mypy`. Configuration lives in `pyproject.toml`.
- **CI.** `.github/workflows/ci.yml`, one job named `ci`, on `pull_request`
  and on `push` to `main`, on `ubuntu-latest`, Python 3.12 only. Steps:
  `actions/checkout`, `astral-sh/setup-uv` (with `uv.lock` caching),
  `uv sync --locked`, then the format check, lint, type check, and
  `uv run pytest`, in that order. The `ci` job is the one to mark required when
  branch protection is turned on.

## Consequences

- Phase 1 is unblocked once the scaffolding lands. The next issue is a `chore`
  that adds `pyproject.toml`, `uv.lock`, the empty packages, one smoke test,
  and `ci.yml`, and nothing else. #5–#9 build on it.
- ADR-0002 can pick any of its candidates; its "dependency it adds" line names
  the Python package or subprocess binary for the chosen solver.
- Every derivation-bearing type is annotated, and `mypy --strict` is a merge
  gate, so "unknown" cannot silently become `None` and a derivation cannot be
  a bare `dict`.
- Harder: a solver written in another language is reached over bindings or a
  subprocess. Acceptable for a hand-checkable slice; a later ADR can supersede
  this one if performance ever matters.
- Tooling versions are pinned in `uv.lock`; bumping them is a `chore` PR.
