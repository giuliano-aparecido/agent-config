---
name: clean-code
description: The Clean Code rules (Robert C. Martin) shared by whoever writes code and whoever reviews it. Read before writing or changing any code, and when reviewing a diff. Covers naming, functions, comments, formatting, objects, error handling, boundaries, classes, tests, and smells. Reviews use the code-review skill, which builds on these rules.
---

# Clean Code

One rule set for writing and reviewing. A writer follows it so the reviewer
finds nothing; a reviewer flags only what it forbids. Source: *Clean Code*
(Martin, 2008); chapter numbers in parentheses. A few rules are house rules
stricter than or beyond the book, marked *(house)*: constant/unit comments and
worked examples in comments and suppression directives (section 3), parse
external data once (section 7),
a final raising `else` (section 11). Where a repo's own
`AGENTS.md`/`CONTRIBUTING.md` documents a deliberate deviation, the repo wins.

Leave the code cleaner than you found it (Boy Scout Rule), but only inside
the code you are already changing.

## 1. Names (ch. 2)
- Intention-revealing: the name says why it exists, what it does, how it is used.
  If a name needs a comment, rename it instead.
- No disinformation (`accountList` that is not a list), no noise words
  (`data`, `info`, `manager`, `helper`, `temp`), no encodings or prefixes.
- Meaningful distinctions, pronounceable and searchable names. A literal used
  more than once, or whose meaning is not obvious, becomes a named constant.
  Put the unit in the name when it is not obvious (`timeout_seconds`).
- Classes and variables are nouns, methods are verbs, booleans read as
  predicates (`is_valid`). One word per concept across the codebase
  (not `fetch`/`get`/`retrieve` for the same thing).
- Name length follows scope: short in a small scope, long and exact in a wide one.
- A vague verb or a name that only restates its body (`process`, `handle`,
  `_solve_rate`) is a finding even when a docstring explains it. Rename so the
  name says what is computed and how (`_find_rate_where_npv_is_zero`). A name
  that lies (`doublings` holding a multiplier) is worse than a vague one.

## 2. Functions (ch. 3)
- Small, and do one thing. Check: can you extract another function from it
  with a name that is not a restatement of its body? Then it does more than one thing.
- One level of abstraction per function; read top-down (step-down rule).
- Arguments: zero is best, then one, two, three. More than three become an object.
  No flag arguments (split into two functions). No output arguments.
- No side effects, no hidden temporal coupling. Command-query separation:
  change state or answer a question, not both.
- Prefer exceptions to error codes. Extract try/catch bodies into their own function.
  Error handling is one thing, so a function that handles errors does nothing else.
- Nesting stays shallow. Replace a conditional that switches on a type with
  polymorphism. Encapsulate non-trivial conditions in a well-named function and
  prefer positive conditions to negative ones.
- No duplication; one decision lives in one place. Do not merge code that only
  looks alike but changes for different reasons.

## 3. Comments (ch. 4)
Comments compensate for a failure to express ourselves in code. Before
writing one, try to rename, extract a function, or introduce an explanatory
variable. If that works, no comment.

Acceptable, when the code cannot say it by itself:
- **Intent**: why a decision was taken, when the code shows only what.
- **Clarification** of something you cannot change (a library return value, a standard).
- **Warning of consequences** ("not thread-safe", "this test takes minutes").
- **TODO** with an owner or ticket, **amplification** of an easily missed detail.
- **Public API docs** for code used by callers who cannot read the source.
  Not a docstring on every function. A new public interface with none is a gap.
- Also a gap: code that hides a non-obvious intent or a dangerous consequence
  (a workaround for a library bug, a required call order) with no comment saying so.

Not acceptable, and each one is a finding:
- Redundant or narrating comments (restate the name or the next line), mumbling,
  misleading or stale comments, noise ("default constructor"), mandated boilerplate.
- Comment that makes up for a bad name (rename instead).
- *(house)* A comment on a constant, or a trailing comment, that only gives its unit,
  origin, or meaning (`MAX_DELAY = 30  # the upstream API's timeout`,
  `LOWER = -0.99  # -99% / year`). Fold it into the name
  (`UPSTREAM_TIMEOUT_SECONDS`, `LOWER_RATE_PER_YEAR`).
- *(house)* A worked example or arithmetic in a comment (`# 0.5 * 2**6 = 32 would exceed
  the cap`, `# +4% over 3 months is ~+17%`). Put it in a test with a name that
  states it, or in the PR.
