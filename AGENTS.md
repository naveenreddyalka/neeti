# AGENTS.md — how agents work in this repo

Contract for any AI agent doing development in `neeti`.

## Source of truth

[docs/PRD.md](docs/PRD.md) is the product. Read it before changing the repo.

This repository is the project definition. Do not add a reasoner, a solver, a knowledge formalism, a language model, or a new dependency until an issue asks for it. When implementation starts, keep each change scoped to one issue.

## Commits

- Author every commit as `Naveen Reddy Alka <naveenreddyalka@gmail.com>`.
- Conventional commits: `feat(scope): ...`, `fix(scope): ...`, `docs: ...`, `chore: ...`.

## Hard limits

Never do these autonomously:

- Git tags, GitHub releases, package publishes
- Force-push, history rewrites, branch protection changes
- New runtime dependencies not named in an issue
- Committing secrets, `.env` files, or credentials
- Adding, removing, or changing an inviolable rule without an issue that names it
