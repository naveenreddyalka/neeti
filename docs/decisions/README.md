# Decisions

Architecture decision records for the choices the PRD left open. One file per
decision, numbered, never edited after merge; a reversal is a new ADR that
supersedes the old one.

A `decision` issue produces one of these. An agent drafts it; a human merges
it. See [AGENTS.md § Working a decision](../../AGENTS.md#working-a-decision).

## Format

```markdown
# ADR-NNNN: <title>

**Status:** proposed | accepted | superseded by ADR-MMMM
**Issue:** #N
**Date:** YYYY-MM-DD

## Context

What the PRD says, what is unknown, what constraints apply.

## Options

### A. <name>
How it works. What it costs. What it rules out.

### B. <name>
...

## Recommendation

One option, and why it beats the others for Neeti specifically.

## Consequences

What becomes possible, what becomes harder, what the next issues are.
```

Keep it under 150 lines. Link sources. Do not pick an option because it is
familiar; pick it because of the constraints in Context. For Neeti the fixed
constraints are: a derivation a person can follow by hand, "unknown" as a
first-class answer, and constraints that no later fact can override.
