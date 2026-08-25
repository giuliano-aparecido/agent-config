# agent-config

Fleet-wide, tool-agnostic conventions for AI coding assistants, meant to
be shared across multiple independent repos as a git submodule rather
than copy-pasted into each one by hand.

See [`AGENTS.md`](AGENTS.md) for the actual conventions — this file is
just about how to wire the submodule into a consuming repo.

## Adding this to a repo

```bash
git submodule add <this-repo-url> agent-config
```

In that repo's own `AGENTS.md`, link to `agent-config/AGENTS.md` for
humans and other tools, and also import it with `@agent-config/AGENTS.md`
(Claude Code's file-import syntax) so Claude loads it automatically. That
combination is what makes it load automatically for Claude specifically:
the repo's `CLAUDE.md` should be the one-line `@AGENTS.md` pointer, whose
target in turn imports this file.

To pick up an update after editing something here:

```bash
git submodule update --remote agent-config   # inside the consuming repo
git add agent-config
git commit -m "Bump agent-config"
```

## The `code-reviewer` subagent

Claude Code only discovers subagents from `.claude/agents/*.md` inside
the repo it's actually running in, and subagent definition files can't
`@import` another file's content — they must be fully self-contained.
So a consuming repo needs its own thin stub at
`.claude/agents/code-reviewer.md`: the same frontmatter as
[`agents/code-reviewer.md`](agents/code-reviewer.md) (needed for
discovery), but a body that just tells the agent to read that file at
runtime and follow it. See `AGENTS.md`'s "Assistant-agnostic structure"
section for why a plain copy, not a symlink.
