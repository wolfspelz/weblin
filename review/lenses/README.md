# Lenses — orchestration

This folder holds the **review lenses** — each a focused runbook applying its own slice of the criteria.
A review runs **one lens, or all of them**; this README is the **orchestrator** that fans out to the selected lenses, verifies, and synthesizes one report.
Each lens runs in its own context and applies only the criteria/patterns relevant to it, so engineering quality and request scope (and every lens added later) are covered in one pass without any one lens's concerns bleeding into another's.

There is no skill wrapping this — **this file is the runbook.**
A review request is the trigger and authorizes the heavy multi-agent run below.
Execute the steps in order.

---

## Trigger

```
review <repo> <PR-number-or-URL-or-branch> [lens] [modifiers] ["<intent>"]
```

- **`<repo>`** → `ext` = `github-n3qExt` (GitHub `lupuslabs/n3qExt`), `server` = `github-nine3q` (GitHub `lupuslabs/nine3q`).
  A PR URL carries the repo and may replace the token.
- **`<PR-number-or-URL-or-branch>`** → a number or URL is a pull request; anything else is a branch name reviewed against `develop`.
- **`<lens>` omitted, or `all`** → run **every active lens** in its own context, then synthesize one report.
  This is the default.
- **a single lens token** → run **only that lens**.
  Tokens, in execution order: `engineering`, `code-review`, `scope`, `architecture`, `security`, `security-adversarial`, `feature-regression`, `cto`.
- **`"<intent>"`** → a quoted sentence stating what the change is supposed to do; handed to the [scope](scope.md) lens as the request, and to every lens as the intent proportionality is judged against.
  Without one, the PR description is the request; for a branch without one, the commit messages are.

Everything above is recognised by **shape**, in any order: `ext`/`server` is the repo, a number/URL/branch is the change, `all` or a lens name selects the lens, a quoted string is the intent.
The modifiers below are recognised by **word**, in any position, and each is independent of the others.

