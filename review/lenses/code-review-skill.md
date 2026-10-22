# Code-review skill (built-in) — diff-centric correctness & cleanup pass

Runs Anthropic's **built-in `/code-review` skill** against the change set.
The skill is a packaged review protocol (finder angles → dedupe → recall-biased verify → capped output) that complements the criteria-driven lenses with angles they don't encode — above all the **removed-behavior auditor** (what invariant did each deleted line enforce, and where is it re-established?) and the **cross-file tracer** (does a changed function break its callers/callees?).

This is one of the [lenses](README.md#lenses) the default review runs.
Select it alone with `review <repo> <change> code-review`.

## How it executes — main context, not the Workflow fan-out

Invoking a skill loads its instructions into the **invoking context**, so this lens does **not** run as a subagent inside the Step-2 Workflow.
The orchestrating context invokes it and drives its protocol **in parallel with** the Workflow fan-out of the other lenses.
The skill's own finder/verifier agents are spawned via the Agent tool per its instructions; independence from the other lenses is preserved because none of them see each other's findings before synthesis.

## Step 1 — Invoke and retarget

- Invoke the skill via the Skill tool: `skill: code-review`, `args: high` (the default effort for this lens; a heavier/lighter pass may be chosen deliberately — low/medium: fewer, high-confidence findings; high→max: broader recall).
- **Never** pass `--fix` (would edit the checkout) or `--comment` (would post partial findings to the PR outside the report).
- **Retarget the protocol at the change's diff** — `$BASE..$HEAD` as pinned by [Step 1 of the default review](README.md#step-1--gather-the-change-and-check-out-the-branch-once-shared-by-every-lens), read via the shared worktree `$WORK/.wt/<ref>` and the pre-captured diff materials.
  The skill's own Phase 0 ("current working diff") does **not** apply: the session's working directory is the weblin root, whose diff is never the review subject.
- `/code-review ultra` (cloud multi-agent) is **user-triggered and billed** — this lens must never attempt it.
  If the user wants it, they run it themselves from a session inside a checkout of the reviewed repo.

## Step 2 — Run the skill's finder phase

Follow the loaded protocol as written (at `high`: 3 correctness angles + reuse / simplification / efficiency + altitude + conventions, up to 6 candidates each, spawned via the Agent tool).
Adaptations that experience shows are needed:

- Hand every finder the shared materials: worktree path, pre-captured diff / name-status / stat files, base and head SHAs, and the `git -C "$WORK" show $BASE:<path>` / targeted-diff recipes.
- **Large diffs:** the line-by-line angle cannot literally cover a very large diff.
  Direct it to the core production paths (`lib/`, `background/`, the grains and aspects at the change's center of gravity) and **record the sampling scope in the report's Coverage section** — a sampled sweep must not read as a full one.
- The conventions angle checks the **reviewed repo's** rule files (its `CLAUDE.md`, the weblin root `CLAUDE.md`, `docs/`), never the review workspace's.

## Step 3 — Dedupe, then verify (the skill's recall-biased 1-vote pass)

- Dedupe candidates per the skill (same defect, same location → one), **and against the other lenses' findings**: a candidate that exactly matches a finding which already survived another lens's *refute-biased* verification carries over as CONFIRMED without a second verifier — the stricter pass subsumes the weaker one.
- Verify the remainder per the skill: one verifier per candidate, **PLAUSIBLE by default**, REFUTED only when constructible from the code.
  Keep CONFIRMED + PLAUSIBLE.
  Note this bar is deliberately *lower* than the other lenses' adversarial step — the skill buys recall here and pays with the cap below.
- Verifiers sometimes surface **adjacent bugs** while refuting a candidate; record those as observations for synthesis rather than dropping them.

## Step 4 — Report

- Apply the skill's cap: at most **10 findings, most severe first, correctness outranks cleanup**.
  Below-cap survivors and cleanup/conventions candidates that never competed are listed briefly (one line each) so they aren't lost.
- Map onto the report's severity scale by impact (the skill itself has no severities): a CONFIRMED correctness bug is typically **major** (blocker if it meets a blocker criterion), PLAUSIBLE typically **minor**, cleanup/conventions **minor/nit**.
  Keep the `CONFIRMED`/`PLAUSIBLE` verdict tag on each finding as evidence of the verification level it passed.
- The lens gets its own **Detail** section in the synthesized report, in the position the [lenses table](README.md#lenses) gives it (directly after Engineering), plus a one-line verdict in the Summary.
  Same `file:line` as another lens → merge per [`reports/README.md`](../reports/README.md#format).
- Every finding still needs the house form: `file:line`, **why it matters**, **what to merge instead**.
