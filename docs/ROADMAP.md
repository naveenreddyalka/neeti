# Neeti roadmap

The order of work. Each phase ends with something a person can run and
observe. Issues carry the detail; this file only says what comes before what.

## Phase 0 — Decide

The PRD left seven decisions open. Three block the first code and are filed as
`decision` issues. The rest are deferred to the phase where they first matter.

| ADR | Decision | Blocks |
|-----|----------|--------|
| 0001 | Implementation language, toolchain, test runner, CI | everything in Phase 1 |
| 0002 | Knowledge formalism and solver | store, reasoner, constraints |
| 0003 | First knowledge slice and the initial inviolable rule set | constraints, all example data |

Deferred: where facts come from and how a source is vetted (Phase 2); how a
contradiction between sources is resolved (Phase 2); which language model sits
at the edge (Phase 2); how the store is versioned and attributed (Phase 3).

## Phase 1 — Thin slice with stand-ins

The four modules exist as small, tested libraries over the slice ADR-0003
chose. The language edge is a stand-in that returns a known translation.
Facts are hand-written files with a `source` field.

Order, from the PRD's Testing Decisions:

1. **Knowledge store.** A fact added with a source reads back with that
   source. A fact that contradicts the store is flagged, not accepted
   silently. Anyone can list every fact and rule.
2. **Reasoner.** A supported query returns an answer and a derivation; every
   fact and rule in the derivation exists in the store. An unsupported query
   returns "unknown". Replaying the derivation by hand yields the same answer.
3. **Constraints.** A derivation that would violate an inviolable rule fails.
   A later fact that would override one is rejected. Anyone can list the
   inviolable rules.
4. **Correction.** Correcting one fact changes only the answers that depend
   on it.
5. **Language edge, stand-in.** The translation is shown. A translation that
   references a fact not in the store is rejected.
6. **Public rules.** `docs/RULES.md` states the reasoning rules in plain
   language, so a reader can check that an answer follows from knowledge and
   nothing else.

Exit: one command loads the slice, answers a query with its derivation,
answers "unknown" to a query outside the slice, and refuses a derivation that
would break an inviolable rule.

## Phase 2 — Replace the stand-ins

- A real language model at the edge, with its output checked against the
  store before use (a `decision` issue first: which model, how checked).
- Source vetting and contradiction resolution between sources.
- A second knowledge slice, to prove the first one was not special-cased.

Exit: a person asks in plain language, sees the translation, gets an answer
with a derivation they can follow, and gets "unknown" where the store is silent.

## Phase 3 — Open to inspect and change

Versioned store with attribution; anyone can read every fact, rule, and
inviolable rule from outside the project; a change to the inviolable set is
its own reviewed event. Only then: the questions the PRD parked (multi-user
editing, permissions, interfaces).
