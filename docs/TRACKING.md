# Tracking

Status log for merged work. One `###` section per issue, newest at the bottom
of the list, above `## How to update`. The loop appends here; humans read it.

## Phase 0 — Decide

_No ADRs merged yet._

## Phase 1 — Thin slice

### docs/RULES.md: the reasoning rules in the open ([#10](https://github.com/naveenreddyalka/neeti/issues/10))

<!-- tracking:#10 -->

**Status:** merged 2026-09-28. Added [docs/RULES.md](RULES.md), which states
the five reasoning rules in plain language with a link to the PRD line each
comes from, and a README link to it. Docs only: no test file. Each rule names
the issue whose tests will enforce it (#5, #6, #7, #9); those links get filled
in as the modules land.

<!-- tracking-append: add the next ### section above ## How to update; on conflict keep both -->

## How to update

1. Add a `### <title> ([#N](https://github.com/naveenreddyalka/neeti/issues/N))`
   section with a `<!-- tracking:#N -->` marker and a **Status** line: what
   merged, the date, and the test file that covers it.
2. Do not mark anything done without a merged PR.
3. If two PRs conflict here, keep both sections.
