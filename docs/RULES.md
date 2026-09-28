# The reasoning rules

The rules Neeti follows when it answers, stated in plain language so a reader
can check that an answer follows from the knowledge and nothing else
([PRD user story 16](PRD.md#user-stories)).

Every rule here comes from a line in [docs/PRD.md](PRD.md). Each rule quotes
that line and links to the section it is in. Once the module that enforces a
rule exists, the rule will also link to the test that enforces it; until then
it names the issue the test will land with. This file states rules; it does
not decide anything the PRD lists as open (see
[What this file does not decide](#what-this-file-does-not-decide)).

## Rule 1 — Knowledge is the source of truth. A model is not.

An answer comes from facts and rules that are written down in the knowledge
store. It never comes from a language model, from a default, or from a guess.
A language model may translate a question into a query or a derivation into
plain language, but it is never the source of an answer, and anything it
produces is checked against the store before it is used.

To check it: every fact and every rule the answer used must exist in the
store. If an answer rests on something that is not in the store, it breaks
this rule.

- PRD: "Knowledge is the source of truth. A language model is not."
  ([Decisions already made](PRD.md#implementation-decisions))
- PRD: "It is never the source of an answer. Everything it produces is checked
  against the knowledge base before it is used." ([Solution](PRD.md#solution))
- PRD: user stories 1 and 15 ([User stories](PRD.md#user-stories))
- Test: none yet. Lands with the reasoner
  ([#6](https://github.com/naveenreddyalka/neeti/issues/6)) and the language
  edge ([#9](https://github.com/naveenreddyalka/neeti/issues/9)).

## Rule 2 — Every answer carries its derivation.

A derivation is the list of facts and rules the answer used, in the order they
were applied. An answer without a derivation is not an answer. The reasoner
never produces an answer that is not in the derivation.

- PRD: "Every answer carries its derivation. An answer without a derivation is
  not an answer." ([Decisions already made](PRD.md#implementation-decisions))
- PRD: "Returns the answer together with the derivation: the facts and rules
  used, in order. ... Never produces an answer that is not in the derivation."
  ([Reasoner](PRD.md#implementation-decisions))
- PRD: user story 2 ([User stories](PRD.md#user-stories))
- Test: none yet. Lands with the reasoner
  ([#6](https://github.com/naveenreddyalka/neeti/issues/6)).

### How to replay a derivation by hand

A person must be able to recompute any answer from its derivation without the
machine. The steps are the same whatever form the facts and rules take:

1. Read the derivation from top to bottom. Each line is either a fact taken
   from the store or a rule applied to lines above it.
2. For each fact, find it in the store. It must be there, word for word, with
   its source. If it is not, stop: the answer does not follow.
3. For each rule, find it in the store, then apply it yourself to the lines it
   names. It must use only lines that appear above it in the derivation. If
   it needs something that is not above it or not in the store, stop: the
   answer does not follow.
4. When you reach the last line, what you have written down is the answer. It
   must match the answer Neeti gave. If it does not, the answer does not
   follow.

If you finish all four steps without stopping, the answer follows from the
knowledge and nothing else.

- PRD: "A person must be able to recompute any answer by hand from the
  derivation." ([Decisions already made](PRD.md#implementation-decisions))
- PRD: "Replaying the derivation by hand yields the same answer."
  ([Testing decisions](PRD.md#testing-decisions))
- PRD: user story 3 ([User stories](PRD.md#user-stories))
- Test: none yet. Lands with the reasoner
  ([#6](https://github.com/naveenreddyalka/neeti/issues/6)).

## Rule 3 — "Unknown" means the store is silent.

When the facts and rules in the store do not support an answer, Neeti answers
"unknown". "Unknown" is a valid answer. It means exactly one thing: nothing in
the store derives an answer to this query. It does not mean "probably no", and
it is never replaced by a guess to fill the gap.

To check it: if Neeti answered "unknown", there should be no derivation from
the store that reaches an answer. If Neeti gave an answer, its derivation
(Rule 2) shows the store was not silent.

- PRD: "'Unknown' is a valid answer. Guessing is not."
  ([Decisions already made](PRD.md#implementation-decisions))
- PRD: "If the knowledge base does not support an answer, Neeti says so instead
  of guessing." ([Solution](PRD.md#solution))
- PRD: user story 4 ([User stories](PRD.md#user-stories))
- Test: none yet. Lands with the reasoner
  ([#6](https://github.com/naveenreddyalka/neeti/issues/6)).

## Rule 4 — An inviolable rule holds everywhere, and nothing overrides it.

An inviolable rule is a rule that is codified once and holds in every
derivation. It is a constraint, not a suggestion:

- A derivation that would violate an inviolable rule fails. There is no
  answer, and no derivation is returned as if there were.
- A fact or rule added later that would override an inviolable rule is
  rejected. The rule cannot be eroded by adding to the store.
- No output of a language model can override an inviolable rule.
- Anyone can read the whole inviolable set.

Which rules are in the set is an open decision. This file does not list them.
The inviolable set is the set defined in ADR-0003 in
[docs/decisions/](decisions/) once that decision is made
([#4](https://github.com/naveenreddyalka/neeti/issues/4)). Adding, removing,
or changing a rule in that set needs an issue that names it
([AGENTS.md § Hard limits](../AGENTS.md#hard-limits)).

To check it: for each inviolable rule in the set, confirm the answer and every
step of its derivation is consistent with it. If any step is not, the
derivation should have failed.

- PRD: "Inviolable rules cannot be overridden by any fact, any rule, or any
  model output." ([Decisions already made](PRD.md#implementation-decisions))
- PRD: "A small set of rules is inviolable. They are codified once and hold
  everywhere. A derivation that would violate one fails. Nothing added later,
  and nothing a language model produces, can override them."
  ([Solution](PRD.md#solution))
- PRD: "The inviolable rules. A derivation that violates one fails. A fact or
  rule added later cannot override one. The set is readable by anyone."
  ([Constraints](PRD.md#implementation-decisions))
- PRD: user stories 9, 10, 11, and 12 ([User stories](PRD.md#user-stories))
- Test: none yet. Lands with the constraints module
  ([#7](https://github.com/naveenreddyalka/neeti/issues/7)).

## Rule 5 — Every fact names its source.

A fact in the store carries the source it came from, so a reader can judge it.
A fact with no source is not a fact the store accepts, and cannot appear in a
derivation.

To check it: every fact line in a derivation should point to a source. When
you look the fact up in the store (Rule 2, step 2), the source should be
there.

- PRD: "Each fact names its source."
  ([Decisions already made](PRD.md#implementation-decisions))
- PRD: "Holds facts and rules. Each fact names its source. Anyone can read the
  whole store." ([Knowledge store](PRD.md#implementation-decisions))
- PRD: user story 6 ([User stories](PRD.md#user-stories))
- Test: none yet. Lands with the knowledge store
  ([#5](https://github.com/naveenreddyalka/neeti/issues/5)).

## Checking an answer

Put together, checking that an answer follows from the knowledge and nothing
else is:

1. It has a derivation (Rule 2), or it is "unknown" (Rule 3).
2. Every fact in the derivation is in the store, with its source (Rules 1
   and 5).
3. Every rule in the derivation is in the store and, applied by hand to the
   lines above it, gives the next line (Rule 2).
4. The last line is the answer Neeti gave (Rule 2).
5. Nothing in the derivation violates an inviolable rule (Rule 4).

If all five hold, the answer follows from the knowledge and nothing else. If
any one fails, it does not, and the failure points at the step to fix.

## What this file does not decide

The PRD lists decisions that are not made yet
([Decisions not made yet](PRD.md#implementation-decisions)). This file states
the rules that hold regardless of how those decisions go, and takes no
position on them:

- The formalism the facts and rules are written in, and the solver
  ([#3](https://github.com/naveenreddyalka/neeti/issues/3), ADR-0002).
- Which rules are in the inviolable set, and who decides
  ([#4](https://github.com/naveenreddyalka/neeti/issues/4), ADR-0003).
- The implementation language and toolchain
  ([#2](https://github.com/naveenreddyalka/neeti/issues/2), ADR-0001).
- Where facts come from, how a source is vetted, how a contradiction between
  sources is resolved, which language model sits at the edge, and how the
  store is versioned.

When one of these is decided, the ADR in [docs/decisions/](decisions/) is the
place it is recorded. The rules above do not change; only the links to the
tests that enforce them get filled in.