- **Too much information**: history, an investigation, worked numbers, long
  tutorials. That belongs in the commit message, the PR, a test, or a design doc.
  Keep the constraint a future editor must respect, in one or two sentences.
- **Non-local information**: describing code elsewhere ("see `span_days`",
  "used by X and Y"). Say it where the thing lives, or nowhere.
- One comment explaining several unrelated decisions: put each next to the part it explains.
- The same fact repeated in several comments: keep one, at the place that depends on it.
- Docstrings on private or internal functions that restate the signature.
- Journal or attribution comments, position markers, closing-brace comments,
  commented-out code (delete it, version control remembers).
- *(house)* Deleting a suppression directive (`# noqa`, `eslint-disable`,
  `@ts-ignore`) is a behavior change, not a comment cleanup.

## 4. Formatting (ch. 5)
- Follow the repo's formatter/linter config; do not hand-format against it.
- Newspaper layout: high-level first, details below; a caller sits above what it calls.
- Related code stays vertically close, unrelated code is separated by a blank line.
  Declare variables near their use. Keep lines and files short enough to read without scrolling sideways.

## 5. Objects and data structures (ch. 6)
- An object hides data and exposes behavior; a data structure exposes data and
  has no behavior. Do not build hybrids.
- Law of Demeter: talk to friends, not to strangers (`a.b().c().d()`).
  Chains that only walk a plain data structure are fine.

## 6. Error handling (ch. 7)
- Exceptions carry context (what was attempted, with what input, why it failed).
- Do not return `null`; do not pass `null`. Return an empty collection, a
  special case object, or raise.
- Never swallow an exception, and do not catch a broader type than you can handle.
- Wrap a third-party API's exceptions at the boundary into your own type.

## 7. Boundaries (ch. 8)
- Wrap third-party and external APIs behind an interface you own, so the
  rest of the code depends on your shape, not theirs.
- Write a learning test when you adopt an external API you do not know.
- *(house)* Validate and parse external data once where it enters; inside, trust it.

## 8. Classes (ch. 10)
- Small by responsibility, not by line count. SRP: describe the class in one
  sentence without "and". A class that grew a second reason to change in this
  diff needs splitting.
- High cohesion: methods use most of the instance variables. When a few
  methods use a few variables, extract a class.
- LSP: a subclass or override must not narrow accepted input, add new
  exceptions, or change return semantics the base type promised
  (`NotImplementedError` on an inherited method is a violation).
- ISP: do not force implementers to depend on methods they do not use; prefer
  several small interfaces to one broad one.
- OCP: add behavior by extension, not by editing a stable class. DIP: depend
  on abstractions, not on a concrete DB or HTTP client, when that hurts testing.
- Organize for change; isolate code that changes for a different reason.

## 9. Systems and design (ch. 11, 12)
- Separate construction from use: no `new`/connect inside business logic, inject.
- Simple design, in this order: runs all the tests; no duplication; expresses
  the programmer's intent; minimizes the number of classes and methods.
  Never add structure the current need does not justify.

## 10. Tests (ch. 9)
- Write the failing test first when the behavior is new or a bug is being fixed;
  see it fail for the right reason, then make it pass.
- F.I.R.S.T.: Fast, Independent, Repeatable, Self-validating, Timely.
- One concept per test. Readable: arrange, act, assert, with the same care as production code.
- Test names state behavior. Test behavior through the public interface, not internals.
- Cover boundary conditions and error paths, not only the happy path.
- No skipped or disabled test without a tracked reason.

## 11. Smells and heuristics (ch. 17)
Flag, and when writing avoid:
- Dead code, unused imports, variables, parameters; unreachable code.
- Duplication (G5), inconsistency for the same concept (G11), clutter (G12).
- Magic numbers (G25), obscured intent (G16), explanatory variables missing (G19).
- Feature envy (G14), inappropriate static (G18), selector arguments (G15).
- Base classes that depend on their derivatives (G7), too much information in an interface (G8).
- Negative conditionals (G29), unencapsulated boundary conditions (G33), hidden
  temporal coupling (G31), functions that do more than one thing (G30).
- *(house)* Non-exhaustive dispatch over a closed set of types: a final `else`/`default`
  must raise so a new variant fails loudly instead of being dropped.
