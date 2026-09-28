# Neeti PRD

**నీతి** is Telugu for ethics, principle, and codified right conduct.

Neeti is a knowledge base of codified facts and rules, and a reasoner that derives answers from them. Every answer traces back to the facts and rules it used. A person can follow that trace and compute the same answer by hand. Some rules are inviolable: no other fact, no other rule, and no language model can override them.

## Problem Statement

A language model predicts the next token. Nothing inside it is a fact that can be pointed to, checked, or guaranteed to hold. "Do not hurt people" is a tendency it learned, not a rule it obeys, so it can be missed. A law, a theorem, or a physical constant is a pattern in its weights, not a statement it can show you and defend.

When such a model gives an answer, no one can open it up and see why. When it gets a fact wrong, no one can correct that one fact. When it violates a rule that should never be violated, there is no place in the system where that rule was written down.

## Solution

Neeti keeps knowledge as knowledge. Facts and rules are written down in a form a machine can reason over and a person can read.

An answer is a derivation: the facts and rules used, in the order they were applied. A person can read the derivation and reach the same conclusion without the machine. If the knowledge base does not support an answer, Neeti says so instead of guessing.

A small set of rules is inviolable. They are codified once and hold everywhere. A derivation that would violate one fails. Nothing added later, and nothing a language model produces, can override them.

A language model may sit at the edge: turning a question into a query the reasoner understands, or a derivation into plain language. It is never the source of an answer. Everything it produces is checked against the knowledge base before it is used.

## User Stories

1. As a person asking a question, I want the answer to come from written facts and rules, so that it is not a guess.
2. As a person asking a question, I want to see which facts and rules produced the answer, so that I can check it.
3. As a person asking a question, I want to follow the derivation by hand and reach the same answer, so that I do not have to trust the machine.
4. As a person asking a question, I want "unknown" when the knowledge base does not support an answer, so that a gap is not filled with a guess.
5. As a person adding knowledge, I want to write a fact once and have it hold everywhere it applies, so that knowledge is not duplicated or lost.
6. As a person adding knowledge, I want each fact to name its source, so that a reader can judge it.
7. As a person adding knowledge, I want to correct one fact and have only the answers that depend on it change, so that a fix is precise.
8. As a person adding knowledge, I want a fact that contradicts the existing base to be flagged, so that the base stays consistent.
9. As anyone, I want a rule I mark inviolable to hold in every derivation, so that it can never be missed.
10. As anyone, I want a derivation that would violate an inviolable rule to fail, so that the rule is a constraint and not a suggestion.
11. As anyone, I want an attempt to override an inviolable rule with a later fact to fail, so that the rule cannot be eroded.
12. As anyone, I want to read every fact, rule, and inviolable rule in the base, so that the knowledge is open to inspect.
13. As a person asking in plain language, I want my question translated into a query the reasoner understands, so that I do not have to learn the formal language.
14. As a person asking in plain language, I want the translation shown to me, so that I can see whether the machine understood the question.
15. As anyone, I want anything a language model produces to be checked against the knowledge base before it is used, so that the model cannot invent a fact.
16. As anyone, I want the reasoning rules in the open, so that I can check that an answer follows from the knowledge and nothing else.

## Implementation Decisions

Neeti is four modules. Each one hides a hard problem behind a small interface.

- **Knowledge store.** Holds facts and rules. Each fact names its source. Anyone can read the whole store. Adding a fact that contradicts the store is flagged, not silently accepted.
- **Reasoner.** Takes a query and derives an answer from the store. Returns the answer together with the derivation: the facts and rules used, in order. Returns "unknown" when the store does not support an answer. Never produces an answer that is not in the derivation.
- **Constraints.** The inviolable rules. A derivation that violates one fails. A fact or rule added later cannot override one. The set is readable by anyone.
- **Language edge.** Optional. Translates a plain-language question into a query, and a derivation into plain language. Shows its translation. Anything it produces that would enter the store or affect an answer is checked against the store first.

Decisions already made:

- Knowledge is the source of truth. A language model is not.
- Every answer carries its derivation. An answer without a derivation is not an answer.
- A person must be able to recompute any answer by hand from the derivation.
- "Unknown" is a valid answer. Guessing is not.
- Inviolable rules cannot be overridden by any fact, any rule, or any model output.
- Each fact names its source.
- The formalism is a Datalog fragment: ground facts, explicitly negated facts, Horn rules without default negation or function symbols, and integrity constraints for the inviolable rules ([ADR-0002](decisions/0002-formalism-and-solver.md)).
- The solver is a purpose-built forward chainer that records a justification for every derived fact; no external reasoner is a runtime dependency ([ADR-0002](decisions/0002-formalism-and-solver.md)).

Decisions not made yet, and not to be invented in code until a later issue chooses them:

- Where facts come from, and how a source is vetted before its facts enter the store.
- How a contradiction between two sources is resolved.
- Which rules are in the inviolable set, and who decides.
- Which language model, if any, sits at the edge, and how its output is checked.
- How the store is versioned and how a change is attributed.

## Testing Decisions

Test the behavior a person asking or a person reading can observe. Do not test the internal structure of a module.

- **Knowledge store.** A fact added with a source can be read back with that source. A fact that contradicts the store is flagged. Correcting one fact changes only the answers that depend on it.
- **Reasoner.** A query the store supports returns an answer and a derivation. Every fact and rule in the derivation exists in the store. A query the store does not support returns "unknown". Replaying the derivation by hand yields the same answer.
- **Constraints.** A derivation that would violate an inviolable rule fails. A later fact that would override an inviolable rule is rejected. A reader can list every inviolable rule.
- **Language edge.** Use a stand-in model that returns a known translation. The translation is shown. A translation that references a fact not in the store is rejected.

There is no existing suite. These are the first tests, and they land with the module they cover.

## Out of Scope

- Training a model, or fine-tuning one.
- A general-purpose chatbot.
- Deciding the full inviolable rule set in the first implementation issues.
- Encoding all of law or all of science. The first issues use a small, chosen slice.
- A graphical interface.
- Multi-user editing, permissions, or accounts.

## Further Notes

The name is the Telugu word నీతి (neeti): ethics, principle, right conduct, and by extension policy and codified guidance. It names the part of the project that makes it different from a knowledge graph: rules that are written down once and cannot be violated.

The comparison to a code of conduct, such as the Ten Commandments, is about the form, not the content. A rule like "do not hurt people" should be a constraint the system cannot derive around, not a tendency it usually follows.

Neeti sits beside Telivi and Daari. Telivi is a model people own. Daari routes requests to the right model. Neeti is what a model can be checked against.
