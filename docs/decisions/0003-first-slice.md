# ADR-0003: First knowledge slice and the initial inviolable rules

**Status:** proposed
**Issue:** #4
**Date:** 2026-09-28

## Context

The PRD says the first issues "use a small, chosen slice" of knowledge and leaves
open "which rules are in the inviolable set, and who decides". The slice must be
small enough that a person hand-checks every derivation, and rich enough to
exercise every behavior in [PRD § Testing Decisions](../PRD.md#testing-decisions):
a supported query, an unsupported one ("unknown"), a contradiction flagged on
entry, a correction that changes only dependents, and an inviolable rule that a
query could try to derive around. Every fact names its source.

Two constraints from the sibling ADRs: ADR-0001 ([#2](https://github.com/naveenreddyalka/neeti/issues/2))
picks the language and ADR-0002 ([#3](https://github.com/naveenreddyalka/neeti/issues/3))
picks the formalism, in parallel with this one. This ADR therefore describes facts and rules
in plain language and must not need arithmetic on real numbers, geometry, or any
feature that would quietly pre-empt ADR-0002.

Who decides the inviolable set: the repository owner. An agent proposes in an ADR;
the owner accepts by merging; a change is a new `decision` issue that names the
rule and a new ADR that supersedes this one (AGENTS.md § Hard limits).

## Options

| Domain | Hand-check | "Unknown" natural | Contradiction natural | Correction natural | Inviolable rule is conduct, not bookkeeping | Real sources | No arithmetic |
|---|---|---|---|---|---|---|---|
| A. Kinship toy | yes | weak | yes | yes | no | no | yes |
| B. Constants and units | no | yes | yes | yes | partly | yes | no |
| C. Traffic law slice | partly | yes | weak | yes | yes, but the law has exemptions | yes | no |
| D. Chess subset | partly | no | no | no | yes | one | no |
| E. Menu allergen safety | yes | yes | yes | yes | yes | yes | yes |

### A. Kinship/genealogy toy
Parent and sex facts; grandparent, sibling, ancestor rules. Easy to hand-check
and to correct. Costs: a missing chain reads as "false" not "unknown", so the
slice never forces the distinction; the only candidate inviolable rules are
integrity constraints ("no one is their own ancestor"), so the result looks like
a knowledge graph, the very thing the PRD says Neeti is not. Sources are invented.

### B. Physical constants and unit rules
CODATA values, SI definitions, conversions. Sources are real and a cross-source
contradiction is free (CODATA 2014 vs 2018 for *G*). Costs: every derivation is
arithmetic on reals, so hand replay means floating point, and the reasoner needs
numeric support, which pre-empts ADR-0002. Dimensional homogeneity is a good
inviolable rule but it is a type rule, not conduct.

### C. A small body of traffic law
Speed limits, yielding, signals from one real code. Conduct-shaped with citable
sections. Costs: real statutes carry exemptions (emergency vehicles), so calling
any of them inviolable misrepresents the source; right-of-way needs geometry;
picking a jurisdiction and vetting statute text is Phase 2 work.

### D. Chess-rules subset
"Never leave your own king in check" has the right form. Costs: one source, so
no natural contradiction or correction; a closed world with no natural
"unknown"; move legality needs coordinate arithmetic.

### E. Menu allergen safety (recommended)
A small fictional canteen: dishes with recipe cards, ingredients, the fourteen
allergen classes of Regulation (EU) No 1169/2011 Annex II, guests with allergy
declarations, a menu, and orders. Querying "may this dish be served to this
guest" chains recipe → ingredient → class → declaration. Costs: the canteen,
its cards and its forms are fixtures; only the class list and most ingredient
classifications cite a real document. That is acceptable in Phase 1, where
source vetting is explicitly deferred.

## Recommendation

Option E. It is the only option that hits every column. Its inviolable rule is
literally "do not hurt a person" in the PRD's sense, it forces the false/unknown
distinction on the very first query a tester writes ("is this dish free of
peanuts?" is "no" for one dish and "unknown" for another), and it needs nothing
but relational chaining, so it steers ADR-0002 toward no engine in particular.

### The slice

Facts, by shape, each carrying a `source`. Relation names are plain words; the
encoding is ADR-0002's.

| Fact shape | Count | Source |
|---|---|---|
| *C is an allergen class* (the 14: cereals containing gluten, crustaceans, eggs, fish, peanuts, soybeans, milk, nuts, celery, mustard, sesame, sulphites, lupin, molluscs) | 14 | [Regulation (EU) No 1169/2011, Annex II](https://eur-lex.europa.eu/eli/reg/2011/1169/oj) |
| *ingredient I is in class C* | 6 | Annex II item number |
| *ingredient I is in none of the 14 classes* | 11 | supplier label, or "Annex II read in full" for plain foodstuffs |
| *dish D's current recipe card is R* | 6 | the recipe card |
| *recipe card R lists ingredient I* | 19 | the recipe card |
| *dish D is on today's menu* | 6 | menu board 2026-09-28 |
| *guest G declared allergen class C* / *guest G declared no allergen classes* | 4 | booking forms B-101..B-103 |
| *guest G ordered dish D* | 5 | order tickets |

Concrete contents. Dishes and their cards: `plain-rice` (basmati rice, salt);
`dal-makhani` (black lentils, butter, tomato, cream); `peanut-chutney` (roasted
peanuts, tamarind, green chilli, salt); `prawn-curry` (prawns, coconut milk,
mustard seeds, curry leaves); `roti` (wheat flour, water); `masala-dal` (red
lentils, onion, house spice blend). Classes: butter and cream → milk (item 7);
roasted peanuts → peanuts (item 5); prawns → crustaceans (item 2); mustard seeds
→ mustard (item 10); wheat flour → cereals containing gluten (item 1). All other
ingredients except `house spice blend` have an explicit "none of the 14" fact;
`house spice blend` has no classification at all. Note coconut milk is not milk
and not an Annex II nut: a person wrote that negative fact down and sourced it;
the machine does not infer it. Guests: Ravi declared peanuts (B-101); Lakshmi
declared milk and nuts (B-102); Tom declared none (B-103); Meena has no form on
file. Orders: Ravi → peanut-chutney, plain-rice; Lakshmi → prawn-curry; Tom →
masala-dal; Meena → plain-rice.

Ordinary rules:

- **R1** A dish *contains* class C when its current recipe card lists an ingredient in C.
- **R2** A dish is *fully classified* when it has a current recipe card and every listed ingredient has a class fact or a "none of the 14" fact.
- **R3** A dish is *free of* class C when it is fully classified and no listed ingredient is in C.
- **R4** A dish is *suitable* for a guest when the guest has a declaration on file and the dish is free of every class the guest declared.
- **R5** A dish *may be served* to a guest when it is on today's menu and the guest ordered it.

Consistency rules (ordinary; they drive flag-on-entry in the store): a dish has
exactly one current recipe card; an ingredient cannot be both in a class and in
none; a guest cannot both declare a class and declare none.

### The inviolable rules

- **I1 — Never serve a declared allergen.** A dish that contains an allergen class a guest has declared is never served to that guest. *Why inviolable:* R5 is permissive by design, and the natural later additions ("the guest waived it", "the chef confirmed") are one ordinary rule away from harming a person. The harm is to a person and irreversible. This is the PRD's motivating example in domain form.
- **I2 — Missing information is not safety.** A dish is served to a guest only when the base positively establishes that it is suitable for that guest (R4). No recipe card, an unclassified ingredient, or no declaration on file is never evidence of absence. *Why inviolable:* the most tempting shortcut in any formalism is a closed-world default ("not listed means not present"). As an ordinary rule I2 could be silently repealed by one plausible addition; as an inviolable rule that addition is rejected and named.

Nothing else. "Every fact names its source" and "every answer carries a
derivation" are PRD-level rules for every slice, not slice rules.

### What the slice exercises

- Supported: "which classes does `dal-makhani` contain?" → milk, via R1 and two facts. "May `plain-rice` be served to Ravi?" → yes; the derivation shows R5 and the R4 chain, so a reader can check I2 by hand.
- False vs unknown: "is `prawn-curry` free of milk?" → yes via R2, R3 (coconut milk has a sourced "none" fact). "Does `masala-dal` contain peanuts?" → **unknown**: neither R1 nor R3 fires because `house spice blend` is unclassified.
- I1: "May `peanut-chutney` be served to Ravi?" → the only derivation (R5) violates I1; the failure names I1.
- I2: "May `plain-rice` be served to Meena?" → fails naming I2 (no form). "May `masala-dal` be served to Ravi?" → fails naming I2 (not free of peanuts). "May `masala-dal` be served to Tom?" → yes: Tom declared none, so R4 holds vacuously.
- Contradiction on entry: a second form B-104 "Ravi declared none"; a second current recipe card for `peanut-chutney`; "coconut milk is in class milk" beside its "none" fact. Each is flagged, not accepted.
- Correction: replace `peanut-chutney`'s current card with one listing roasted chana (sourced "none") instead of peanuts. Ravi's chutney query flips to yes; every other answer is unchanged.
- Override attempts, rejected at entry naming the rule: the rule "an ingredient with no class fact is in none of the 14" (I2); the fact "Ravi waived peanuts for tonight" plus a rule "a waived dish is suitable" (I1).

## Consequences

- #7 loads I1 and I2 from their own file, verbatim from this ADR, separate from R1–R5. #5, #6 and #8 use this slice as their example data; no other data is invented for Phase 1.
- Changing I1 or I2, or adding a third rule, needs a `decision` issue naming the rule and an ADR superseding this one. The owner decides by merging.
- Harder: the slice needs "for every class the guest declared" (R4) and "no listed ingredient" (R3). ADR-0002 must say how universal and negative conclusions are derived and shown in a derivation without collapsing into a closed-world default that I2 forbids.
- Phase 2's second slice should be a different shape (option B is the natural candidate) to prove the modules were not special-cased to this one. The US FASTER Act list (nine allergens; it lacks celery, mustard, lupin, sulphites and molluscs, and names wheat rather than all gluten cereals) is a ready cross-source scenario for source vetting.
