# Review reports

Saved review reports — one Markdown file per review, written by the closing step of the [review runbook](../lenses/README.md#step-5--save-the-report-commit-it-and-post-it-to-the-pr), which then commits it and, for a pull request, posts it as a PR comment.
**This file is the single source of truth for a report's name, format, how it is committed, and how it is posted** — see [Committing](#committing) and [Posting to the pull request](#posting-to-the-pull-request) below.

## File name

```
<YYMMDD-HHMM>-<repo>-<ref>-<slug>-<suffix>-<model>.md
```

- `<YYMMDD-HHMM>` — the review's local date and time, short: 2-digit year, no dots, and hour+minute only (`date +%y%m%d-%H%M` → e.g. `261023-1430`), so reports sort chronologically.
  The `<model>` (and, for two same-minute runs, a re-run) keep names distinct.
- `<repo>` — always: `ext` for `github-n3qExt`, `server` for `github-nine3q`.
- `<ref>` — always: `PR-<n>` for a pull request, else the branch name with every `/` replaced by `-` (`feature/aw` → `feature-aw`).
- `<slug>` — always: **one lowercase word** (letters/digits only, no `-`/`_` — `-` is the field separator) that characterizes the change, taken from the PR title, the branch name or the first commit (e.g. `badgetool`).
  It sits after the ref and before the suffix, so a report is recognisable at a glance instead of being pure ids.
  Chosen at [Step 1 of the runbook](../lenses/README.md#step-1--gather-the-change-and-check-out-the-branch-once-shared-by-every-lens) when the change is first read, so it is also part of the run's clone slug.
- `<suffix>` — `all` for the default all-lenses review, else the single lens name (`engineering`, `scope`, `cto`, …) for a specialized review.
- `<model>` — always: the short name of the model that ran the review (`opus`, `sonnet`, `fable`, `haiku`), the **same token used in the run's clone slug**.
  This is what keeps two parallel same-change reviews' reports apart.

Examples (run on 2026-10-23 at 14:30, on Opus):

| Trigger | File |
|---------|------|
| `review ext 12` | `261023-1430-ext-PR-12-badgetool-all-opus.md` |
| `review ext 12 cto` | `261023-1430-ext-PR-12-badgetool-cto-opus.md` |
| `review server feature/aw` | `261023-1430-server-feature-aw-badgetool-all-opus.md` |
| `review server feature/aw cto` (on Fable) | `261023-1430-server-feature-aw-badgetool-cto-fable.md` |

(`badgetool` is the illustrative slug; a real run derives it from the change's own title.)

## Format

A short **Summary** first — the verdict and a per-lens one-liner, then the change **statistics** — then a longer **Short** section, a one-line **Findings** index, **Detail** per lens ordered by severity, and **Coverage**:

```markdown
# Review — <repo> · <PR <n> | branch> · <lens> · <model>

- Change: [<title of the PR, or the branch name>](<PR web URL>)   ← for a branch: `Change: `<branch>` (no PR)`
- Intent: <the stated request in one line, and its source: trigger / PR description / commits>
- Branch: `<source branch>`
- Runtime: <wall-clock of the whole run, e.g. `1 h 47 min`>

## Summary

Verdict: <merge | merge-with-nits | needs-work | reject>
- <Lens>: <one-line per-lens verdict>
- <next lens>: …

| Stat | Count |
|-|-|
| Files changed | <n> |
| Code files changed | <n> |
| Code files changed (excluding tests) | <n> |
| Code lines delta (excluding tests) | <n> |
| Scaffolding files changed (non .ts/.cs) | <n> |
| Code files new (excluding tests) | <n> |
| Code files deleted (excluding tests) | <n> |
| Unit test files | <n> |
| Unit test functions/methods | <n> |
| Grain/integration test files | <n> |
| Grain/integration test functions/methods | <n> |
| Docs in `docs/` touched | <n> |

## Short

Cover the same ground — what the change does, what the call turns on, the most severe
finding(s), and any coverage caveat (lenses skipped) so a thin review isn't read as a
clean one — but **format it for the material rather than defaulting to one shape**. When
these collapse into a few sentences of connected narrative, write a paragraph (not a
single sentence). When they are genuinely distinct points — most often the most severe
findings, which each carry their own `file:line` and consequence — break them into a
**list** so the distinct points read as distinct instead of as one run-on paragraph.
Short bolded lead-ins (e.g. **What it is** / **What's good** / **What turns the call**)
are fine when they help a reader scan. Prose and list may be mixed: a narrative opener
followed by a bulleted list of the load-bearing findings is often the clearest form.
Whatever the shape, this stays a **short** orientation, not a second Findings index — the
full write-up for each finding still lives in Detail.

**Useless tests — surface the count in Short when material.** When a review's
useless/trivial-test findings ([`criteria/test.md`](../criteria/test.md) *Useless tests*)
number **5 or more**, or exceed **5% of the changed test methods**, the Short section must
call the count out explicitly — the test suite has a material triviality problem a reader
should see up front, not one buried among the `nit`s in Findings/Detail. Below that
threshold the individual findings in Findings/Detail suffice; the per-test findings are
always reported regardless (that is required by `criteria/test.md`), this rule only governs
when the *aggregate* is escalated into Short.

## Findings

One line per finding, across all lenses — `severity` · `file:line` · <what it is in a
sentence>. A scannable index; the full write-up for each lives in Detail below.

## Detail

### <Lens> — <one-line per-lens verdict>

Severity-ordered. Each finding: **severity** (blocker / major / minor / nit),
`file:line`, **why it matters**, **what to merge instead**.

### <next lens…>

## Coverage

Which lenses ran, and which were skipped (planned, or no request); whether the PR
description was read; whether the code-review line scan was sampled.
```

### The header list

Four lines, directly under the `#` heading, always all four and always in this order:

- **Change** — for a PR, the title linked to the pull request; the URL is the `url:` field `gh pr view <N>` prints, e.g. `https://github.com/lupuslabs/n3qExt/pull/12`.
  For a branch review, the branch name in backticks and `(no PR)`.
- **Intent** — the stated request in one line, followed by its source in parentheses: `(trigger)`, `(PR description)` or `(commits)`.
  Write `Intent: -` when none could be found; the scope lens is then skipped.
- **Branch** — the source branch, in backticks, so the branch under review is on the record and can be checked out from the report.
  For a PR read it from `gh pr view <N> --json headRefName`.
- **Runtime** — how long the whole run took, from the trigger to writing the report, in whole minutes: `1 h 47 min`, or `38 min` under an hour.
  The clock starts at the first command of [Step 1](../lenses/README.md#step-1--gather-the-change-and-check-out-the-branch-once-shared-by-every-lens), which records `date +%s`, and stops when the report is written at Step 5.
  It measures the run, not the reading: a review whose lenses ran in parallel shows the wall-clock span, not the sum of the lenses.

### Plain words — Summary and Short

**Summary and Short are written for the person deciding whether to merge**, skimming before they open a single file.
Write them the way you would explain the change to a colleague out loud: short sentences, ordinary words, no résumé voice.

- **Effect first, mechanism second.**
  "Avatars of other users never appear after a tab wakes up" — then, if it adds anything, *why* ("the presence handler drops stanzas that arrive before the room snapshot").
  Not the other way round.
- **Keep the precise words, drop the impressive ones.**
  Identifiers, `file:line`, error names and the project's own domain terms stay — they are exact.
  What goes is the padding around them: *load-bearing*, *blast radius*, *surface*, *leverage*, *non-trivial*, *orthogonal*, *idiomatic*, *sane defaults*, *cognitive overhead*, *first-class*, *robust*.
  If a plainer word means the same thing, the plainer word is the right one.
- **No hedging or throat-clearing.**
  Not "it is worth noting that this is arguably problematic" — "this breaks X".
  Say it, or leave it out.
- **Say who is hurt and when.**
  "Every user on an old extension version loses their backpack items after the server deploys" beats "introduces a deployment-ordering constraint".
- **Detail may stay technical.**
  A reader who gets that far wants the mechanism in full, and precision beats simplicity there.
  This rule governs **Summary** and **Short** only.

The **statistics table** is filled by the Engineering lens (it always runs and owns scope/blast-radius), per [`lenses/engineering.md`](../lenses/engineering.md#step-1b--count-the-change-fill-the-statistics-table).
The row set above is fixed — keep every row even when a count is `0`.

- Order the per-lens **Detail** sections by lens execution order — the order lenses are listed in the [lenses](../lenses/README.md#lenses) table (Engineering → Code-review → Scope → … → CTO).
- Where two lenses flag the same line, **merge** the findings.
- Keep nitpicks at `nit` severity so they don't drown the signal.
- The verdict is **advisory** — the human makes the merge decision.

## Committing

The report is the durable record of the review, so **committing it is the closing step of the run** (the runbook's [Step 5](../lenses/README.md#step-5--save-the-report-commit-it-and-post-it-to-the-pr)): each finished review commits its **own** report as its own commit.
Reports are committed into the **weblin root repository** (`C:/Heiner/weblin`), which owns this workspace; the code repos are never committed to.
To keep reports transient instead, add `review/reports/` to the root `.gitignore` and drop this step.

Stage **only the report you just wrote**, by its explicit path — never `git add .` / `git add -A`, so unrelated working changes and other runs' reports are never swept in, and never touch a report from another run.
Run it from the weblin root, not the clone:

```bash
cd C:/Heiner/weblin                                                 # the root repo, not the clone
git add review/reports/<YYMMDD-HHMM>-<repo>-<ref>…-<model>.md       # the one report — never `git add .`/`-A`
git commit -m "review: <repo> <PR n | branch> (<slug>, <suffix>, <model>)"
```

The message stays on one line; the root `CLAUDE.md` allows two at most and forbids a `Co-Authored-By` line.

Skip this — and say so — when the trigger carried a **`throwaway`** modifier ([Trigger](../lenses/README.md#trigger)), or when the user asked you not to commit.
A plain **`re-run`** is not a throwaway: it commits like any other review.

## Posting to the pull request

A committed report is the record; the **posted** report is what the author actually reads.
So every finished **PR** review also posts its report to the pull request, **as the body of a comment** — never as an uploaded file.
GitHub renders a comment body as Markdown, so the headings, the statistics table, the Findings index and every inline `file:line` come out formatted.
A **branch** review has no pull request; skip this step and say so.

Post **after** the commit, so what the author reads is exactly what is in git.
Post the file, verbatim — do not re-summarize, re-order or trim it for the comment.
Run it from the weblin root, substituting the repo, the PR number and the report path:

```bash
cd C:/Heiner/weblin                                                            # the root repo, not the clone
gh pr comment <N> -R lupuslabs/<n3qExt|nine3q> --body-file review/reports/<YYMMDD-HHMM>-<repo>-PR-<N>…-<model>.md
```

`gh pr comment` prints the comment's URL on success and exits non-zero on failure; read the URL back into the terminal summary.
If `gh` is missing or unauthenticated, say the report was **not** posted and leave it at the commit.

**Known limitation — relative links don't resolve.**
A report links back into this workspace (`../lenses/…`, `../criteria/…`); in a comment on the *reviewed* repo those paths point at nothing.
Only a handful of links per report are affected, all in Summary and Coverage, and every `file:line` citation in the findings is plain text and unaffected — so post as-is by default, and only inline those link texts if the user asks.

Skip the post in exactly the cases the [commit](#committing) is skipped — a **`throwaway`** on the trigger, or the user asking you not to — and say so.
The two travel together: a report that is not worth committing is not worth putting in front of the PR author, and a report worth committing belongs on the PR.
A plain **`re-run`** posts like any other review; a second report on the same change is usually the point of running it again.