- **`throwaway`** — also `temp`, `tmp`, `transient`, `scratch`, `local`, `nosave`, `dontsave`, `nocommit`.
  Run the review in full, write the report to `reports/` and surface it, then **skip the commit and skip the PR comment**, and say so.
  Nothing else changes — the lenses, the verification and [Step 6](#step-6--tear-down-this-runs-clone)'s teardown are unaffected.
- **`re-run`** — also `rerun`, `redo`, `repeat`, `again`.
  Says the repeat is deliberate: this change already has a report and you are asked for another one anyway.
  Its **only** effect is to suppress the "there is already a report for this change" question, so do not raise it.
  A re-run is committed and posted exactly like any other review; it is **not** a throwaway.
  Combine the two words when you want both.

**These lists are the recognised words, not the whole language.**
Any plain-language equivalent carries the same meaning — "don't commit this", "just show me", "keep it local" is a throwaway; "yes, do it again" is a re-run.
Judge intent, not spelling.
When a modifier is present and you cannot tell whether it means throwaway, **treat it as a throwaway and say so**: asking for the commit and the comment afterwards costs the user one line, while a comment the PR author has already read cannot be un-read.

## Lenses

**The order lenses are listed here is the execution order** — selected lenses are dispatched, and their sections in the synthesized report ordered, top-to-bottom as below.
To reorder, move a row.

| Lens | Runbook | Status |
|------|---------|--------|
| Engineering (core) | [engineering.md](engineering.md) | active |
| Code-review skill (built-in) | [code-review-skill.md](code-review-skill.md) | active |
| Scope | [scope.md](scope.md) | active |
| Architecture | [architecture.md](architecture.md) | planned |
| Security | [security.md](security.md) | planned |
| Adversarial security | [security-adversarial.md](security-adversarial.md) | planned |
| Feature regression | [feature-regression.md](feature-regression.md) | planned |
| CTO (long-term stewardship) | [cto.md](cto.md) | active |

Active lenses run on the default (`all`) review; planned lenses are **skipped until fleshed out**, each joining the default in its listed position once its `planned` status is removed.

## Step 0 — Parse the request

From the trigger, resolve the **repo**, the **change** (PR number or branch), an optional **intent**, the **lens**, and any **[modifiers](#trigger)**.
Omitted or `all` → the full active set; a single lens token → just that one.
Note a `throwaway` here and carry it to Step 5 — it is the one thing on the trigger that no lens sees and that only takes effect at the very end of the run.

## Step 1 — Gather the change *and* check out the branch (once, shared by every lens)

**Provision an isolated clone for this run first — never review inside the developer's checkouts.**
`C:/Heiner/weblin/github-n3qExt` and `C:/Heiner/weblin/github-nine3q` are the developer's own working copies and serve only as a hardlink base; reviews run in a disposable per-run clone under `work/` so several reviews (e.g. the same change under different models) can run in parallel without racing on one working tree, `FETCH_HEAD`, or index lock.
The call fetches both repos first, so the run starts from current objects and refs — no separate pull step is needed:

```bash
REVIEW=C:/Heiner/weblin/review                                            # this workspace, absolute
date +%s                                                                  # run start — Step 5 turns it into the report's Runtime
WORK=$("$REVIEW"/bin/review-clone <ext|server> <repo>-<ref>-<slug>-<suffix>-<model>)   # prints the clone path
```

**Pin `$REVIEW` first and address every workspace tool through it.**
From Step 1 on, the working directory is `$WORK`, so a relative `bin/review-clone` resolves inside the *reviewed* repo and dies with `No such file or directory` — always `"$REVIEW"/bin/…`.

**The clone slug is the report's name minus the leading timestamp and `.md`** — `<repo>-<ref>-<slug>-<suffix>-<model>` (`<repo>` = `ext`/`server`; `<ref>` = `PR-<n>`, or the branch name with `/` replaced by `-`; `<slug>` = one lowercase word characterizing the change, read off the PR title, branch name or first commit — e.g. `badgetool`; `<suffix>` = `all` or the single lens; `<model>` = `opus`/`sonnet`/`fable`/`haiku`), so a clone and its report share one identity (clone `work/ext-PR-12-badgetool-all-fable` ↔ report `…-ext-PR-12-badgetool-all-fable.md`).
Derive the `<slug>` **before** this call — it is fixed once, and reused verbatim in the report name at Step 5.
The trailing `<model>` is what lets two same-change runs coexist.
The clone hardlinks the code repo's objects (fast, ~1 MB extra) and points `origin` at GitHub.
Everything below runs inside `$WORK`, never the developer's checkout.
Substitute `<N>` or `<branch>`:

```bash
cd "$WORK"
git fetch origin develop                          # refresh develop FIRST — a stale base skews the diff
git fetch origin refs/pull/<N>/head               # PR review: the PR head → FETCH_HEAD
git fetch origin <branch>                         # branch review: the branch tip → FETCH_HEAD
HEAD=$(git rev-parse FETCH_HEAD)                   # pin it before anything clobbers FETCH_HEAD
BASE=$(git merge-base $HEAD origin/develop)        # = the merge-base GitHub diffs the PR against
git diff $BASE..$HEAD                 # the author's real change set
git diff --stat $BASE..$HEAD          # size at a glance (proportionality)
git log --format='%h %s%n%b' $BASE..$HEAD          # the commits — for a branch review, the stated intent
gh pr view <N> --comments -R lupuslabs/<n3qExt|nine3q>                       # PR review: description + discussion; `url:` → the report's PR link
gh pr view <N> --json headRefName -R lupuslabs/<n3qExt|nine3q> -q .headRefName   # PR review: branch name for the report header
git worktree add .wt/<ref> $HEAD      # full branch checkout — read beyond the diff
```

**`gh` guardrail — name the repository, keep stderr.**
`gh` resolves the repository from the *current directory's* git remote, so it must either run inside `$WORK` or carry `-R lupuslabs/<repo>`; from the review workspace it would fail on the weblin root, which has no GitHub remote.
Never discard stderr on a data fetch, and never pipe a response straight into a parser: capture it, eyeball it, then parse.
If `gh` is missing or unauthenticated, say so, continue the review from the git data alone, and record in Coverage that the PR description and discussion were not read.

**Shell variables die with each command.**
`$REVIEW`, `$WORK`, `$BASE` and `$HEAD` live only inside the one shell invocation that set them — a later command starts fresh in the *original* working directory with none of them set.
Keep each block above self-contained (re-`cd`, re-derive, or write the literal absolute paths), and record `$WORK`, `$BASE` and `$HEAD` as literal values in your notes the moment they are computed, so Step 2's dispatch and Step 6's teardown can quote them verbatim.
The same holds for the start `date +%s` — write the number down; Step 5 subtracts it and there is no variable left to read it from.

**Base guardrail — the diff must match the PR, or every lens reviews a phantom change.**
Always diff `$BASE..$HEAD` with `$BASE` the merge-base above.
**Never** diff against a `develop` *tip* — neither two-dot `git diff origin/develop $HEAD` nor the worktree's local `develop`.
A `develop` that has advanced past the branch point sweeps *its own* later commits into the diff as if the author wrote them: inflated file/line counts and phantom "out-of-scope features" the change never touched.
**Sanity check before fanning out:** for a PR, the `git diff --stat` file count must equal the PR's **Files changed** count on GitHub.
If it doesn't, the base is wrong — stop and fix it, don't review the phantom diff.

Capture the diff, the diff stat, and the change's stated intent **once** — every lens reviews the same change set, and the intent is what "proportionality" is judged against.
The intent, in order of preference: the quoted sentence on the trigger; the PR description; the commit messages.
Record which source it came from — the scope lens reports it.

**The diff is the starting point, not the boundary.**
Always check the branch out into a worktree under this run's clone (`$WORK/.wt/<ref>`) so every lens can read the *full files the change touches and the code around them* — callers, callees, the types and interfaces involved, the existing patterns the change should have followed.
A review judges how the change fits the **existing codebase as it stands on the branch**, which the hunks alone never show: a deletion's blast radius, a new function's real callers, an abstraction that already existed, a convention the diff silently breaks.
Investigate the branch deeply enough to understand *what was implemented and how it fits* before judging it.

**Read the project's own rules before fanning out.**
The reviewed repo's root `CLAUDE.md` (code style, naming, patterns), the weblin root `C:/Heiner/weblin/CLAUDE.md`, and the architecture docs under `C:/Heiner/weblin/docs/` are the project's standard — see [`criteria/project.md`](../criteria/project.md).
Hand their absolute paths to every lens.

## Step 2 — Fan out: one context per lens

Run as a Workflow — except the [code-review skill lens](code-review-skill.md), which a skill invocation loads into the **orchestrating context** and which therefore runs there, in parallel with the Workflow fan-out (its own finder/verifier subagents are spawned per its protocol).
Every other selected lens runs in its **own context** (a separate subagent), handed the shared change + intent **and the branch worktree path** (`$WORK/.wt/<ref>`, an absolute path), told to follow its own runbook and loading only the criteria/patterns that lens applies.
**Spell every path out in full in the dispatch prompt** — the review workspace root, the clone, the worktree, the rule files, and `$BASE`/`$HEAD` as raw SHAs.
A subagent gets a fresh shell in its own working directory: it inherits none of your variables, so a lens told to diff `$BASE..$HEAD` simply fails.
Each lens is expected to **read beyond the diff** — open the full touched files and the surrounding code in the worktree to ground its findings in how the change fits the existing branch, not just the hunks.
Lenses don't see each other's work — that independence is the point.
Dispatch the selected lenses — and order their sections in the synthesized report — in the order listed under [Lenses](#lenses) above:

- [Engineering](engineering.md) — the core criteria + their patterns + the project's own rules; itself a multi-agent dimension fan-out.
- [Code-review skill](code-review-skill.md) — the built-in `/code-review` skill's diff-centric protocol (removed-behavior audit, cross-file trace, line scan + cleanup/conventions angles) at effort `high`, retargeted at the change's diff; runs in the main context, never with `--fix`/`--comment`, never `ultra`.
- [Scope](scope.md) — the solution vs. the stated request (runs when an intent, PR description or commit messages exist; reports *skipped — no request* otherwise).
- [CTO](cto.md) — long-term stewardship: maintainability/extensibility, reliability of the extension and the silo in long-running operation, user privacy, and whether functionality is guaranteed (by clean engineering or by behavioural proof); itself a multi-agent accountability fan-out.
- …each planned lens, once active, in its listed slot (Architecture → Security → Adversarial security → Feature regression, ahead of CTO).

## Step 3 — Verify (adversarial)

Every lens verifies its own non-trivial findings with independent verifier(s) prompted to **refute** them (default to "refuted" when uncertain).
Findings that don't survive are dropped or downgraded.
This is the guard against confident-but-wrong AI findings — no lens skips it.

**One carve-out:** a **test-triviality** finding (one invoking [`criteria/test.md`](../criteria/test.md) *Useless tests — concrete shapes to flag*) is **not** subject to the refute-default — it is verified per that file's *Verifying a "useless test" finding* rule (confirm the shape; "covered elsewhere" confirms deletion, it does not refute).
Without this, refute-biased verification silently drops exactly the trivial-test signal the criterion demands be reported.

## Step 4 — Synthesize the report

Reconcile the per-lens results into one report, using the format in [`reports/README.md`](../reports/README.md#format) (the most severe lens usually sets the overall verdict; where two lenses flag the same line, merge them).
Order the per-lens **Detail** sections by the lens order under [Lenses](#lenses) (Engineering → Code-review → Scope → … → CTO).
The verdict is **advisory** — the human makes the merge decision.

## Step 5 — Save the report, commit it, and post it to the PR

Write the report to `reports/`, named per [`reports/README.md`](../reports/README.md#file-name), and surface it to the user.
**Close the clock first** — the header's `Runtime` line needs it:

```bash
date +%s   # run end; Runtime = this minus the Step 1 start, in whole minutes ("1 h 47 min")
```

Then **commit it** into the weblin root repository — its own commit, that report's explicit path only — per the procedure and guardrails in [`reports/README.md`](../reports/README.md#committing).

Finally, for a PR review, **post it to the pull request** as the body of a comment so GitHub renders it, per [`reports/README.md`](../reports/README.md#posting-to-the-pull-request).
A branch review has nothing to post to; say so.

**End the terminal summary with the link.**
The last thing the user sees is where to go next, so close the summary with the PR link when there is one, else the branch name and the report path:

```
PR 12 · ext: Badge tool …
https://github.com/lupuslabs/n3qExt/pull/12
```

- **Bare, absolute URLs, one per line** — that is what a terminal turns into a click target.
  Markdown link syntax hides the URL behind a label and is not reliably clickable, so do not use it here.
- **The same URL the report header already carries** ([`reports/README.md`](../reports/README.md#the-header-list)) — the `url:` field from Step 1's `gh pr view`.
- **Shown on every run, including a `throwaway`** — the link is how the user reaches the PR, not part of committing or posting.

## Step 6 — Tear down this run's clone

Once the report is saved and surfaced, **always remove this run's clone** so `work/` holds only in-progress reviews:

```bash
C:/Heiner/weblin/review/bin/review-clone --rm <repo>-<ref>-<slug>-<suffix>-<model>   # same slug used in Step 1
```

`C:/Heiner/weblin/review/bin/review-clone --list` shows any clones still live.
Nothing outside `work/` is modified by a review: the code repos are only fetched into, and the developer's working trees are never read at all.
