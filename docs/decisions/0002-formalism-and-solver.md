# ADR-0002: knowledge formalism and solver

**Status:** proposed
**Issue:** #3
**Date:** 2026-09-28

## Context

The PRD leaves open "the formalism" and "the solver or engine". Everything in
Phase 1 (store, reasoner, constraints) depends on both. The binding
requirements, from the PRD and issue #3:

1. **Derivation.** Every answer lists the facts and rules used, in order, and a
   person can recompute the answer by hand from that list alone.
2. **Unknown.** "Not derivable" is a first-class answer, distinct from "false".
   A gap in the store must never be turned into a conclusion.
3. **Hard constraints.** Inviolable rules are constraints, not defaults. A
   derivation that would violate one fails; a later fact or rule cannot
   override one.
4. **Contradiction.** A fact that contradicts the store is flagged on entry.
5. **Readable.** Every fact, rule, and inviolable rule is readable text on disk.
6. **Dependencies.** AGENTS.md forbids a runtime dependency no issue has named,
   so each option names the dependency it would need.

ADR-0001 (#2) chooses the implementation language in parallel; this ADR holds
for any mainstream language and states the interaction under Consequences.

Two observations shape the comparison. Requirement 2 rules out *default
negation* (negation as failure, `not p`) in rule bodies: it is exactly the
operator that turns "absent from the store" into a conclusion. Without it the
formalism is monotonic, which makes requirement 3 cheap: adding knowledge can
only add conclusions, so "override" has no mechanism, and a constraint either
becomes violated (rejected, with the derivation) or stays satisfied. And
requirement 1 is the product, not a debugging aid: an engine whose proof output
is an add-on makes the core deliverable depend on that engine's weakest part.

## Options

### A. Datalog with an existing engine (Soufflé, or an in-process library)

Ground facts, Horn rules, least fixpoint. *Derivation:* Soufflé has a
provenance/`explain` mode that yields proof trees
([docs](https://souffle-lang.github.io/provenance)); in-process libraries
(`crepe`/`ascent` for Rust, `pyDatalog`, and the like) mostly do not record
justifications, so derivations would be reconstructed afterwards. *Unknown:*
closed-world; "not derivable" is reported as an empty result the caller must
reinterpret, and stratified `not` must be forbidden by convention.
*Constraints and contradiction:* not native; encoded as a `violation` relation
checked after evaluation, plus a convention for negative facts. *Dependency:*
Soufflé is a C++ toolchain driven through files and a CLI; a library ties the
choice to ADR-0001's language. *Rules out:* nothing, but the four things Neeti
cares most about are conventions layered on top of the engine.

### B. Answer set programming (clingo)

Stable-model semantics with classical negation `-p`, integrity constraints
`:- body.`, and default negation `not`. *Derivation:* clingo outputs answer
sets, not proofs; explanations need a separate tool (xclingo) or a
meta-encoding, and once `not` is used a stable model is justified by a fixpoint
over the whole program that a person cannot follow line by line. *Unknown:*
only by forbidding `not`, at which point there is exactly one answer set and
ASP is being used as Datalog. *Constraints:* native, exactly the right shape.
*Contradiction:* native (`p` and `-p`, or a violated constraint, mean no answer
set), reported as "unsatisfiable" with no derivation unless one is added.
*Dependency:* clingo, C++ with Python bindings
([potassco.org](https://potassco.org/clingo/)). *Rules out:* nothing; but the
power it adds (choice, defaults, optimisation) is power the PRD forbids.

### C. Prolog (SWI-Prolog, or a hand-written interpreter)

Backward chaining by SLD resolution. *Derivation:* the resolution trace is a
proof, readable when the program is pure. *Unknown:* failure means "false" and
`\+` is negation as failure; unknown-versus-false would be a convention against
the language's own semantics. *Constraints and contradiction:* not native;
goals run after each update. Recursive rules over cyclic data do not terminate
without tabling; clause order changes results; cut is impure. *Dependency:*
SWI-Prolog is a large runtime; a hand-written interpreter inherits the
semantics above. *Rules out:* guaranteed-terminating, order-independent
evaluation.

### D. Description logic (OWL 2 with a DL reasoner)

Classes, properties, individuals, axioms. *Derivation:* reasoners produce
*justifications* (minimal axiom sets), not ordered steps; a tableau cannot be
followed by hand. *Unknown:* open-world semantics fits requirement 2 exactly.
*Constraints:* disjointness and cardinality axioms; violation makes the
ontology inconsistent. *Contradiction:* native consistency check.
*Dependency:* HermiT, Pellet, or ELK (Java), or Owlready2 wrapping HermiT.
*Rules out:* n-ary relations and general rules (SWRL is a bolt-on); "fact and
rule" maps awkwardly onto class axioms; Turtle/Manchester syntax is verbose.

### E. A purpose-built minimal rule engine over a fixed Datalog fragment

Define the formalism first, then write the smallest engine that evaluates it.
The formalism is a fragment of Datalog that is also valid ASP syntax:

- **Facts:** ground atoms `p(a, b).` and explicitly negated ground atoms
  `-p(a, b).` Every fact carries a source (syntax decided in #5).
- **Rules:** `head :- lit1, ..., litn.` where each literal is an atom or an
  explicitly negated atom, every head variable appears in the body, and there
  are no function symbols, arithmetic, aggregates, or **default negation**.
- **Constraints:** `:- lit1, ..., litn.` The inviolable set is a file of these
  that the store cannot modify (#7).

*Derivation:* semi-naive forward chaining to the least fixpoint, in file order
so it is deterministic. Each derived atom records the rule instance and premise
atoms that produced it. The answer's derivation is the proof tree of the
queried atom, linearised premises-first: each line is a stored fact with its
source, or `rule R with X=a, Y=b applied to lines i, j`. A person checks a line
by substituting into the rule body and finding the premises above it; #6
replays it mechanically. *Unknown:* `p` derived is true, `-p` derived is false,
neither is unknown. Absence is never a premise, so nothing follows from a gap.
*Constraints:* after any fixpoint (store load, fact entry, a query carrying
assumed facts), a constraint whose body is derivable fails the operation and
returns the body's derivation, naming the rule. Monotonicity means a later fact
cannot make a violation disappear, so "override" cannot be expressed.
*Contradiction:* `p` and `-p` both derivable, or a constraint body derivable,
is flagged on entry with both derivations and the store left unchanged.
*Correction (#8):* the recorded justifications are the dependency graph.
*Dependency:* none. The engine is a few hundred lines in any language: parser,
matcher over ground atoms, semi-naive loop, proof extraction. *Rules out:*
arithmetic, aggregation, defaults, and scale; each can be added by a later ADR
that widens the fragment, and none is needed for a "small, chosen slice".

## Recommendation

**Option E.** The formalism is the Datalog fragment above; the solver is a
purpose-built forward chainer that records a justification for every derived
atom. It wins because the PRD's requirements are its semantics rather than
conventions: unknown falls out of removing `not`, non-override falls out of
monotonicity, constraints and contradiction are one fixpoint check, and the
derivation is the engine's own data structure. Every other option requires
forbidding half the engine (B, C), reconstructing proofs the engine did not
keep (A, B, D), or a runtime dependency larger than Neeti for a slice that
fits in a text file. Because the fragment is valid clingo syntax, a later issue
may add clingo as a *test-only* oracle to cross-check the fixpoint; that would
not change this decision.

## Consequences

- #5, #6, #7, #8 are unblocked on the formalism side. #5 fixes the source
  syntax; #7 loads constraints from a separate file; the derivation format
  (stored fact with source, or rule id + substitution + premise line numbers)
  is the contract #6 tests.
- No runtime dependency. ADR-0001 chooses the language freely and the engine
  is written in it. A mature Datalog library in that language does not change
  this ADR: the on-disk format is engine-independent, and a library may replace
  the hand-written fixpoint later only if it emits the same justifications.
- Harder: anything needing arithmetic, counting, or defaults. That is a new
  ADR that widens the fragment and shows the derivation stays hand-followable.
- `docs/RULES.md` (#10) can state the reasoning rules in three sentences: a
  fact holds if it is in the store; a rule's head holds when every body literal
  holds; nothing else holds.
