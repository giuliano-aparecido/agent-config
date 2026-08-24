# D:\projects — cross-repo notes

This directory holds several independent repos (each with its own `.git`,
not a monorepo). This file is the tool-agnostic source of truth for
fleet-wide conventions; each repo's own `CLAUDE.md` is a one-line
`@AGENTS.md` pointer so Claude Code's hierarchical-load mechanism still
picks this up automatically no matter which repo under here you're
actually working in. Other assistants don't walk up parent directories the
way Claude Code does, so if you're configuring one against a single repo
under here, point it at this file directly (`../AGENTS.md` from the repo).
A repo's own `AGENTS.md`/`CONTRIBUTING.md` can narrow or override anything
below by saying so explicitly; the default in the absence of that is what's
written here.

## The repos

- **financial-sentiment-web** + **financial-sentiment-api** — a portfolio/
  curriculum project. Next.js frontend proxies to a FastAPI backend
  (Render) that calls a fine-tuned HF model for financial-news sentiment.
  Explicitly not meant to scale; hardened for the fun of it, not because
  it needs to be. Direct pushes to `main` have been the accepted workflow
  here so far — no CONTRIBUTING.md says otherwise (yet).
- **portfolio-manager-frontend** + **portfolio-manager-backend** — a real
  multi-currency investment portfolio tracker (FIFO cost basis, live
  Yahoo Finance pricing). Next.js frontend, calls a separate FastAPI +
  Postgres (Neon) backend. Each has its own `README.md`/`PROJECT.md`/
  `DEVELOPMENT.md`/`CONTRIBUTING.md` — read those for the real depth on
  this pair. **Branch + PR is required here** (see below) — this repo's
  own `CONTRIBUTING.md` says so explicitly.
- **MyPortfolio** — a separate, self-contained investment-tracker app
  (Next.js + Prisma, its own Postgres/SQLite, no separate backend
  service). Has its own full `AGENTS.md`. Its relationship to
  `portfolio-manager-frontend`/`-backend` (earlier attempt? parallel
  experiment?) hasn't been established — worth asking the owner if it
  matters for a given task, rather than assuming.
- **financial-sentiment-model** — dataset generation, LoRA fine-tuning, and
  evaluation scripts (Colab-only, no local training path) for the model
  `financial-sentiment-api` calls. No production app of its own.

## Git workflow

**Never commit or push directly to `main`/`master`.** Cut a feature
branch, commit there, and open a PR instead.

This was independently documented as house style in three of the repos
above (`portfolio-manager-frontend`, `portfolio-manager-backend`,
`MyPortfolio`) before being generalized to the whole fleet here on
2026-08-05 — treat it as the default for every repo under `D:\projects`,
including `financial-sentiment-web`/`-api` which don't have a
CONTRIBUTING.md of their own yet, unless a repo's own docs explicitly say
direct pushes are fine.

- Branch naming: `feature/`, `fix/`, `docs/`, `refactor/`, `test/` prefixes.
- Before opening the PR, run whatever build/test/lint gate the repo
  documents (check its `CONTRIBUTING.md`/`README.md` — commands differ per
  repo/stack).
- A PR description should explain *why*, not just *what* — the diff
  already shows what changed.

## Production data safety

Treat any production database URL, API key, or credential the same way
regardless of which repo it's in: never point a migration, seed, or
destructive script at a production `DATABASE_URL` (or equivalent) without
confirming a recent backup exists, and prefer a dry run first when the
tooling supports one. Never let a real credential end up in a permissions
allowlist, shell history that gets persisted, or any file that isn't
already gitignored for secrets — this has happened before via an inline
env var on an approved command (e.g. `DATABASE_URL_PROD="postgresql://...
real-password..." some-command`) getting stored verbatim in
`.claude/settings.local.json`.

## Assistant-agnostic structure

Every repo's real instructions live in its own `AGENTS.md` (the emerging
cross-tool convention — read natively by Cursor, Copilot, Codex, Aider,
and others). `CLAUDE.md` in each repo is just `@AGENTS.md`, Claude Code's
file-import syntax, so Claude picks up the same content without a
duplicate copy to keep in sync. This root `CLAUDE.md` is the same
one-liner, pointing here.

This doesn't extend to tool-specific runtime config — `.claude/settings*.json`
(permissions, hooks) has no cross-tool equivalent and isn't meant to;
other assistants keep their own config files (`.cursor/rules`,
`.github/copilot-instructions.md`, etc.) alongside this shared layer, not
instead of it.
