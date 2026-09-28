# AGENTS.md — how agents work in this repo

Contract for any AI agent doing development in `neeti`, including the Cursor
Project coordinator and the agents it delegates to.

## Source of truth

[docs/PRD.md](docs/PRD.md) is the product. Read it before changing the repo.
[docs/ROADMAP.md](docs/ROADMAP.md) says what order the work happens in.

Do not add a reasoner, a solver, a knowledge formalism, a language model, or a
new dependency until an issue asks for it. Do not decide anything the PRD lists
under "Decisions not made yet" inside a feature PR; that needs a `decision`
issue and an ADR first. Keep each change scoped to one issue.

Knowledge is the source of truth. Every answer carries a derivation a person
can follow by hand. "Unknown" is a valid answer; guessing is not. Inviolable
rules cannot be overridden by any fact, rule, or model output.

## Two kinds of issue

| Label | What it is | Who merges |
|-------|------------|------------|
| `decision` | A choice the PRD left open. The deliverable is an ADR in `docs/decisions/` plus the matching edit to the PRD's "Decisions already made" list. | A human. Open the PR, do **not** enable auto-merge. |
| `auto-dev` | Agent-workable behavior with acceptance criteria. The deliverable is tests plus the smallest implementation that passes them. | The loop. Open the PR and enable auto-merge. |

An `auto-dev` issue that turns out to need an open decision is `blocked`: file
the `decision` issue, link it, add the `blocked` label, remove `agent:working`,
and pick the next issue.

## Working an issue

1. Pick the highest-priority open `auto-dev` issue (P1 > P2 > P3, then lowest
   number) that has no `agent:working` label, no `blocked` label, and no open
   linked PR. Add `agent:working` when you start.
2. Branch from latest `main`: `autodev/<issue-number>-<slug>`.
3. TDD: write the failing test first, then the minimal implementation. The
   acceptance criteria in the issue are the definition of done. Test what a
   person asking or a person reading can observe, not module internals
   ([PRD § Testing Decisions](docs/PRD.md#testing-decisions)).
4. Keep the change scoped to the issue. No drive-by refactors, no new
   dependencies unless the issue names them.
5. Add a status row to [docs/TRACKING.md](docs/TRACKING.md) above
   `## How to update`.
6. Run the test suite named in `docs/decisions/0001-*.md`. Fix what you broke.
7. Commit, push, open a PR with `Closes #<issue>`, a summary, and the test
   output tail. Enable auto-merge: `gh pr merge --auto --squash`.
8. Remove `agent:working`. Pick the next issue. Do not ask whether to continue.

## Working a decision

1. Same claim and branch steps. Branch: `adr/<issue-number>-<slug>`.
2. Write `docs/decisions/NNNN-<slug>.md` using the format in
   [docs/decisions/README.md](docs/decisions/README.md): context, options with
   real trade-offs, a recommendation, consequences.
3. In the same PR, move the item from "Decisions not made yet" to "Decisions
   already made" in `docs/PRD.md`, worded as the recommendation.
4. Open the PR without auto-merge. Comment on the issue that it is ready for a
   human. Remove `agent:working`. Move on.
5. If the human merges as-is, the decision stands. If they comment with a
   different choice, update the ADR and PRD in the same PR.

## Refilling the backlog

When there is no eligible `auto-dev` issue, file 3–5 new ones from the PRD:
user stories without a covering issue, Testing Decisions without a test,
or the next step in `docs/ROADMAP.md`. Each issue has context, acceptance
criteria an agent can verify, and the files it is likely to touch. Never file
an issue that quietly makes an open decision; file a `decision` issue instead.

## Commits and PRs

- Author every commit as `Naveen Reddy Alka <naveenreddyalka@gmail.com>`.
- Conventional commits: `feat(scope): ...`, `fix(scope): ...`, `docs: ...`,
  `chore: ...`. Scopes are the four modules: `store`, `reasoner`,
  `constraints`, `edge`, plus `adr` and `docs`.
- One PR per issue. Body includes `Closes #<issue>`.
- Before opening a PR, merge `origin/main` if the branch is behind. If
  `docs/TRACKING.md` conflicts, keep both new sections.

## Hard limits

Never do these autonomously:

- Git tags, GitHub releases, package publishes
- Force-push, history rewrites, branch protection changes
- New runtime dependencies not named in an issue
- Merging a `decision` PR
- Committing secrets, `.env` files, or credentials
- Adding, removing, or changing an inviolable rule without an issue that names it
- Returning an answer without a derivation, or letting model output enter the store unchecked
