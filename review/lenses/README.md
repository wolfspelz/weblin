# Lenses — orchestration

This folder holds the **review lenses** — each a focused runbook applying its own slice of the criteria.
A review runs **one lens, or all of them**;
this README is the **orchestrator** that fans out to the selected lenses, verifies, and synthesizes one report.
Each lens runs in its own context and applies only the criteria/patterns relevant to it,
so engineering quality and request scope (and every lens added later) are covered in one pass
without one lens's concerns mixing into another's.

There is no skill wrapping this — **this file is the runbook.**
A review request is the trigger and authorizes the heavy multi-agent run below.
Execute the steps in order.

---

## Trigger

```
review <repo> <PR-number-or-URL-or-branch> [lens] [modifiers] ["<intent>"]
```

- **`<repo>`** → `ext` = `github-n3qExt` (GitHub `lupuslabs/n3qExt`),
  `server` = `github-nine3q` (GitHub `lupuslabs/nine3q`).
  A PR URL carries the repo and may replace the token.
- **`<PR-number-or-URL-or-branch>`** → a number or URL is a pull request;
  anything else is a branch name reviewed against `develop`.
- **`<lens>` omitted, or `all`** → run **every active lens** in its own context, then synthesize one report.
  This is the default.
- **a single lens token** → run **only that lens**.
  Tokens, in execution order:
  `engineering`, `code-review`, `scope`, `architecture`, `security`, `security-adversarial`,
  `feature-regression`, `cto`.
- **`"<intent>"`** → a quoted sentence stating what the change is supposed to do;
  handed to the [scope](scope.md) lens as the request,
  and to every lens as the intent proportionality is judged against.
  Without one, the PR description is the request; for a branch without one, the commit messages are.

Everything above is recognised by **shape**, in any order:
`ext`/`server` is the repo, a number/URL/branch is the change, `all` or a lens name selects the lens,
a quoted string is the intent.
The modifiers below are recognised by **word**, in any position, and each is independent of the others.

