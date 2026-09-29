# ADR-0004: Which language model the approach experiments call, and how a model call is recorded

**Status:** proposed
**Issue:** #18
**Date:** 2026-09-29

## Context

Neeti's approach is an open research question: a symbolic solver, a language
model, or a model combined with a solver, under one fixed requirement — every
answer names the exact facts and rules that produced it, and nothing a model
produces enters the store or an answer unchecked (PRD § Solution, story 15;
[RULES.md Rule 1](../RULES.md)). ADR-0002 (#3) names a symbolic first
candidate and lists the others: an LLM as translator with a symbolic verifier,
and an LLM+ASP hybrid. Neither can be run on the ADR-0003 slice without
calling a model, and AGENTS.md forbids an experiment issue from choosing one.

**Scope.** This ADR decides the model the *experiments* call and how a call is
recorded. It is **not** the Phase 2 decision "which language model, if any,
sits at the edge, and how its output is checked" ([ROADMAP § Phase
2](../ROADMAP.md)); that item stays open in the PRD, and this ADR produces the
evidence it will be decided on.

Constraints:

1. **Replayable without the model.** A person must be able to re-check a run
   after the model is gone, changed, or unaffordable: hosted versions retire,
   local runtimes drift, Telivi does not exist yet. The record is the object
   of study, not the model.
2. **Checkable against Rule 1.** The record must make it mechanical to show
   that nothing in an answer came from the model rather than the store.
3. **Fair to the research question.** The model Neeti would ship with is
   Telivi, a member-owned model that starts small. Evidence from a frontier
   model alone would overstate what the approach can do.
4. **No training or fine-tuning** (PRD § Out of Scope). **No runtime
   dependency** without an issue naming it (AGENTS.md § Hard limits).
   **No secrets** in anything committed.

## Options

### A. Stand-in only: no real model in Phase 1 experiments

The stand-in of #9 — a fake that returns a known translation — is the only
model until Phase 2. *Reproducible:* perfectly. *Cost and dependency:* none.
*Rules out:* learning anything about the LLM and LLM+ASP candidates. A
stand-in measures the checker, not the candidate; two of the three rows of
the experiment protocol (#19) stay empty, and Phase 2 decides with no evidence.

### B. A hosted API model

A frontier model over HTTPS (OpenAI, Anthropic, Google, or similar).
*Capability:* highest; the best upper bound. *Reproducible:* only through
recorded fixtures — `temperature 0` is not deterministic, and a version can
be retired mid-project, so a live re-run is a new run, never a re-check.
*Cost:* per call, with the owner's key in the environment. *Dependency:* an
HTTP client or SDK, plus network, for the experiment only. *Rules out:*
offline replay of anything not recorded; people-owned evidence.

### C. A small local open-weights model

An instruct-tuned model of a few billion parameters, weights pinned by
content digest, run offline with a fixed seed. *Capability:* lower than B; it
may fail translations B passes, which is itself a finding about the size
Telivi will start at ([solver-in-the-loop
tuning](https://arxiv.org/abs/2512.17093) shows small open models can emit
ASP). *Reproducible:* weights can be re-fetched and re-run indefinitely;
outputs may still differ across hardware, so fixtures remain the record.
*Cost:* none per call; a multi-GB download and a machine. *Dependency:* a
runtime binary and the weights, for the experiment only. *Rules out:*
nothing; it is the closest stand-in for Telivi that exists.

### D. Telivi's model, once it exists

The sibling project the PRD names: the model people own. *Capability:*
unknown; data collection is still an open plan item. *Reproducible:* as C, if
its weights are versioned. *Rules out:* any LLM experiment until it ships,
which blocks Neeti's research question on another project.

### How a call is recorded

1. **Free-form logs.** Whatever the run prints. Cheap; not replayable; not
   checkable, because prompt and response are not tied to the check.
2. **A fixture per call, and a replay mode.** Every call is written verbatim
   to a file before its response is used; replay serves responses from those
   files and never calls a model. Costs a small harness and disk.

## Recommendation

**Option C for the model; recording as fixture-per-call with replay.** The
stand-in (A) runs first in every candidate to prove the harness, checker, and
recording work, but is not evidence about a model. A hosted model (B) may be
run as an additional, labelled run of the same protocol for an upper bound,
at the owner's cost; it is never the only evidence. Telivi (D) joins as
another model under the same record when it is callable; no new ADR needed.
The experiment issue that first runs an LLM candidate names and pins the
exact model id, weight digest, and runtime version; that issue is the one
that "names the dependency", and the dependency is experiment-only, never in
the package's runtime set.

C beats A because A answers nothing; B alone because B's evidence cannot be
reproduced live and overstates what a Telivi-sized model can do; D alone
because D does not exist. Recording makes the model choice low-stakes: the
fixtures are the result, and any model can be swapped in behind them.

**The record of a call.** One JSON file per call, named in call order:
`docs/experiments/runs/<run-id>/calls/NNNN.json`, beside the run's record
file from #19 (`docs/experiments/runs/<run-id>.md`). JSON, because replay
matches the prompt byte for byte and needs no dependency to read. Fields:

- `model`: provider, model id, version or weight digest, runtime and version;
  `parameters`: temperature, seed, max tokens, anything else sent;
  `timestamp` (UTC); `purpose`: the protocol step the call serves (e.g.
  "translate question Q3", "propose rules for I2 scenario").
- `prompt`: every message sent, verbatim, in order, with roles; no headers,
  no credentials. `prompt_sha256` over the canonical JSON of `prompt` plus
  `parameters` is the replay key.
- `response`: the text returned, verbatim, with finish reason and any tool
  calls; `raw` may hold the provider's full body minus authentication.
- `check`: what the candidate did with the response before using it —
  `accepted` or `rejected`; every store id the response referenced; the ids
  **not** in the ADR-0003 register; the inviolable rule named if a proposal
  was refused; if accepted, the query or derivation lines that entered the run.

**Replay.** Two modes. `live` calls the model and writes the fixture before
returning the response. `replay` looks the prompt hash up in the run's
`calls/` and returns the stored response; a miss is a hard failure, never a
live call. CI runs `replay` only: no network, no key. A re-check is a replay
of the fixtures; a live re-run is a new run.

**Why this makes Rule 1 checkable.** A model response enters a run only
through a `check` block, so the model-originated material is exactly the
`accepted` entries. Two lookups confirm Rule 1: every id in the derivation is
in the register (RULES.md, checking step 2), and every accepted output is a
query or a store-checked line, never a fact or rule that reached the
derivation without a register id. Either failing is a Rule 1 violation.

## Consequences

- The LLM and LLM+ASP rows of the experiment protocol (#19) can be filled;
  its "link to the verbatim prompt/response log" is the `calls/` directory.
  #9 is unaffected: its stand-in and `replay` are the same thing, a model
  that returns a recorded translation, so #9 can be the harness's first user.
- The Phase 2 edge-model decision stays open and gains its evidence: how often
  the checker rejected each model's output, and what the rejections were.
- Harder: a local model needs hardware and a large download, and results may
  differ across machines, so only fixtures are authoritative. Hosted runs need
  the owner's key and cost money. Fixtures are secret-scanned before commit.
- Next issues: the experiment issue that first runs an LLM candidate (naming
  model, digest, runtime, and experiment-only dependency); the replay harness.
