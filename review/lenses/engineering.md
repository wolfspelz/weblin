# Engineering Review (core lens)

The baseline lens, run on every change.
A **heavy multi-agent** pass: independent reviewers each own one engineering dimension, findings are adversarially verified, then synthesized.
This catches more than a single pass and filters out plausible-but-wrong findings.

This is one of the [lenses](../README.md#lenses) the [default review](README.md) runs in its own context.
Select it alone with `review <repo> <change> engineering`.

## Criteria & patterns this lens applies

- **Criteria** — the five in [`criteria/engineering.md`](../criteria/engineering.md): §1 Proportionality, §2 Simplicity, §3 Blast radius, §4 Clean code & naming, §5 Engineering patterns.
- **Anti-patterns** ([`criteria/anti-patterns.md`](../criteria/anti-patterns.md)) — Gratuitous rewrite / scope creep · Over-naming · Premature abstraction · Drive-by churn · Duplicated source of truth · Swallowed errors · Unbounded blocking · Hidden temporal coupling.
- **Desired patterns** ([`criteria/desired-patterns.md`](../criteria/desired-patterns.md)) — Surgical proportional change · Intentional called-out refactors · Idempotent, restart-safe operations · Fail closed, degrade gracefully.
- **Tests** ([`criteria/test.md`](../criteria/test.md)) — Meaningful unit tests · Useless tests · Test the real production code, isolated by DI · Test-only production path · Test helpers shipped in production code.
  Plus this lens's own [**Behaviour coverage**](#behaviour-coverage--the-delivered-behaviour-must-be-pinned-by-a-test) requirement.
- **Project conventions** ([`criteria/project.md`](../criteria/project.md)) — the project's **own** rules, applied straight from the reviewed repo's `CLAUDE.md`, the weblin root `CLAUDE.md`, and the architecture docs in `docs/`.
  `project.md` defers to them and re-encodes nothing, so this lens always judges against the project's *current* standard.

Each reviewer cites the criterion / pattern it invokes **by its heading**.

## Input

The change set + the stated intent, gathered once by the [default review](README.md) (Step 1).
Running this lens standalone?
Gather them first via that step — the intent is what proportionality is judged against, so read it before judging the code.

## Step 1 — Fan out (one reviewer per dimension)

Run as a Workflow: spawn the dimension reviewers in parallel, each given the diff + intent + worktree path + the rule-file paths, and told which criteria to apply.
Each reviewer cites the relevant criterion / pattern entry **by its heading**.

1. **Scope & blast radius** — proportionality vs. the stated problem; unrelated or drive-by changes.
   (engineering §1, §3; anti-patterns *Gratuitous rewrite*, *Drive-by churn*; desired-patterns *Surgical proportional change*, *Intentional called-out refactors*.)
   Mind the project's legacy rule: `CLAUDE.md` says legacy code is converted *when touched*, so style conversion of lines the change already edits is expected, not churn — conversion of untouched lines is.
2. **Simplicity** — simplest solid solution vs. bloat / speculative generality.
   (engineering §2; anti-patterns *Premature abstraction*.)
3. **Naming & clean code** — accurate names, small functions, single responsibility.
   (engineering §4; anti-patterns *Over-naming*.)
   The project's naming rules (`Get` prefix, boolean predicates, `Make` for factories, `SendXxx`/`HandleXxxRequest`, `OnXxx`, no metaphors) live in the repo's `CLAUDE.md` and are checked by dimension 6; this dimension judges the names that no rule reaches.
4. **Engineering principles** — single source of truth, separation of concerns, SOLID.
   (engineering §5; anti-patterns *Duplicated source of truth*.)
5. **Correctness & risk** — bugs, edge cases, regression risk, and **meaningful unit tests** (do the tests guard real future breakage, or are they TDD relics?).
   (tests *Meaningful unit tests*, *Useless tests*, *Test the real production code, isolated by DI*, *Test-only production path*; anti-patterns *Swallowed errors*, *Unbounded blocking*, *Hidden temporal coupling*.)
   Extension-specific risks to probe: `setTimeout`/`setInterval` in background code (forbidden by `CLAUDE.md`), message ordering assumptions across entities (the server-notification ordering rule), shadow-DOM leaks, `any` or un-narrowed boundary data.
   Server-specific risks: grain-to-grain deadlocks, mutable grain state records, missing `[TestCategory]`, `out` parameters, untyped `throw new Exception`.
6. **Project conventions & idioms** — read the reviewed repo's `CLAUDE.md`, the weblin root `CLAUDE.md`, and every `docs/` document that covers the touched area (per [`project.md`](../criteria/project.md)) and check the changed code against them.
   Cite each finding by the **rule's** file and heading; raise any rule that looks stale or self-contradictory rather than overriding it.
   Three project-level checks belong here: the **documentation gate** (does a `docs/` document cover the touched area at all, and does the change keep it accurate?), the **mirror rule** (an edit to a mirrored `CLAUDE.md` section must land in both repos), and the **changelog rule** (a user-visible change is recorded in `_Changes.ts` / `Changes.cs`).
7. **Behaviour coverage** — is the delivered behaviour pinned by a test that would go red if this change were reverted?
   The full rule, including what does *not* count and the severities, is [below](#behaviour-coverage--the-delivered-behaviour-must-be-pinned-by-a-test).

## Step 1b — Count the change (fill the statistics table)

This lens owns scope/blast-radius, so it also produces the change **statistics** the report's Summary shows (see [`reports/README.md`](../reports/README.md#format)).
Count straight from the gathered diff — the same change set the dimension reviewers judge — and emit the fixed row set, keeping every row even when a count is `0`:

| Stat | How to count |
|------|--------------|
| Files changed | every changed path in the diff (code, tests, scaffolding, docs, config) |
| Code files changed | all changed `.ts`/`.cs` files (added + modified + deleted), tests included |
| Code files changed (excluding tests) | changed `.ts`/`.cs` files that are **not** under a test path |
| Code lines delta (excluding tests) | added minus removed lines in those files |
| Scaffolding files changed (non `.ts`/`.cs`) | changed non-source files (`.json`, `.csproj`, `.mjs`, `.css`, `.html`, `.cshtml`, CI, lock files) |
| Code files new (excluding tests) | added (`A`) non-test source files |
| Code files deleted (excluding tests) | deleted (`D`) non-test source files |
| Unit test files | changed unit-test files |
| Unit test functions/methods | test functions/methods in those files |
| Grain/integration test files | changed test files that start a silo or cluster |
| Grain/integration test functions/methods | test methods in those files |
| Docs in `docs/` touched | changed files under the weblin root `docs/` — usually `0`, since the docs live in a different git repo than the change; count them when the user names a docs commit |

**Test paths by repo.**
Extension: `ChromeExt/src/test/Test*.ts` are unit tests (Chai); there is no integration tier.
Server: `*.Test` projects (`n3q.Items.Test`, `n3q.Aspects.Test`, `n3q.WebIt.Test`, `n3q.WebEx.Test`, `Shared/*.Test`) are unit tests; `Items/n3q.GrainTests` (on `IsolatedSiloTestBase` from `n3q.TestTools`) is the grain/integration tier; `n3q.SiloTestClient` and `ConfigSharp.TestClient` are manual clients, not tests.
Count test methods with `grep -c "^\s*it(" ` in the extension and `grep -c "\[TestMethod\]"` on the server.
Exclude files dragged in by an unrelated merge commit (note them in Coverage).
Hand these counts to the synthesis step so the report's Summary table is filled.

## Behaviour coverage — the delivered behaviour must be pinned by a test

**The requirement.**
A change that delivers a feature or fixes a bug ships at least one test that **drives the production code it changed and asserts the behaviour it delivers**, in the repo's own test tier: a Chai test under `ChromeExt/src/test/` for the extension, an MSTest test in the owning `*.Test` project or in `n3q.GrainTests` for the server.
weblin has no cross-service harness between extension and server, so this rule asks for the strongest test the repo *can* express, not for a system test it cannot.

**What counts.**
All three must hold:

- **The changed production code is on the executed path.**
  Name the changed file and symbol the test actually drives.
  The decisive question: **if this change's production diff were reverted, would this test go red?**
  If it would still pass, it covers the old system, not the new behaviour.
- **The real thing runs, not a stand-in for it.**
  Fakes and in-memory collaborators are legitimate for the **peers** the code under test talks to (a fake `BackgroundToContentCommunicator`, an in-memory storage provider, the isolated test silo) — never for the thing the change delivers.
  This is [`criteria/test.md`](../criteria/test.md) *Test the real production code, isolated by dependency injection*.
- **It drives the real entry point.**
  A test that reaches the new behaviour through a method, flag or hook that exists only for the harness is *Test-only production path* ([`criteria/test.md`](../criteria/test.md)) and does not count.

**What does not count** on its own: an assertion-free test, a mock-asserting test, a tautological test — the useless-test shapes of [`criteria/test.md`](../criteria/test.md) apply here too, and such a test is a finding in its own right **and** leaves the behaviour uncovered.

**Where behaviour can be asserted on the server**, prefer the grain tier: `n3q.GrainTests` runs the real grains on an isolated silo, so a test there pins what WebIt and the extension observe.
A unit test in `n3q.Items.Test` pins the item logic beneath it; both are valid, the grain test is the stronger evidence for grain changes.

**Severity.**

- `major` — the change delivers a feature or a bug fix and ships **no** qualifying test, and the repo's test tier could have expressed one.
  A test that ships but would stay green with the production diff reverted counts as **none** — same severity, and say so.
- `minor` — a qualifying test exists but covers only part of the delivered behaviour, or the assertion stops at an intermediate step.
- **Not applicable** — pure refactors, chores, tooling, docs, and UI code whose behaviour is DOM layout that the test harness cannot observe (say which).
  Say so explicitly in the verdict rather than skipping the dimension silently, so a reader can tell "no behaviour to cover" from "nobody looked".

**Evidence, either way.**
A finding cites the test `file:line` that covers the behaviour **together with the changed production symbol it drives**, or names the behaviour and states that no test in the diff or on the branch exercises it.
"A test was added" is not evidence of coverage and must not be reported as if it were.
Search the branch worktree, not just the diff: the covering test may already exist and need only an added case.

## Step 2 — Verify (adversarial)

Every non-trivial finding is checked by independent verifier(s) prompted to **refute** it (default to "refuted" when uncertain).
Findings that don't survive are dropped or downgraded.
This is the guard against confident-but-wrong AI findings — do not skip it.

**Exception — missing-coverage findings are refuted only by naming the test.**
A verifier refutes a [Behaviour coverage](#behaviour-coverage--the-delivered-behaviour-must-be-pinned-by-a-test) finding only by **naming a test that meets all three counting conditions and demonstrably runs this change's changed code** (which changed symbol it drives, and why reverting the diff would turn it red), or by showing the dimension is not applicable.
"A test was added in this change" refutes nothing on its own.

**Exception — test-triviality findings are not refute-dropped.**
A finding that invokes [`criteria/test.md`](../criteria/test.md) *Useless tests — concrete shapes to flag* is verified per that file's *Verifying a "useless test" finding* rule, **not** by the default-refuted bias: confirm the shape is real, and treat "the behaviour is covered elsewhere" as confirming deletion, never as a refutation.
Adjust severity if warranted, but do not drop it once the shape is confirmed.

**Exception — project-rule findings are refuted only by the rule text.**
A dimension-6 finding cites a rule in `CLAUDE.md` or `docs/`; a verifier refutes it by quoting the rule and showing the code complies, or by showing the rule marks the form as accepted legacy on untouched lines.
"The rule is too strict" is an observation for the human, not a refutation.

## Step 3 — Engineering verdict

- **Verdict** — merge / merge-with-nits / needs-work / reject + one-sentence why.
- **Findings** grouped by dimension, severity-ordered, each with `file:line`, **why it matters**, and **what to merge instead**.
  Nitpicks clearly separated so they don't drown the signal.
- **Statistics** — the filled table from Step 1b, passed through to the combined report's Summary section.

This verdict feeds the combined report the [default review](README.md) synthesizes across lenses.
It is **advisory** — the human makes the merge decision.
