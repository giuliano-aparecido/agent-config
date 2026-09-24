---
name: code-review
description: Use this skill to perform a structured code review of a diff, pull request, or file set. Checks correctness and security first, then Clean Code principles, SOLID, and architectural boundaries. Produces categorized, actionable findings with severity levels rather than general commentary.
---

# Code Review Skill

## Purpose
Perform a systematic, reproducible code review of the given diff, PR, or file set. Output must be
structured findings (not prose essays) so it can be posted as PR comments or
aggregated into a report. Every finding must reference a specific file/line and
explain *why it matters*, not just *what rule it breaks*.

## Review Process

1. **Scope the review** — identify changed files, new files, deleted files. Review
   changed/added code in full context (read surrounding unchanged code too — a
   function can look fine in isolation and still violate SRP in context). When
   the review is explicitly a full-codebase/no-PR review (the request names no
   diff, PR, or commit range), treat every tracked source file as in scope
   instead — skip build artifacts, lockfiles, vendored/generated code, and
   binary/data files — and expect several checklist sections to legitimately
   produce no findings on a mature codebase; name those in the coverage line
   rather than forcing a finding. An empty `git diff` on its own is not this
   mode — right after a commit the work is in `git log -p` or the commit
   range, not the working tree. Prioritize Correctness and Security in the
   output even more heavily than usual, and roll up a smell repeated across
   many files into one systemic finding with representative file:line
   citations rather than one finding per occurrence.
2. **Run each checklist section below** against the in-scope code, in the order given
   (Correctness → Security → then the rest) — don't lead with style while a bug
   or vulnerability goes unmentioned.
3. **Deduplicate** — one root cause should produce one finding, not five repeated
   comments on similar lines.
4. **Assign severity** to every finding (see Severity Levels).
5. **Output** using the Output Format section. Every checklist section must be
   accounted for — findings go in the flat list, and sections that produced none
   are named in the coverage line so reviewers see they were checked, not skipped.

## Severity Levels

| Level | Meaning | Examples |
|---|---|---|
| **Blocker** | Must fix before merge | Security issue, broken logic, data loss risk, breaks public API contract |
| **Major** | Should fix before merge | SRP violation, missing error handling, untested critical path, architectural boundary violation |
| **Minor** | Should fix, non-blocking | Naming, long function, duplication, missing comment on why (not what) |
| **Nit** | Optional / style | Formatting, minor consistency, personal preference |

Never mark style/formatting as Blocker or Major — those belong in Nit and should
ideally be caught by a linter/formatter, not human/agent review time.

## Checklist

### 1. Correctness & Reliability (check first — nothing else matters if this fails)
- Does the code do what the PR description / ticket says it does?
- Are edge cases handled (empty input, null, zero, max values, concurrent access)?
- Null/undefined dereference, unchecked optional access, missing null checks on
  data crossing a trust boundary.
- Off-by-one errors, incorrect loop bounds, wrong boundary conditions.
- Resource leaks: unclosed files, streams, connections, sockets, DB cursors —
  missing try-with-resources / `using` / `defer` / `finally` equivalents.
- Unhandled exceptions, empty catch blocks, catching overly broad exception types
  (`Exception`/`Throwable`).
- Incorrect equality checks — reference vs. value equality, floating-point `==`.
- **Non-exhaustive type dispatch**: an `if`/`elif isinstance(...)` chain,
  `switch`, or `match` over a closed set of known types with no final
  `else`/`default` that raises or otherwise handles the unmatched case. Flag this
  even when every current branch is covered — the bug is that a future/unexpected
  variant is silently dropped rather than erroring, and it's worth flagging at 2
  branches as much as at 10.
- Race conditions, non-atomic check-then-act, unsynchronized shared mutable state.
- Type-coercion bugs (implicit conversions that change behavior).
- Dead code, unreachable code, code after `return`/`throw`.
- Infinite loops / recursion without a guaranteed base case or termination
  condition.
- Improper use of async/await or promises (missing `await`, unhandled rejections,
  fire-and-forget).
- Do existing tests still make sense, and are new tests added for new behavior?

