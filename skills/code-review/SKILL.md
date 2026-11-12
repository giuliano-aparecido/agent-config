---
name: code-review
description: Use this skill to review a diff, pull request, or file set. Applies the clean-code skill's rules plus correctness, security, and architecture checks, and reports categorized findings with severity levels. Read-only.
---

# Code Review

Read `skills/clean-code/SKILL.md` (`agent-config/skills/clean-code/SKILL.md`
from a consuming repo) first. It is the rule set for style, naming,
functions, comments, classes, errors, and tests. This skill adds what only a
review needs: process, the checks that are not Clean Code (correctness,
security, architecture), severity, and output format. The review is
read-only: the findings report is the whole deliverable.

## Process

1. **Scope.** Identify changed, added, and deleted files. Read the surrounding
   unchanged code too; a function can look fine alone and still violate SRP in
   context. For an explicit full-codebase review (no diff, PR, or commit range
   named), every tracked source file is in scope except build artifacts,
   lockfiles, vendored or generated code, and binary/data files. Roll up a
   smell repeated across many files into one systemic finding with
   representative `file:line` citations. An empty `git diff` right after a
   commit is not this mode; look at `git log -p` for the commit range.
2. **Run the sections below in order**: correctness, security, architecture,
   then each section of the clean-code skill. Do not lead with style while a bug or
   vulnerability goes unmentioned.
3. **Deduplicate.** One root cause is one finding.
4. **Assign severity**, then **output** as specified at the end. Every section
   is accounted for: findings in the list, empty sections in the coverage line.

## Correctness and reliability (check first)
- Does the code do what the PR description or ticket says?
- Edge cases: empty, null, zero, maximum values, concurrent access.
- Off-by-one, wrong boundary conditions, floating-point `==`, implicit coercion.
- Resource leaks (files, connections, cursors); missing `await`, unhandled
  rejections, fire-and-forget.
- Race conditions, non-atomic check-then-act, shared mutable state.
- Infinite loops or recursion without a guaranteed end.
- Null dereference or unchecked optional access on data crossing a trust
  boundary; type-coercion bugs that change behavior.
- Empty catch blocks and catching overly broad types (`Exception`/`Throwable`).
- External calls (network, DB, filesystem) without timeouts or failure
  handling, or with unbounded retry loops.
- Non-exhaustive dispatch: an `if`/`elif`/`switch`/`match` over a closed set
  of types with no final `else`/`default` that raises. Flag it even at two
  branches; a new variant must fail loudly, not be silently dropped.
- Do existing tests still make sense; are there tests for new behavior?

## Security (every hit is at least Major; a suspected vulnerability is a Blocker until disproven)
- Secrets, tokens, or credentials in source or logs; sensitive data logged.
- Injection (SQL, command, path traversal, template/HTML), unvalidated input
  reaching queries, commands, or file paths.
- Weak crypto (MD5/SHA1 for security, ECB, weak RNG), insecure deserialization.
- Missing authentication or authorization on sensitive operations (BOLA).
- Permissive CORS, missing security headers, `eval` or unsafe reflection.
- New dependency that is untrusted, unpinned, or has known CVEs.

## Architecture and boundaries
- Layering: no lower layer calling a higher one; the domain does not import
  infrastructure. Dependencies point toward abstractions and the core domain.
- No reaching into another module's internals that are not its public API.
- Consistency: a new pattern for something the codebase already solves is Major.
- Blast radius: for shared code, check who else calls it and whether the change
  is backward compatible.
- Destructive or irreversible operations (deletions, migrations, broad
  permission changes, pattern-matched cleanup) are Blocker unless scoped,
  guarded, and reversible, or behind an explicit confirmation or dry run.

## Severity

| Level | Meaning | Examples |
|---|---|---|
| **Blocker** | Must fix before merge | Security issue, broken logic, data loss risk, breaks a public API contract, unguarded irreversible operation |
| **Major** | Should fix before merge | SRP violation, missing error handling, untested critical path, boundary violation, any comment forbidden by clean-code section 3 (narrating, redundant, too much information, non-local, unit or worked-example, bundled, repeated, commented-out code, a misleading comment, a private docstring that restates the signature), a name that needs a comment, a flag argument, non-exhaustive dispatch over a closed set, removing a suppression directive (`# noqa`, `eslint-disable`, `@ts-ignore`; it re-arms a suppressed error), deleting a comment that carried intent or a warning of consequences |
| **Minor** | Should fix, non-blocking | Long function, duplication, 4+ arguments, stale comment, magic number |
| **Nit** | Optional | Formatting, consistency, personal preference |

A stale comment (outdated but not wrong in a way that misleads) is Minor. A
new public interface with no docs, and code hiding a non-obvious intent or
dangerous consequence with no comment (clean-code section 3), are Minor. Any
other clean-code item not named in the table also defaults to Minor. Only
correctness, security, and architecture reach Blocker. "Style" here means
formatting only; naming and comment findings are not style. Formatting is never
Blocker or Major and belongs to the linter. Raise
a style finding only when the repo's lint/format config is actually violated;
otherwise it is a Nit at most. A genuinely subjective finding is marked Nit and
says so.

## Output

For each finding:

```
[SEVERITY] file/path.ext:line — short title
Why it matters: <1-2 sentences, concrete impact, not just the rule name>
Suggestion: <concrete fix or direction>
```

Then:

```
Strengths: <1-2 genuinely good things; omit if none>
Checked, no findings: <section names that produced nothing>
Summary: X blockers, Y major, Z minor, W nits. Recommendation: [approve / approve with comments / request changes]
```

Recommendation: any Blocker means request changes; Major without Blocker means
approve with comments, or request changes if Majors are many relative to the diff;
only Minor/Nit means approve with comments.

## Guardrails

- **Read-only.** Never stage, commit, push, merge, or change any file or git
  state, and never apply your own suggested fix, even a trivial one and even if
  the conversation's pattern was "review, fix, commit". Running tests or a
  linter to observe behavior is fine.
- Do not invent findings; "no issues" is a valid result for a section.
- Scale strictness to purpose (a one-off script is not payment code), but
  correctness and security are always flagged.
- If context is insufficient to judge (cannot tell if a boundary violation is
  intentional), say so and ask.
- A diff whose purpose is comment cleanup gets no lighter pass: check every
  removed comment, and every removed suppression directive is a finding.
- Reviewing the agent's own irreversible actions: default to Blocker until a
  human confirms.