- **`throwaway`** — also `temp`, `tmp`, `transient`, `scratch`, `local`, `nosave`, `dontsave`, `nocommit`.
  Run the review in full, write the report to `reports/` and surface it,
  then **skip the PR comment**, and say so.
  (No run commits its report, throwaway or not — see [Step 5](#step-5--save-the-report-and-post-it-to-the-pr).)
  Nothing else changes
  — the lenses, the verification and [Step 6](#step-6--tear-down-this-runs-clone)'s teardown are unaffected.
- **`re-run`** — also `rerun`, `redo`, `repeat`, `again`.
  Says the repeat is deliberate: this change already has a report and you are asked for another one anyway.
  Its **only** effect is to suppress the "there is already a report for this change" question,
  so do not raise it.
  A re-run is posted exactly like any other review; it is **not** a throwaway.
  Combine the two words when you want both.

**These lists are the recognised words, not the whole language.**
Any plain-language equivalent carries the same meaning
— "don't commit this", "just show me", "keep it local" is a throwaway; "yes, do it again" is a re-run.
Judge intent, not spelling.
When a modifier is present and you cannot tell whether it means throwaway,
**treat it as a throwaway and say so**: asking for the comment afterwards costs the user one line,
while a comment the PR author has already read cannot be withdrawn.

## Lenses

**The order lenses are listed here is the execution order**
— selected lenses are dispatched, and their sections in the synthesized report ordered, top-to-bottom as below.
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

Active lenses run on the default (`all`) review;
planned lenses are **skipped until completed**,
each joining the default in its listed position once its `planned` status is removed.

## Step 0 — Parse the request

From the trigger, resolve the **repo**, the **change** (PR number or branch), an optional **intent**,
the **lens**, and any **[modifiers](#trigger)**.
Omitted or `all` → the full active set; a single lens token → just that one.
Note a `throwaway` here and carry it to Step 5
— it is the one thing on the trigger that no lens sees
and that only takes effect at the very end of the run.

## Step 1 — Gather the change *and* check out the branch (once, shared by every lens)

**Provision an isolated clone for this run first — never review inside the developer's checkouts.**
`C:/Heiner/weblin/github-n3qExt` and `C:/Heiner/weblin/github-nine3q` are the developer's own working copies
and serve only as a hardlink base;
reviews run in a disposable per-run clone under `work/`
so several reviews (e.g. the same change under different models) can run in parallel
without racing on one working tree, `FETCH_HEAD`, or index lock.
The call fetches both repos first, so the run starts from current objects and refs
— no separate pull step is needed:

```bash
REVIEW=C:/Heiner/weblin/review                                            # this workspace, absolute
date +%s                                                                  # run start — Step 5 turns it into the report's Runtime
WORK=$("$REVIEW"/bin/review-clone <ext|server> <repo>-<ref>-<slug>-<suffix>-<model>)   # prints the clone path
```

**Set `$REVIEW` first and address every workspace tool through it.**
From Step 1 on, the working directory is `$WORK`,
so a relative `bin/review-clone` resolves inside the *reviewed* repo
and fails with `No such file or directory` — always `"$REVIEW"/bin/…`.

**The clone slug is the report's name minus the leading timestamp and `.md`**
— `<repo>-<ref>-<slug>-<suffix>-<model>`
(`<repo>` = `ext`/`server`; `<ref>` = `PR-<n>`, or the branch name with `/` replaced by `-`;
`<slug>` = one lowercase word characterizing the change, read off the PR title, branch name or first commit
— e.g. `badgetool`;
`<suffix>` = `all` or the single lens; `<model>` = `opus`/`sonnet`/`fable`/`haiku`),
so a clone and its report share one identity
(clone `work/ext-PR-92-badgetool-all-fable` ↔ report `…-ext-PR-92-badgetool-all-fable.md`).
Derive the `<slug>` **before** this call
— it is fixed once, and reused verbatim in the report name at Step 5.
The trailing `<model>` is what lets two same-change runs coexist.
The clone hardlinks the code repo's objects (fast, ~1 MB extra) and points `origin` at GitHub.
Everything below runs inside `$WORK`, never the developer's checkout.
Substitute `<N>` or `<branch>`, and `<repo>` as `n3qExt` or `nine3q`:

```bash
cd "$WORK"
git fetch origin develop                          # refresh develop FIRST — a stale base skews the diff
git fetch origin refs/pull/<N>/head               # PR review: the PR head → FETCH_HEAD
git fetch origin <branch>                         # branch review: the branch tip → FETCH_HEAD
HEAD=$(git rev-parse FETCH_HEAD)                   # capture it before anything overwrites FETCH_HEAD
BASE=$(git merge-base $HEAD origin/develop)        # = the merge-base GitHub diffs the PR against
git diff --stat $BASE..$HEAD                       # the file count the base guardrail checks
git diff $BASE..$HEAD                              # the reviewed change set
git log --format='%h %s%n%b' $BASE..$HEAD          # the commits — for a branch review, the stated intent
"/c/Program Files/GitHub CLI/gh.exe" pr view <N> --comments -R lupuslabs/<repo>                        # PR review: description + discussion; `url:` → the report's PR link
"/c/Program Files/GitHub CLI/gh.exe" pr view <N> --json headRefName,title,url -R lupuslabs/<repo>      # PR review: branch, title and URL for the report header
git worktree add .wt/<ref> $HEAD                   # full branch checkout — read beyond the diff
```

**`gh` guardrail — name the repository, keep stderr.**
`gh` resolves the repository from the *current directory's* git remote,
so it must either run inside `$WORK` or carry `-R lupuslabs/<repo>`;
from the review workspace it would fail on the weblin root, which has no GitHub remote.
Never discard stderr on a data fetch, and never pipe a response straight into a parser:
capture it, check it, then parse.
If `gh` is missing or not signed in, say so, continue the review from the git data alone,
and record in Coverage that the PR description and discussion were not read.

**Shell variables do not survive between commands.**
`$REVIEW`, `$WORK`, `$BASE` and `$HEAD` exist only inside the one shell invocation that set them
— a later command starts fresh in the *original* working directory with none of them set.
Keep each block above self-contained (re-`cd`, re-derive, or write the literal absolute paths),
and record `$WORK`, `$BASE` and `$HEAD` as literal values in your notes the moment they are computed,
so Step 2's dispatch and Step 6's teardown can quote them verbatim.
The same holds for the start `date +%s` — write the number down;
Step 5 subtracts it and there is no variable left to read it from.

**Base guardrail — the diff must match the PR, or every lens reviews a change the author never made.**
Always diff `$BASE..$HEAD` with `$BASE` the merge-base above.
**Never** diff against a `develop` *tip*
— neither two-dot `git diff origin/develop $HEAD` nor the worktree's local `develop`.
A `develop` that has advanced past the branch point
pulls *its own* later commits into the diff as if the author wrote them:
inflated file/line counts and "out-of-scope features" the change never touched.
**Sanity check before fanning out:** for a PR, the `git diff --stat` file count
must equal the PR's **Files changed** count on GitHub.
If it doesn't match, the base is wrong — stop and fix it, do not review the wrong diff.
For a branch there is no external count; state the file count in Coverage instead.

Capture the diff, the diff stat, and the change's stated intent **once**
— every lens reviews the same change set,
and the intent is what "proportionality" is judged against.
The intent, in order of preference: the quoted sentence on the trigger; the PR description; the commit messages.
Record which source it came from — the report header and the scope lens name it.

**The diff is the starting point, not the boundary.**
Always check the branch out into a worktree under this run's clone (`$WORK/.wt/<ref>`)
so every lens can read the *full files the change touches and the code around them*
— callers, callees, the types and interfaces involved, the existing patterns the change should have followed.
A review judges how the change fits the **existing codebase as it stands on the branch**,
which the hunks alone never show: a deletion's effect on its callers, a new function's real callers,
an abstraction that already existed, a convention the diff breaks unnoticed.
Investigate the branch deeply enough to understand *what was implemented and how it fits* before judging it.

**Read the project's own rules before fanning out.**
The reviewed repo's root `CLAUDE.md` (code style, naming, patterns),
the weblin root `C:/Heiner/weblin/CLAUDE.md` (the documentation gate, the index of architecture docs),
and the architecture docs under `C:/Heiner/weblin/docs/` are the project's standard
— see [`criteria/project.md`](../criteria/project.md).
Hand their absolute paths to every lens.

## Step 2 — Fan out: one context per lens

Run as a Workflow — except the [code-review skill lens](code-review-skill.md),
which a skill invocation loads into the **orchestrating context**
and which therefore runs there, in parallel with the Workflow fan-out
(its own finder/verifier subagents are spawned per its protocol).
Every other selected lens runs in its **own context** (a separate subagent),
handed the shared change + intent **and the branch worktree path**
(`$WORK/.wt/<ref>`, an absolute path),
told to follow its own runbook and loading only the criteria/patterns that lens applies.
**Spell every path out in full in the dispatch prompt**
— the review workspace root, the clone, the worktree, the rule files, and `$BASE`/`$HEAD` as raw SHAs.
A subagent gets a fresh shell in its own working directory: it inherits none of your variables,
so a lens told to diff `$BASE..$HEAD` simply fails.
Each lens is expected to **read beyond the diff**
— open the full touched files and the surrounding code in the worktree
to ground its findings in how the change fits the existing branch, not just the hunks.
Every dispatch prompt also carries the three cross-cutting rules below:
the [origin rule](#every-finding-carries-an-origin--introduced-or-pre-existing),
which classifies each finding *introduced* or *pre-existing* and so decides where it is reported;
the [reporting bar](#finding-nothing-is-a-good-result--report-what-must-or-should-be-fixed),
which decides whether it is reported at all — a lens that finds nothing says so;
and the [assumptions](#assumptions-about-the-operating-environment),
which rule whole failure scenarios out of the world the system runs in.
Hand the lens [`criteria/assumptions.md`](../criteria/assumptions.md) by absolute path
(`C:/Heiner/weblin/review/criteria/assumptions.md`) along with its own criteria,
and say it is to be read before findings are written up
— an assumption applied after the write-up saves no work.
Lenses don't see each other's work; that independence is intentional.
Dispatch the selected lenses — and order their sections in the synthesized report —
in the order listed under [Lenses](#lenses) above:

- [Engineering](engineering.md) — the core criteria + their patterns + the project's own rules;
  itself a multi-agent dimension fan-out.
- [Code-review skill](code-review-skill.md) — the built-in `/code-review` skill's diff-centric protocol
  (removed-behavior audit, cross-file trace, line scan + cleanup/conventions angles) at effort `high`,
  retargeted at the change's diff; runs in the main context, never with `--fix`/`--comment`, never `ultra`.
- [Scope](scope.md) — the solution vs. the stated request
  (runs when an intent, a PR description or commit messages exist; else reports *skipped — no request*).
- [CTO](cto.md) — long-term stewardship: maintainability/extensibility,
  reliability of the extension and the silo in long-running operation, user privacy,
  and whether functionality is guaranteed (by clean engineering or by behavioural proof);
  itself a multi-agent accountability fan-out.
- …each planned lens, once active, in its listed slot
  (Architecture → Security → Adversarial security → Feature regression, ahead of CTO).

## Every finding carries an origin — introduced or pre-existing

**This applies to every lens**, and each lens's dispatch prompt must carry it.
A review exists to judge *this change*,
so each finding says whether the change caused the problem or only touched code where it already existed.

- **introduced** — the diff creates it, or the diff makes existing code newly wrong
  (a caller now passes a value the old code never saw, a path that was dead is now reachable).
  A pre-existing weakness the change **amplifies** — puts on a frequently called path, makes reachable, multiplies —
  is *introduced*, and the finding says what the change did to it.
- **pre-existing** — the same problem is true of the code at the merge-base, and the change only revealed it:
  it touched the file, moved the lines, or made a reviewer read them.
  Reformatting, reindenting or relocating a bad line does not make it this change's.
  The project's own legacy rule applies here:
  `CLAUDE.md` says old forms are converted when a line is touched,
  so legacy left on a line the change edits for its own reasons is *introduced*,
  and legacy on untouched lines is not a finding at all.

**Decide it from the base, not from the hunk.**
The line being *in the diff* proves nothing — a moved line arrives as `+`.
Check the merge-base: `git show $BASE:<path>`, or `git log -L <start>,<end>:<path>` in the worktree,
and compare the behaviour rather than the text.
When it genuinely cannot be settled, mark it **introduced** and say the origin is uncertain
— the change under review is the thing being decided,
so an unresolved finding is put in front of the decision-maker.

**Where each kind is reported** — this is the whole point of the classification:

| Origin | Severity | Where it appears |
|---|---|---|
| introduced | any | Verdict, Summary, Short, Findings, Detail |
| pre-existing | `blocker` / `major` | the same places, each marked **pre-existing**; it may still set the verdict |
| pre-existing | `minor` / `nit` | **[Pre-existing Debt](../reports/README.md#pre-existing-debt) only** — never the verdict, Summary, Short, Findings or Detail |

A pre-existing `minor` or `nit` **must not move the verdict**
and must not be counted among the findings the change has to answer for.
It is also framed differently: not *what to merge instead*, but **debt this review revealed**
— see [`reports/README.md`](../reports/README.md#pre-existing-debt).
Severity is judged on the problem itself, exactly as before; origin only decides where it lands.

## Finding nothing is a good result — report what must or should be fixed

**This applies to every lens**, and each lens's dispatch prompt must carry it.
A lens is not measured by how much it finds,
but by whether what it reports must or should be fixed for the code to be good
— and by whether it missed any of that.

**A review that finds nothing has succeeded.**
"No findings" is the right answer for a change that has no problems,
and it is reported plainly and briefly ("merge — nothing above nit"), not padded out.
The failure this guards against is the opposite one:
a lens reads a clean diff and produces a finding anyway, rather than reporting nothing.
Such findings are true but useless
— a rename nobody needs, a style preference the project never expressed,
an alternative approach of equal merit.
Each one costs the reader attention and the author a reply,
and together they hide the findings that matter.

**The bar — would a good engineer change the code because of this?**

- **Must fix** — the merge decision turns on it:
  it is broken, it will break, or it violates a rule the project actually holds.
- **Should fix** — it does not block the merge, but leaving it makes the code worse to live with,
  and the author would want to know.
- **Neither** — an observation about the code rather than a reason to change it.
  Leave it out.

**This is not a rule against nits.**
The `nit` severity stays, and a nit a good engineer would fix
— a misleading name, a dead branch, a comment that says the opposite of the code —
is a real finding and is reported as one.
What is cut is not the small finding but the *manufactured* one.
Severity says how much a finding matters; this rule decides whether it is a finding at all.

**It lowers the bar for reporting, never for looking.**
Every lens still reads the full files and the surrounding code
as [Step 2](#step-2--fan-out-one-context-per-lens) requires;
a shallow pass that found nothing because it looked at nothing is a failed review, not a clean one.

**It is separate from verification.**
[Step 3](#step-3--verify-adversarial) drops findings that are *wrong*;
this rule drops findings that are *right but not worth reporting*.
A finding has to clear both — surviving refutation does not by itself earn a place in the report.

## Assumptions about the operating environment

**This applies to every lens**, and each lens's dispatch prompt must carry it.
Some findings are sound about the code and wrong about the world:
they describe a clock that jumps backwards or a storage write that reported success and did nothing,
in a system deployed where neither happens.
The facts that rule such scenarios out are written down once,
in [`criteria/assumptions.md`](../criteria/assumptions.md), and every lens takes them as given.

**A finding whose *only* failure path violates a listed assumption is dropped.**
The test is the one that file states: does the problem still happen in a world where the assumption holds?
If it does, the finding stands at unchanged severity
and is simply reworded so it no longer leans on the assumed-away scenario.
An assumption is a switch, never a severity discount, and never a hedge inside a finding.

**A dropped finding leaves no trace.**
It is not reported, not listed as suppressed, not counted in the statistics,
and neither the finding nor the assumption is mentioned in Coverage or anywhere else in the report.
The file exists so the report reads as if the false positive had never been generated.

**The full rules — including the request override and the limits on what may be assumed —
live in [`criteria/assumptions.md`](../criteria/assumptions.md), which is authoritative.**
Two of them matter enough to repeat in the dispatch prompt:
an assumption never applies when the stated request puts the assumed-away case explicitly in scope,
and a finding is kept whenever it takes a strained reading of an entry to make it go away.

**It is separate from both other filters.**
[The reporting bar](#finding-nothing-is-a-good-result--report-what-must-or-should-be-fixed)
drops findings that are right but not worth reporting,
[Step 3](#step-3--verify-adversarial) drops findings that are wrong about the code,
and this drops findings that are wrong about the environment.
It lowers the bar for reporting, never for looking
— a lens still reads the full files and the surrounding code,
and still judges everything the assumptions do not reach.

## A recorded decision clears a finding — `docs/` and the deliberate divergences

The project writes its decisions down:
an architecture document in the weblin root `docs/` for a design decision,
and the two `CLAUDE.md` files for the conventions, including each **deliberate divergence** they name
(no `I` prefix on TypeScript interfaces, no `Async` suffix on C# methods, `logError` emitting `console.warn`).
The root `CLAUDE.md` tells a coding agent to follow the documented architecture instead of improvising;
a review does the same.

**A finding the record covers is cleared** — whether the record is new or already sits in `docs/`.
The decision is taken and documented, so raising it again costs the author a reply and changes nothing.
It is not reported as a finding, in any lens.

**Only the record that actually covers it clears it.**
The record must name this behaviour rather than a neighbouring one, and the code must do what the record says.
Code that contradicts its own `docs/` document or a `CLAUDE.md` rule is a finding, cited at the record's `file:line`,
and so is a document that breaks the conventions `docs/README.md` sets for it.
A `docs/` document the change makes untrue is a finding against the change, not a licence to re-argue the design.

## Step 3 — Verify (adversarial)

Every lens verifies its own non-trivial findings with independent verifier(s) prompted to **refute** them
(default to "refuted" when uncertain).
Findings that don't survive are dropped or downgraded.
This is the guard against confident-but-wrong AI findings — no lens skips it.

**A listed [assumption](#assumptions-about-the-operating-environment) is valid grounds for refutation**,
and the verifiers are the second net for one the lens missed:
a finding whose scenario the environment rules out is refuted outright, not downgraded.
It cuts the other way too — a verifier may not invent an assumption to refute with.
Only what [`criteria/assumptions.md`](../criteria/assumptions.md) actually lists counts,
and "in practice this never happens" is not a refutation.

**One carve-out:** a **test-triviality** finding
(one invoking [`criteria/test.md`](../criteria/test.md) *Useless tests — concrete shapes to flag*)
is **not** subject to the refute-default
— it is verified per that file's *Verifying a "useless test" finding* rule
(confirm the shape; "covered elsewhere" confirms deletion, it does not refute).
Without this, refute-biased verification drops exactly the trivial-test findings
the criterion requires to be reported.

## Step 4 — Synthesize the report

Reconcile the per-lens results into one report,
using the format in [`reports/README.md`](../reports/README.md#format)
(the most severe lens usually sets the overall verdict;
where two lenses flag the same line, merge them).
Order the per-lens **Detail** sections by the lens order under [Lenses](#lenses)
(Engineering → Code-review → Scope → … → CTO).
**Assign every surviving finding its ID here**
— a lens prefix and a per-lens counter running in Detail order,
per [`reports/README.md`](../reports/README.md#finding-ids) —
and carry the same ID into every section the finding reaches,
so Summary, Short, Findings and Detail can be read against each other.
A merged finding gets one ID, owned by the lens earliest in the order above.
**Lens output is raw material, not final wording.**
Summary and Short are rewritten in plain words per
[`reports/README.md`](../reports/README.md#plain-words--summary-and-short);
a term a lens used is not a reason to keep it.
The verdict is **advisory** — the human makes the merge decision.

## Step 5 — Save the report and post it to the PR

Write the report to `reports/` (create the folder if absent),
named per [`reports/README.md`](../reports/README.md#file-name), and surface it to the user.
**Close the clock first** — the header's `Runtime` line needs it:

```bash
date +%s   # run end; Runtime = this minus the Step 1 start, in whole minutes ("1 h 47 min")
```

**Check the wording before posting.**
Read Summary and Short once more against [`criteria/plain-words.md`](../criteria/plain-words.md)
and rewrite every word on that list;
the rule is [`reports/README.md`](../reports/README.md#plain-words--summary-and-short).
A report is posted only after that pass.

Then, for a PR review, **post it to the pull request**, as the body of a comment so GitHub renders it,
per [`reports/README.md`](../reports/README.md#posting-to-the-pull-request).
That is the closing action of the run.
A branch review has nothing to post to; say so.

**Do not commit it.**
The report stays in the weblin root's working tree for the user to inspect and commit when they choose
— say in the summary that it is uncommitted and where it lives.
The procedure for committing it, once they ask, is in [`reports/README.md`](../reports/README.md#committing).

**End the terminal summary with the link.**
The last thing the user sees is where to go next,
so close the summary with the PR link when there is one, else the branch name and the report path:

```
PR 92 · ext: Feature/aw thoughts
https://github.com/lupuslabs/n3qExt/pull/92
```

- **Bare, absolute URLs, one per line** — that is what a terminal turns into a click target.
  Markdown link syntax hides the URL behind a label and is not reliably clickable,
  so do not use it here.
- **The same URL the report header already carries**
  ([`reports/README.md`](../reports/README.md#the-header-list))
  — the `url` field from Step 1's `gh pr view`.
  It is already in hand: do not spend another lookup on it.
- **Shown on every run, including a `throwaway`**
  — the link is how the user reaches the PR, not part of posting.

## Step 6 — Tear down this run's clone

Once the report is saved and surfaced, **always remove this run's clone**
so `work/` holds only in-progress reviews:

```bash
C:/Heiner/weblin/review/bin/review-clone --rm <repo>-<ref>-<slug>-<suffix>-<model>   # same slug used in Step 1
```

`C:/Heiner/weblin/review/bin/review-clone --list` shows any clones still live.
Nothing outside `work/` is modified by a review:
the code repos are only fetched into, and the developer's working trees are never read at all.