### 2. Security (every hit is at least Major; a suspected vuln is a Blocker until disproven)
- No secrets/credentials/API keys/tokens in source or logs.
- Injection risks: SQL/NoSQL injection, command injection, path traversal,
  unsanitized template/HTML injection (XSS).
- User input validated/sanitized before use in queries, commands, file paths.
- Weak/broken crypto: MD5/SHA1 for security, ECB mode, weak RNG for
  security-sensitive values.
- Insecure deserialization of untrusted data.
- Missing authentication/authorization checks on sensitive operations (broken
  object-level authorization / BOLA).
- Sensitive data (passwords, tokens, PII) logged in plaintext.
- CORS misconfiguration, missing security headers, permissive wildcard origins.
- Insecure use of `eval`, dynamic code execution, or unsafe reflection.
- New dependency from an untrusted source, unpinned, or with known CVEs (flag for
  a dependency audit if suspicious).

### 3. Naming
- Intention-revealing: does the name say *why*/*what*, without needing a comment?
- No noise words (`data`, `info`, `manager`, `helper`, `temp`) unless truly generic.
- No disinformation (e.g. `userList` that isn't a `List`).
- Verbs for methods, nouns for classes, consistent vocabulary for the same concept
  across the codebase (don't mix `fetch`/`get`/`retrieve` for the same operation).
- Booleans read as predicates (`isValid`, `hasPermission`), not `flag`/`status`.
- Searchable names for anything referenced more than once — no bare literals or
  single letters for values used across the file.

### 4. Function / Method Design
- Length: flag functions over ~20–30 lines as Minor, over ~50 as Major — but weigh
  against genuine cohesion, not line count alone.
- Parameters: 4+ positional params → Minor, suggest a parameter object.
- Nesting depth: more than 2–3 levels → Minor/Major depending on readability impact.
- Boolean/flag parameters that branch internal behavior → Major (function is doing
  two things; suggest splitting).
- Command-query separation: does a function that returns a value also mutate state?
- Single level of abstraction: is high-level orchestration mixed with low-level
  detail in the same function?

### 5. Class Design & SOLID
- **SRP**: Can the class's responsibility be described in one sentence without
  "and"? If not → Major. Look for classes that grew a second responsibility via a
  new method in this diff — this is the most common review catch.
- **OCP**: Does this change require modifying a stable/shared class's internals to
  support a new case, where extension (subclass, strategy, new implementation of
  an existing interface) would have worked instead? Flag `if`/`else` or
  `switch`/`match` chains that dispatch on type regardless of length — a short
  chain today still forces editing this function for every new case tomorrow.
  Favor polymorphism/strategy, or at minimum the exhaustiveness check from
  section 1 so a missed case fails loudly instead of silently.
- **LSP**: Does a subclass/override change behavior in a way that would break code
  written against the base type's contract (throwing new exceptions, narrowing
  accepted input, changing return semantics, `NotImplementedError` for an
  inherited method)?
- **ISP**: Is a new/changed interface forcing implementers to implement methods
  they don't need? Prefer several small interfaces over one broad one.
- **DIP**: Does business logic depend directly on a concrete implementation
  (a specific DB client, HTTP client, filesystem call) instead of an abstraction?
  Flag as Major if it hurts testability or couples layers that shouldn't know
  about each other.
- Law of Demeter: flag multi-dot chains reaching through objects (`a.getB().getC().getD()`).
- Data classes vs. behavior classes: flag hybrids that expose fields *and* carry
  business logic — pick one.

### 6. Architecture & Boundaries
- **Layering**: does this change let a lower layer (e.g. data access) call into a
  higher layer (e.g. UI/controller), or a domain/business layer import
  infrastructure concerns directly? → Major.
- **Dependency direction**: do dependencies point toward abstractions/core domain,
  not outward to frameworks/infrastructure? Flag violations that will make future
  swaps (DB, framework, external API) harder.
- **Module/package boundaries**: does the change reach into another module's
  internal package/class that isn't part of its public API?
- **Consistency with existing patterns**: does this diff introduce a new pattern
  for something the codebase already solves elsewhere (e.g. a new HTTP client
  wrapper when one already exists)? Flag as Major — inconsistency compounds.
- **Blast radius**: for changes to shared/widely-used code (utilities, base
  classes, shared config), explicitly check who else calls this and whether the
  change is backward compatible.
- **Destructive or irreversible operations** (deletions, migrations, broad
  permission changes): flag any such operation as Blocker unless it is scoped,
  guarded, and reversible or has an explicit confirmation/dry-run step. Do not
  approve broad automated cleanup (e.g. deleting branches/resources by pattern
  match) without an explicit allow-list or human confirmation step.

### 7. Error Handling
- Exceptions over error codes; no swallowed exceptions.
- No returning `null` from methods where an empty collection/Optional/explicit
  result type would do; no passing `null` as an argument without justification.
- Exceptions carry enough context to debug (not just "Error occurred").
- Error-handling scaffolding doesn't bury the main logic path — the happy path
  should still be readable at a glance.
- External calls (network, DB, filesystem) have failure handling — timeouts,
  retries where appropriate, no unbounded retry loops.

### 8. Duplication (DRY)
- Structural duplication (same logic copy-pasted with minor variation) → Minor/Major
  depending on size and how many call sites already exist.
- Near-identical boilerplate (same setup/teardown) repeated 3+ times across
  functions in one file → extract a shared function, context manager, or decorator.
- Don't flag superficial similarity that isn't the same *reason to change*
  (premature abstraction is also a smell — don't force a merge of unrelated logic
  just because it looks similar today).

### 9. Comments & Documentation (default: no comment unless a good name can't carry the why; public/exported APIs are the standing exception — see below)
- Comments that compensate for a bad name → suggest rename instead, flag as Major.
- Even with a good name, check whether the name already implies the rationale before
  crediting any comment as load-bearing: read the identifier (function, class,
  variable) in isolation and ask whether a competent reader would already infer the
  same conclusion from it alone, no comment needed — e.g. a function named
  `clampToMaxAttempts` with a comment saying "caps the attempt count at the maximum"
  adds nothing the name didn't already say. This applies per sentence, the same as
  narrate-what below: a sentence that only restates the name is cut; a sentence that
  adds new, name-independent content (even on the same function) is evaluated on its
  own merits by the bars below, not automatically disqualified alongside it. Flag as
  Major, same bucket as narrate-what; if a comment also fails a later bar (e.g. the
  textbook-knowledge case below) for the same underlying fact, that's one finding, not
  two.
- Comments that narrate *what* the code does rather than non-obvious *why* → Major.
  Suggest removing the failing sentence(s) outright, not just rewording them — trim
  down to only the sentence(s) that are genuinely load-bearing if any survive, or
  remove the comment entirely if none do. A multi-paragraph or multi-line comment
  block is a signal to look here, not a trigger on its own — a long comment is fine if
  every sentence earns its keep with genuine non-obvious rationale; flag length only when
  it's padding or restating the code.
- **Clearing the external-bug-workaround bar caps *what* can be said, not how much.**
  Once a comment identifies the qualifying fact, restate only the fact and its
  consequence for the code beside it — not the investigation that discovered it.
  Specific reproduction data, exact numbers from a live check, dates, and the
  step-by-step reasoning that led to the fix belong in the commit message or PR
  description, which is where a future reader looks for *how someone figured this
  out*; the inline comment is where they look for *the constraint they must not
  violate*. A comment attached to a declaration (a variable, a constant, a single
  function signature) should describe only what's non-obvious about *that*
  declaration — if most of its sentences are actually about the surrounding
  system, the module's problem space, or the general shape of the fix, that
  content is misplaced, not merely long. Test: could the qualifying fact be
  stated in 1-2 sentences without losing anything a future editor needs to avoid
  reintroducing the bug? If yes, flag the excess as Major, same bucket as
  narrate-what, and suggest cutting to that 1-2 sentence version with the rest
  moved to the commit message (or deleted if it's pure narrative). This is a
  distinct check from the length signal above: that one asks whether each
  sentence is *about* something real; this one asks whether a sentence that
  passes that test is still saying more than the code beside it needs.
- **The same qualifying fact restated across more than one comment in a diff is a
  duplication finding, not several valid comments.** When a diff explains the same
  external-bug-workaround fact near a constant, again near the function that uses
  it, and again in a test, only the location closest to where the workaround
  actually happens (typically the function/line that branches on or depends on the
  fact) should carry the full explanation — every other occurrence should be a
  one-clause pointer to it ("see `_normalize_pence_quote`'s docstring") or removed
  outright. Flag every occurrence beyond the first genuinely necessary one as
  Major, same bucket as narrate-what, and name the file:line of the location that
  should keep the explanation.
- **Consistency with the surrounding file's existing comment style is never a
  reason to skip the checks above.** A codebase that already has a habit of
  long, evidence-heavy comments is a pre-existing condition to note (optionally,
  as a separate systemic finding per the full-codebase-review rule in the Review
  Process section) — it is not a justification for a new comment in the diff
  being reviewed. Judge every new or changed comment against these bars on its
  own; "the file already does this" answers a different question (is this
  diff introducing a new problem) than the one this checklist asks (does this
  comment, here, earn its content).
- A stated *why* isn't automatically exempt, and the bar here is deliberately high:
  the ONLY category that earns an inline comment is a workaround for an external
  library/platform bug or limitation, where the reasoning has no other place to live
  (not inferable from a test, a type, a linked issue, or a commit message) — e.g. "don't
  pre-encode this before handing it to URLSearchParams — it already percent-encodes
  once, and pre-encoding double-encodes; this broke sign-out in prod." A hidden
  constraint, a business rule, a cross-file invariant, a magic-number/config rationale,
  a hotfix note, or a "concrete tradeoff" number are NOT enough on their own — see the
  explicit carve-outs below for the categories this most often shows up as. Anything
  short of the external-bug-workaround bar → Major, same as narrate-what — including a
  generic justification ("for simplicity", "because it's cleaner") or a code-shape claim
  that never says why the alternative would actually hurt *here* ("keeps call sites a
  one-liner" alone, without saying what a throw/void alternative would cost this
  codebase specifically). Don't downgrade to Minor just because the comment names *some*
  reason — a present-but-vague (or present-but-no-longer-qualifying) reason is exactly
  the failure mode this rule exists to catch. Reserve Nit for the Guardrails' general
  "genuinely subjective" case: whether a rationale clears the external-bug-workaround
  bar at all, not how weakly it clears it.
- These specific categories do NOT qualify as load-bearing on their own, even though
  earlier revisions of this skill treated each as sufficient — flag one that's present
  as Major (narrate-what bucket, per the bullet above), and treat one that's *removed*
  as a neutral cleanup, not a finding (see the Missing-comments and Comment-removal
  bullets below):
  - **Cross-file/cross-module invariants** ("a signed-in session no longer implies X
    membership since Y changed elsewhere") — accepted as tribal knowledge, or covered by
    a test that asserts the actual behavior, not spelled out inline.
  - **Magic-number/config rationale tied to an external constraint** (a platform's
    timeout ceiling, a cold-start window, a rate limit) — the value itself can get a
    named constant if it needs a name, but the "why this number" story doesn't need to
    live next to it.
  - **Rationale for a suppression/pragma directive** (`eslint-disable-next-line`,
    `@ts-ignore`, `# noqa`, etc.) — the directive itself must stay (see the separate
    bullet below on removing those), but a prose explanation of why it's safe to
    suppress is no longer required or rewarded.
  Flag a comment that only contains one of these as Major, same bucket as narrate-what.
- Two more things that don't count as non-obvious, even when the sentence stating them
  is true and specific, applying the same detectability lens the categories above rule
  out on more generally: (1) a consequence that ordinary code review, tests,
  type-checking, or just looking at the visible output would catch anyway — "silent"
  means genuinely hard to detect, not "a comment would have made this easier to
  notice." A wrong-but-visible UI string, for instance, doesn't qualify — a screenshot
  or a glance at the two components' output catches it, no comment needed. (2) expected
  knowledge for anyone who'd touch this kind of code — why you don't blindly retry a
  non-idempotent POST is textbook HTTP practice, not a fact specific to this system,
  even though a sentence stating it is perfectly concrete. Flag as Major, same bucket as
  narrate-what; if this overlaps with the external-bug-workaround bar above for the same
  underlying fact, that's one finding, not two. As with the bullet above, whether
  something clears "textbook" is a judgment call at the margin — reserve Nit for the
  Guardrails' genuinely-subjective case.
- A comment stating only *where or how* something is used ("shared by three call
  sites", "called by X and Y") isn't exempt either, even though it's neither pure
  narrate-what nor a stated rationale — for internal code within the reviewed
  codebase, that's exactly what a find-references/grep search already shows for free
  to a reader working in it, so writing it down doesn't earn a comment. Flag as Major,
  same bucket as narrate-what (remove or trim per that bullet's guidance), unless it
  also explains *why* that reach matters — using the same external-bug-workaround test
  above, not a generic gloss like "intentional for consistency", which fails that bar
  exactly as it would for a stated design rationale. This bullet doesn't apply to
  public API / exported docs read by callers who can't grep the implementation
  (external consumers, other repos/services) — there, a usage-context statement can be
  the documentation itself, not padding; see the Public API bullet below.
- Even a rationale that clears the external-bug-workaround bar must be *local* to earn
  its place: relevant to the code it's attached to, not just true somewhere in the
  system. This is a placement test, separate from whether the why itself qualifies at
  all — a comment can clear that bar and still fail here if the code beside it isn't
  the actual workaround; if both point to the same root cause on the same comment,
  that's one finding, not two. If the code doesn't branch on the fact, depend on it, or
  need a future editor to preserve it, the comment doesn't belong there no matter how
  concrete the fact is. Passing contrast: a comment on the exact line that special-cases
  a known bug in a pinned dependency version (named, with a link to the upstream issue)
  clears this bar — the workaround code is right there, and removing the comment would
  strand a future editor with no way to tell the special case isn't dead code. Failing
  contrast: a component that renders identically regardless of whether the state it's
  showing was set by a cron job or a button click doesn't need a comment explaining
  that ambiguity is "on purpose"; the component doesn't act on that distinction either
  way, so the explanation (if it belongs anywhere) belongs where the distinction is
  actually made or consumed, not repeated at every place that merely reads the
  resulting state. Flag as Major, same bucket as narrate-what.
- Commented-out code → Minor, request removal (version control preserves history).
- Stale or misleading comments that no longer match the code → Minor, worse than
  no comment.
- Missing comments where they matter: a workaround for an external library/platform bug
  or limitation, with no other place the reasoning is recorded — flag as Minor if
  absent. Nothing else triggers this bullet anymore: a missing cross-file invariant, a
  missing magic-number/config rationale, or a missing suppression-directive rationale
  (the carved-out categories above) is not a finding — those are meant to be tribal
  knowledge or inferable from tests/types, not written down.
- **Comment removal is a distinct, higher-risk case from missing comments** — the
  diff is actively deleting something, not just failing to add it, so evaluate
  every removed comment (not only ones the diff also touches the code around)
  against the external-bug-workaround bar above before accepting a "trim unnecessary
  comments" pass at face value. Escalate to **Major** (not Minor) only when the removed
  sentence documented a previously-diagnosed workaround for an external library/
  platform bug or limitation, whose reasoning has no other place to live (e.g. "don't
  pre-encode this before handing it to URLSearchParams — it already percent-encodes
  once, and pre-encoding double-encodes; this broke sign-out in prod"). A removed
  comment that instead stated a cross-file invariant, a magic-number/config rationale,
  or a suppression-directive rationale (the carved-out categories above) is NOT an
  escalation case — treat its removal as ordinary, unremarkable cleanup, not a finding.
  Where this escalation DOES apply, the risk isn't absent documentation in general —
  it's that this diff is the one reintroducing the exact failure the comment existed to
  prevent, and nothing else (tests, types, lint) will catch that regression. Restoring a
  one-line version of just the load-bearing sentence(s) resolves the finding; it does
  not justify reverting the whole comment block, and the rest of the same block can
  still be correctly-removed filler.
- Suppression/pragma directives written as comments (`eslint-disable`,
  `eslint-disable-next-line`, `@ts-ignore`, `@ts-expect-error`, `# type: ignore`,
  `# noqa`, `// NOLINT`, etc.) are not documentation and are out of scope for
  comment-quality cleanup — they change tool behavior (silently re-arming a
  lint/type error the directive was suppressing), regardless of whether the
  relevant linter/type-checker currently runs in this repo's CI. Flag removal as
  **Major**: this is a correctness/tooling regression, not a comment-style
  judgment call, and "no lint config exists yet" is not a mitigation — it only
  means the regression is currently invisible rather than absent.
- Public API / exported docs (function, class, type, or constant): flag Major if
  missing for a new public interface.

### 10. Maintainability
- Magic numbers/strings that should be named constants.
- Large classes / "god objects" doing too much.
- Dead/unused code, unused imports, unused variables.
- Inconsistent naming conventions within the same codebase.
- Non-idiomatic constructs where the language/framework offers a clearly cleaner
  one (Nit unless it also hurts correctness or readability).
- Boy Scout Rule: note whether nearby code was left cleaner or dirtier than found
  (informational, not a blocker).

### 11. Tests
- New/changed logic has corresponding tests (Major if missing on non-trivial logic).
- Tests actually assert meaningful outcomes, not just exercise code.
- Edge cases and error paths are tested, not just the happy path.
- Tests are independent, deterministic, and test one concept each.
- Test names describe behavior/spec, not implementation detail.
- No disabled/skipped tests introduced without a tracked reason (ticket link).

## Output Format

For each finding:

```
[SEVERITY] file/path.ext:line — short title
Why it matters: <1-2 sentences, concrete impact, not just rule name>
Suggestion: <concrete fix or direction, not just "fix this">
```

End with a strengths note, a coverage line, and a one-line summary:
```
Strengths: <1-2 things genuinely done well — real signal, not filler; omit if none>
Checked, no findings: <comma-separated checklist section names that produced nothing>
Summary: X blockers, Y major, Z minor, W nits. Recommendation: [approve / approve with comments / request changes]
```

Recommendation logic:
- Any Blocker → request changes
- Any Major and no Blocker → approve with comments (reviewer/team discretion) or
  request changes if Major count is high relative to diff size
- Only Minor/Nit → approve with comments

## Guardrails for the Agent

- **This skill is read-only.** The findings report is the entire
  deliverable — never stage, commit, push, merge, or otherwise mutate the
  repo (any repo) while running this skill, and never apply your own
  suggested fix. This holds even for a change that looks trivially small
  or obviously correct, and even when the surrounding task/conversation's
  pattern has been "review, then fix, then commit" — that next step
  belongs to whoever invoked this skill, not to the review itself. Running
  the test suite or a linter to observe current behavior is fine; changing
  files or git state is not.
- Do not invent issues to pad the review — "No issues found" is a valid and
  expected outcome for a section.
- Do not treat this checklist as a hard gate that blocks merges purely on line
  counts or naming style — those are Minor/Nit by default. Only correctness,
  security, and architecture violations should ever reach Blocker.
- Scale strictness to the code's purpose — a prototype or one-off script doesn't
  need the rigor of production payment code — but security and correctness bugs
  are always flagged regardless of context.
- Only raise a style/convention finding when a lint/format config actually present
  in the repo (`.eslintrc`, `.editorconfig`, `pyproject.toml`, `.prettierrc`,
  etc.) is violated — otherwise it's a Nit at most, and often not worth raising.
- When a finding is genuinely subjective (style preference not covered by team
  convention/linter config), mark it Nit and say so explicitly rather than
  asserting it as an objective flaw.
- If context is insufficient to judge a finding (e.g. can't tell if a
  boundary violation is intentional), say so and ask, rather than guessing.
- A diff whose stated purpose is comment trimming/cleanup does not get a lighter
  pass on section 9 — the opposite: every removed comment is a candidate finding,
  and the comment-removal escalation rules there mean a "successful" cleanup diff
  can still surface Major findings even when most of the removals in it were
  correct.
- For any action the review would gate that is itself irreversible (this skill
  reviewing the agent's *own* proposed changes, e.g. branch deletions or
  migrations), the default recommendation is Blocker until a human confirms —
  never auto-approve irreversible actions.
