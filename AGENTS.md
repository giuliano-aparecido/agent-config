# Fleet-wide conventions

A repo's own `AGENTS.md`/`CONTRIBUTING.md` can narrow or override
anything below by saying so explicitly; the default in the absence of
that is what's written here.

## Git workflow

**Never commit or push directly to `main`/`master`.** Cut a feature
branch, commit there, and open a PR instead, unless a repo's own docs
explicitly say direct pushes are fine.

- Branch naming: `feature/`, `fix/`, `docs/`, `refactor/`, `test/` prefixes.
- A PR description should explain *why*, not just *what* — the diff
  already shows what changed.

### Before every commit/push

The main agent (not every subagent it spawns — a subagent doing
exploratory or intermediate work doesn't need this) always does both of
these before running `git commit`, and therefore before any `git push`,
in a repo that includes this submodule:

1. **Run the repo's tests.** Whatever it documents
   (`CONTRIBUTING.md`/`README.md`/`DEVELOPMENT.md` — `pytest`, `npm run
   test`, etc.). A failing suite blocks the commit — fix it, or stop and
   explain why to the user rather than committing around it.
2. **Invoke the `code-reviewer` subagent** (see "Assistant-agnostic
   structure" below for where it lives) against the actual diff about to
   be committed, and address its blocking findings first. Nitpicks are a
   judgment call; correctness/security findings are not optional to skip.

Do this once per commit, not once for a whole multi-commit branch — each
commit that lands should individually have been tested and reviewed, not
just the branch's final state.

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
duplicate copy to keep in sync. A repo that includes this submodule
should link to `agent-config/AGENTS.md` from its own `AGENTS.md` for
humans and other tools, and also import it with `@agent-config/AGENTS.md`
so Claude loads it automatically (see `README.md` for the exact wiring).

This doesn't extend to tool-specific runtime config — `.claude/settings*.json`
(permissions, hooks) has no cross-tool equivalent and isn't meant to;
other assistants keep their own config files (`.cursor/rules`,
`.github/copilot-instructions.md`, etc.) alongside this shared layer, not
instead of it. `.claude/agents/` is the one exception worth a shared
convention: `agents/code-reviewer.md` in this submodule is the canonical
entry point for a review subagent used across every consuming repo.
Claude Code discovers subagents only from `.claude/agents/*.md` inside
the repo it's running in, and subagent definition files must be fully
self-contained — no `@import`/external-file-inclusion mechanism exists
for them (unlike `CLAUDE.md`/`AGENTS.md`). So each consuming repo's own
`.claude/agents/code-reviewer.md` is a thin stub: the same frontmatter
(needed for discovery) but a body that just instructs the agent to read
`agent-config/agents/code-reviewer.md` at runtime and follow it — never
a full duplicate. That means editing the source here takes effect
everywhere immediately, with no re-sync step and no drift to catch.
A plain copy of the *stub itself* (not the content) still needs
recreating in a repo only if the stub's own wording changes, which
should be rare. Symlinking the stub instead was considered and rejected:
symlink support is inconsistent enough across platforms and git configs
(`core.symlinks` isn't always on) that even the stub is better off as a
real, if tiny, file.

Within `agents/code-reviewer.md` itself the same split applies once more:
it holds only the workspace-specific scoping (which repo you're in, load
that repo's conventions, read for intent) and a verification pass, then
defers the review checklist, severity levels, and output format to
`skills/code-review/SKILL.md`, which it reads at runtime. Change *what* a
review checks by editing that skill; change *how* a review is scoped in
this multi-repo workspace by editing the agent file. The skill is also
directly invokable on its own — it does not depend on the agent file.
