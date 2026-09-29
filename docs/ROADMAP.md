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

## Landscape

Outside work that bears on Neeti's fixed requirement: every answer names the
exact facts and rules that produced it, a person can replay it, "unknown" is
an answer, and inviolable rules cannot be derived around. One line per
finding, with what it offers or where it stops short. Appended by the PRD
cycle ([docs/automations/prd-cycle.md](automations/prd-cycle.md)); nothing
here decides anything.

Scanned 2026-09-29.

- **ASP with justifications.** [s(CASP)](https://github.com/SWI-Prolog/sCASP) (Arias et al., [ICLP 2020](https://cliplab.org/papers/sCASP-ICLP2020/TC-explainCASP.pdf)) runs ASP goal-directed without grounding and prints a minimal justification tree per answer, including for negated literals, as literals or in English (`--tree --human`); closest existing engine to "an answer that names its rules", at the cost of a Prolog runtime and dual rules a hand-replayer must understand.
- **ASP with justifications.** [xclingo 2](https://github.com/bramucas/xclingo2) (Cabalar, Fandinno, Muñiz) builds derivation trees from clingo answer sets using rule labels written as comments, so the annotated program still runs in plain clingo; only labelled rules appear and negated literals are not justified, so the trace is a selection, not the whole derivation.
- **Datalog with provenance.** [Soufflé provenance](https://souffle-lang.github.io/provenance) (Zhao, Subotić, Scholz, [TOPLAS 2020](https://psubotic.github.io/papers/toplas20.pdf)) instruments bottom-up evaluation so any tuple yields a minimal-height proof tree on demand at ~1.3× overhead, with an `explainnegation` mode for absent tuples; shows a derivation can be an engine feature rather than an add-on, but the toolchain is C++ and the proof is a tree over ground tuples, not an ordered list with sources.
- **Datalog with provenance.** [Nemo](https://github.com/knowsys/nemo) (Krötzsch group, [KR 2024](https://iccl.inf.tu-dresden.de/web/Nemo/en)) is a Rust rule engine whose `--trace 'fact'` prints the proof tree for one derived fact, with a browser visualiser ([nev](https://github.com/imldresden/nev), EvonNemo, XLoKR 2024); a ready Datalog candidate that already emits per-fact derivations, though it offers stratified negation, which Neeti would have to forbid by convention.
- **LLM writes the program, solver runs it.** [LLM+ASP](https://aclanthology.org/2026.findings-acl.1151/) (Ishay and Lee, ACL Findings 2026) has the model translate a question into an ASP program and uses clingo's error feedback in a self-correction loop, with no per-task engineering; the "LLM+ASP hybrid" candidate in concrete form, but the program, and the final human-readable answer, are model-written per question, so nothing in it is a store the answer is checked against.
- **LLM writes the program, solver runs it.** [LLM-ARC](https://arxiv.org/abs/2406.17663) (Kalyanpur et al., 2024) is the same loop as actor-critic with clingo as the critic and model-written tests; [solver-in-the-loop tuning](https://arxiv.org/abs/2512.17093) (Dec 2025) shows small open models can be fine-tuned on clingo feedback to emit ASP; relevant to a Telivi-sized model at the edge, with verification staying in the solver.
- **LLM writes the program, solver runs it.** [Faithful CoT](https://arxiv.org/abs/2301.13379) (Lyu et al., 2023) translates a question into Datalog, Python, or PDDL and executes it deterministically, so the answer is faithful to the chain; the chain is not faithful to any knowledge base, since the model may invent a fact inside the program, which is exactly the gap PRD story 15 closes.
- **LLM writes the program, solver runs it.** [Language Models and Logic Programs for Trustworthy Tax Reasoning](https://arxiv.org/abs/2508.21051) (Aug 2025) has the model write Prolog over US tax statutes, SWI-Prolog executes it, and a program that fails to run is a refusal priced into a cost model against wrong answers; a worked example of refusal-as-answer and of statute-to-rule slices, still with the model as the source of the rules.
- **LLM proposes, symbolic check accepts each step.** [SymStep](https://arxiv.org/abs/2607.23055) (Jul 2026) has the model make one atomic claim per turn and a deterministic constraint propagator accept it, reject it with the exact contradiction, or cascade its consequences; whole-program translation (Logic-LM, LINC) scored 0% on the same puzzles. The accepted-claim sequence is a derivation checked line by line against a fixed rule set: the shape PRD story 15 asks for.
- **LLM proposes, symbolic check accepts each step.** [LLM-Modulo](https://proceedings.mlr.press/v235/kambhampati24a.html) (Kambhampati et al., ICML 2024) is the position paper: models generate and translate, sound external critics verify, the model never verifies itself; this is the PRD's language edge stated as an architecture.
- **LLM proposes, symbolic check accepts each step.** [L4L](https://arxiv.org/abs/2511.21033) (Nov 2025) has model agents formalise statutes into SMT constraints validated against known cases before use, then a solver adjudicates prosecutor- and defence-agent arguments; the validate-before-entry step is a pattern for Phase 2 source vetting.
- **LLM proposes, symbolic check accepts each step.** Lean-based provers ([AlphaProof](https://www.cs.virginia.edu/~rmw7my/Courses/AgenticAISpring2026/Major%20Breakthroughs%20in%20Lean%204-Based%20Auto-Formalized%20Mathematics.html), [DeepSeek-Prover-V2](https://github.com/deepseek-ai/DeepSeek-Prover-V2)) are the strong form of model-proposes-kernel-checks: a perfect verifier means nothing wrong is ever accepted; a Lean proof is machine-checkable but not hand-followable, so Neeti's requirement is stricter than theirs.
- **Knowledge graphs with provenance.** [Wikidata references](https://www.wikidata.org/wiki/Help:Sources) attach `stated in` / `reference URL` to each statement, and [ProVe](https://doi.org/10.3233/SW-233467) checks automatically whether the cited page supports the triple; fact-level sourcing at a billion statements, and an automated source-vetting pattern for Phase 2, with the known gap that references cannot be attached to qualifiers.
- **Knowledge graphs with provenance.** [PROV-K](https://doi.org/10.5281/zenodo.15187372) (Giachelle, Marchesin, Menotti, Silvello, [IJDL 2025](https://link.springer.com/article/10.1007/s00799-025-00431-x)) extends PROV-O to assertions drawn from several sources with supporting and conflicting evidence and trust between agents; a vocabulary for the Phase 2 question of how a contradiction between sources is recorded before it is resolved.
- **Knowledge graphs with provenance.** [PROV-STAR](https://github.com/HenrikDibowski/PROV-STAR/) (Dibowski, [FOIS 2024](https://doi.org/10.3233/FAIA241309)) tracks every change to a graph at triple level with RDF-star and restores any past version with one query; a pattern for Phase 3 versioning and attribution.
