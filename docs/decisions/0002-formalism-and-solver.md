# ADR-0002: knowledge formalism and solver — first candidate

**Status:** proposed, held — names the first candidate to try, not a settled
decision; the PRD's "What we are trying to find out" section owns the question
**Issue:** #3
**Date:** 2026-09-28

## Context

The PRD leaves open "the formalism" and "the solver or engine". Neeti's
approach is itself a research question: a symbolic solver, a language model
(possibly Telivi's), or a model combined with a solver, under the fixed
requirement that every answer names the exact facts and rules that produced
it. This ADR does not close that question: it compares the symbolic options,
names a **first candidate** to try on the ADR-0003 slice, and says what the
slice must show for it to stand. The binding requirements (PRD, issue #3):

1. **Derivation.** Every answer lists the facts and rules used, in order, and a
   person can recompute the answer by hand from that list alone.
2. **Unknown.** "Not derivable" is a first-class answer, distinct from "false".
   A gap in the store must never be turned into a conclusion.
3. **Hard constraints.** Inviolable rules are constraints, not defaults. A
   derivation that would violate one fails; a later fact cannot override one.
4. **Contradiction.** A fact that contradicts the store is flagged on entry.
5. **Readable.** Every fact, rule, and inviolable rule is readable text on disk.
6. **Dependencies.** AGENTS.md forbids a runtime dependency no issue has named,
   so each option names the dependency it would need.

ADR-0001 (#2) chooses the implementation language in parallel; this ADR holds
for any mainstream language (see Consequences). Two observations shape the
comparison. Requirement 2 rules out *default negation* (`not p`) in rule
bodies: it turns "absent from the store" into a conclusion. Without it the
formalism is monotonic: adding knowledge only adds conclusions, so "override"
has no mechanism and a constraint is either violated (rejected, with the
derivation) or satisfied. And requirement 1 is the product, not a debugging
aid: an engine whose proof output is an add-on rests on its weakest part.

## Options

### A. Datalog with an existing engine (Soufflé, or an in-process library)

Ground facts, Horn rules, least fixpoint. *Derivation:* Soufflé has a
provenance/`explain` mode ([docs](https://souffle-lang.github.io/provenance));
in-process libraries (`crepe`/`ascent`, `pyDatalog`) mostly do not record
justifications. *Unknown:* closed-world; "not derivable" is an empty result the
caller reinterprets, and stratified `not` must be forbidden by convention.
*Constraints and contradiction:* a `violation` relation checked after
evaluation, plus a convention for negative facts. *Dependency:* Soufflé is a
C++ toolchain driven through files; a library ties the choice to ADR-0001's
language. *Rules out:* nothing, but the four things Neeti cares most about are
conventions layered on the engine.

### B. Answer set programming (clingo)

Stable-model semantics with classical negation `-p`, integrity constraints
`:- body.`, and default negation `not`. *Derivation:* answer sets, not proofs;
explanations need xclingo or a meta-encoding, and with `not` a stable model is
justified by a whole-program fixpoint a person cannot follow line by line.
*Unknown:* only by forbidding `not`, leaving one answer set and ASP used as
Datalog. *Constraints:* native, the right shape. *Contradiction:* native,
reported as "unsatisfiable" without a derivation. *Dependency:* clingo, C++
with Python bindings ([potassco.org](https://potassco.org/clingo/)). *Rules
out:* nothing; its extra power (choice, defaults) is power the PRD forbids.

### C. Prolog (SWI-Prolog, or a hand-written interpreter)

SLD resolution. *Derivation:* the trace is a proof when the program is pure.
*Unknown:* failure means "false"; `\+` is negation as failure. *Constraints
and contradiction:* goals run after each update. Recursion over cyclic data
does not terminate without tabling; clause order changes results.
*Dependency:* SWI-Prolog, or an interpreter with the same semantics. *Rules
out:* terminating, order-independent evaluation.

### D. Description logic (OWL 2 with a DL reasoner)

*Derivation:* justifications (minimal axiom sets), not ordered steps; a
tableau cannot be followed by hand. *Unknown:* open-world, fits requirement 2
exactly. *Constraints and contradiction:* disjointness/cardinality axioms and
a native consistency check. *Dependency:* HermiT, Pellet, or ELK (Java).
*Rules out:* n-ary relations and general rules; verbose on disk.

### E. A purpose-built minimal rule engine over a fixed Datalog fragment

The formalism is a fragment of Datalog that is also valid ASP syntax:

- **Facts:** ground atoms `p(a, b).` and explicitly negated ground atoms
  `-p(a, b).` Every fact carries a source (syntax decided in #5).
- **Rules:** `head :- lit1, ..., litn.`; literals are atoms or explicitly
  negated atoms, every head variable appears in the body, and there are no
  function symbols, arithmetic, aggregates, or **default negation**.
- **Constraints:** `:- lit1, ..., litn.` The inviolable set is a file of these
  that the store cannot modify (#7).

*Derivation:* semi-naive forward chaining to the least fixpoint, in file order
so it is deterministic. Each derived atom records the rule instance and
premises that produced it; the answer's derivation is that proof tree,
linearised premises-first: each line is a stored fact with its source, or
`rule R with X=a, Y=b applied to lines i, j`. *Unknown:* `p` derived is true,
`-p` derived is false, neither is unknown; absence is never a premise.
*Constraints:* after any fixpoint (load, entry, a query carrying assumed
facts), a constraint whose body is derivable fails the operation with the
body's derivation, naming the rule; monotonicity means a later fact cannot
make a violation disappear. *Contradiction:* `p` and `-p` both derivable, or a
constraint body derivable, is flagged on entry with both derivations.
*Correction (#8):* the justifications are the dependency graph. *Dependency:*
none; a few hundred lines in any language. *Rules out:* arithmetic,
aggregation, defaults, and scale, until an ADR widens the fragment.

## Recommendation: first candidate

Try **Option E** first on the ADR-0003 slice. Among the symbolic options it is
the one where the PRD's requirements are the semantics rather than conventions:
unknown falls out of removing `not`, non-override falls out of monotonicity,
constraints and contradiction are one fixpoint check, and the derivation is
the engine's own data structure. It adds no dependency, so trying it costs
least and biases the research least; since the fragment is valid clingo
syntax, clingo can later serve as a test-only oracle.

## Candidates the PRD research section will weigh

The PRD's "What we are trying to find out" section, once merged, owns this
list; it is recorded here so the first slice is built to compare them fairly.

- **ASP via clingo** (Option B): the same fragment plus defaults and choice if
  the slice needs them; the open cost is derivations.
- **LLM as translator with a symbolic verifier:** the model turns a question
  into a query and a derivation into prose; a symbolic engine (E or B) is the
  only source of answers and checks every model output. This is the PRD's
  language edge (#9); it composes with E rather than replacing it.
- **LLM+ASP hybrid:** the model proposes facts, rules, or candidate
  derivations; ASP checks them against the store and the inviolable set. Open:
  whether a proposal can be verified line by line, and how a rejection reads.

## Evidence from the first slice

Confirms E: the slice's facts and rules fit the fragment without encoding
tricks; every #6 test derivation replays by hand in under a page; #7
rejections name the inviolable rule with a derivation; a person unfamiliar
with the code reads a derivation and reaches the same answer.

Rejects or reopens E: a natural rule of the slice needs arithmetic,
aggregation, or defaults; derivations for ordinary questions grow too long to
follow by hand; a plain-language question cannot be translated into the
fragment without the model inventing a fact; or the same slice under clingo
yields answers E cannot reproduce.

## Consequences

- #5, #6, #7, #8 may proceed against E as the candidate; their tests are the
  evidence above. The derivation format (stored fact with source, or rule id +
  substitution + premise lines) is the contract #6 tests.
- No runtime dependency. ADR-0001 chooses the language freely; a Datalog
  library in it may replace the hand-written fixpoint only if it emits the
  same justifications.
- Closing the question is a follow-up ADR citing the slice evidence and the
  PRD research section; only then do "formalism" and "solver" move to
  "Decisions already made". [docs/RULES.md](../RULES.md) already states the
  reasoning rules in plain language; E is their direct mechanisation.
