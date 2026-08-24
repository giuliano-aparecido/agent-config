# Fleet-wide conventions (`agent-config`)

This file is the tool-agnostic source of truth for conventions shared
across a set of independent repos (each with its own `.git` — not a
monorepo) that all live side by side under one developer's workspace.
It's distributed as the `agent-config` git submodule, checked out at
`agent-config/` inside each consuming repo, rather than duplicated into
every repo by hand — edit it here, then bump the submodule pointer in
each repo that needs the update (`git submodule update --remote
agent-config` inside that repo, then commit the pointer change).

**How a repo wires this in:** its own `AGENTS.md` links to
`agent-config/AGENTS.md` for humans and other tools, and also imports it
with `@agent-config/AGENTS.md` (Claude Code's file-import syntax) so
Claude loads it automatically — no separate action needed to "check the
fleet docs." That combination is what makes this file load: the repo's
`CLAUDE.md` is `@AGENTS.md`, whose target imports this file in turn
(nested imports are supported up to a few hops deep). A repo's own
`AGENTS.md`/`CONTRIBUTING.md` can narrow or override anything below by
saying so explicitly; the default in the absence of that is what's
written here.

If you're configuring a non-Claude tool against a single repo that has
this submodule, point it at `agent-config/AGENTS.md` inside that repo
directly — these tools don't follow Claude's `@import` syntax, so the
repo's own `AGENTS.md` gives them a plain link instead.

## The repos

Repos that currently include this submodule:

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

Repos in the same workspace that do **not** (yet) include this submodule —
don't assume its conventions apply there without checking their own docs
first:

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

This was independently documented as house style in three repos
(`portfolio-manager-frontend`, `portfolio-manager-backend`, `MyPortfolio`)
before being generalized to the whole fleet here on 2026-08-05 — treat it
as the default for every repo that includes this submodule, including
`financial-sentiment-web`/`-api` which don't have a `CONTRIBUTING.md` of
their own yet, unless a repo's own docs explicitly say direct pushes are
fine.

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
duplicate copy to keep in sync — and that chain is also how this file
reaches Claude, per "How a repo wires this in" above.

This doesn't extend to tool-specific runtime config — `.claude/settings*.json`
(permissions, hooks) has no cross-tool equivalent and isn't meant to;
other assistants keep their own config files (`.cursor/rules`,
`.github/copilot-instructions.md`, etc.) alongside this shared layer, not
instead of it. `.claude/agents/` is the one exception worth a shared
convention: see `agents/code-reviewer.md` in this submodule for a review
agent meant to be copied into every consuming repo's `.claude/agents/`
(not symlinked — this workspace's git config has `core.symlinks=false`,
and Windows/cross-platform symlink support is unreliable enough that a
plain copy, re-synced by hand after edits here, is the more robust
choice). Keep it in sync after editing the source here.
