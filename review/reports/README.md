# Review reports

Saved review reports — one Markdown file per review,
written by the closing step of the
[review runbook](../lenses/README.md#step-5--save-the-report-and-post-it-to-the-pr),
which then posts it to the pull request and leaves it uncommitted.
**This file is the single source of truth for a report's name, format,
how it is posted, and how it is committed once you ask**
— see [Posting to the pull request](#posting-to-the-pull-request) and [Committing](#committing) below.
It defines the report's structure;
how the prose inside it is written is [AGENTS.md › Writing style](../AGENTS.md#writing-style),
which applies to reports as much as to the workspace's own docs.

## File name

```
<YYMMDD-HHMM>-<repo>-<ref>-<slug>-<suffix>-<model>.md
```

- `<YYMMDD-HHMM>` — the review's local date and time, short:
  2-digit year, no dots, and hour+minute only (`date +%y%m%d-%H%M` → e.g. `261023-1430`),
  so reports sort chronologically.
  The `<model>` (and, for two same-minute runs, a re-run) keep names distinct.
- `<repo>` — always: `ext` for `github-n3qExt`, `server` for `github-nine3q`.
- `<ref>` — always: `PR-<n>` for a pull request,
  else the branch name with every `/` replaced by `-` (`feature/aw` → `feature-aw`).
- `<slug>` — always: **one lowercase word**
  (letters/digits only, no `-`/`_` — `-` is the field separator) that characterizes the change,
  taken from the PR title, the branch name or the first commit (e.g. `badgetool`).
  It sits after the ref and before the suffix,
  so a report is recognisable at a glance instead of being pure ids.
  Chosen at
  [Step 1 of the runbook](../lenses/README.md#step-1--gather-the-change-and-check-out-the-branch-once-shared-by-every-lens)
  when the change is first read, so it is also part of the run's clone slug.
- `<suffix>` — `all` for the default all-lenses review,
  else the single lens name (`engineering`, `scope`, `cto`, …) for a specialized review.
- `<model>` — always: the short name of the model that ran the review (`opus`, `sonnet`, `fable`, `haiku`),
  the **same token used in the run's clone slug**.
  This is what keeps two parallel same-change reviews' reports apart.

Examples (run on 2026-10-23 at 14:30, on Opus):

| Trigger | File |
|---------|------|
| `review ext 92` | `261023-1430-ext-PR-92-badgetool-all-opus.md` |
| `review ext 92 cto` | `261023-1430-ext-PR-92-badgetool-cto-opus.md` |
| `review server feature/aw` | `261023-1430-server-feature-aw-badgetool-all-opus.md` |
| `review server feature/aw cto` (on Fable) | `261023-1430-server-feature-aw-badgetool-cto-fable.md` |

(`badgetool` is the illustrative slug; a real run derives it from the change's own title.)

## Format

A short **Summary** first — the per-lens one-liners, then the change **statistics** —
then a longer **Short** section, a one-line **Findings** index,
**Pre-existing Debt** (only when there is any), **Detail** per lens ordered by severity, and **Coverage**:

```markdown
# Review — <repo> · <PR <n> | branch> · <lens> · <model>

- Change: [<title of the PR>](<PR web URL>)        ← for a branch: Change: `<branch>` (no PR)
- Intent: <the stated request in one line> (<trigger | PR description | commits>)
- Branch: `<source branch>`
- Runtime: <wall-clock of the whole run, e.g. `1 h 47 min`>
- Verdict: <merge | merge-with-nits | needs-work | reject>

## Summary

- <Lens>: <verdict — the problems, as a one-liner; and nothing more when the lens has nothing in the way of the merge> (<ID>)
- <next lens>: …

**Summary carries only what stands between the change and merge.**
The review exists to answer one question — can this be merged —
so anything that does *not* hinder the merge does not belong here.
It costs the reader attention and answers a question nobody asked.
A per-lens line therefore carries the lens's verdict and its problems, in one line, and nothing else:

- **No recap** of what the change does.
- **No cleared checks, no reassurance** — "grains correctly placed", "all demanded fixes are in".
  What the change *is* and what is *good* about it belong in [Short](#format), never here.
- **An excused check is a cleared check.**
  A check that passes only because the request, the PR description or a `docs/` document excuses it
  — a behaviour explicitly deferred, a documented deviation — is not a problem.
- **No stated absence of a problem** — "no introduced defect", "nothing beyond the above",
  "nothing unique above the reporting bar", "no additional risk".
  Every lens keeps its line, so the reader sees at a glance which lenses ran and how each came out;
  a lens with nothing in the way of the merge writes its verdict, then the em dash, and stops there:
  `- CTO: merge —`.

Where an exemption or a cleared check is worth recording, it is recorded in Detail.

A per-lens line resting on a pre-existing finding carries a `(pre-existing)` suffix —
see [Marking pre-existing findings](#marking-pre-existing-findings-in-summary-and-short).

| Stat | Count |
|-|-|
| Files changed | <n> |
| Code files changed | <n> |
| Code files changed (excluding tests) | <n> |
| Code lines delta (excluding tests and comments) | <n> |
| Scaffolding files changed (non .ts/.cs) | <n> |
| Code files new (excluding tests) | <n> |
| Code files deleted (excluding tests) | <n> |
| Unit test files | <n> |
| Unit test functions/methods | <n> |
| Grain/integration test files | <n> |
| Grain/integration test functions/methods | <n> |

## Short

Cover the same ground — what the change does, what the call turns on, the most severe
finding(s), and any coverage caveat (lenses skipped) so a thin review isn't read as a
clean one — but **format it for the material rather than defaulting to one shape**.
This is also the one place above Detail for the *positive* material Summary excludes:
what the change is, what is sound about it, and conditions that are met all live here. When
these collapse into a few sentences of connected narrative, write a paragraph (not a
single sentence). When they are genuinely distinct points — most often the most severe
findings, which each carry their own `file:line` and consequence — break them into a
**list** so the distinct points read as distinct instead of as one run-on paragraph.
Short bolded lead-ins (e.g. **What it is** / **What's good** / **What turns the call**)
are fine when they help a reader scan. Prose and list may be mixed: a narrative opener
followed by a bulleted list of the decisive findings is often the clearest form.
Whatever the shape, this stays a **short** orientation, not a second Findings index — the
full write-up for each finding still lives in Detail.

**Mark the pre-existing ones.** A finding that reaches Short while being pre-existing
carries a `(pre-existing)` suffix after its lead-in — see
[Marking pre-existing findings](#marking-pre-existing-findings-in-summary-and-short).

**Useless tests — state the count in Short when material.** When a review's
useless/trivial-test findings ([`criteria/test.md`](../criteria/test.md) *Useless tests*)
number **5 or more**, or exceed **5% of the changed test methods**, the Short section must
call the count out explicitly — the test suite has a material triviality problem a reader
should see up front, not one buried among the `nit`s in Findings/Detail. Below that
threshold the individual findings in Findings/Detail suffice; the per-test findings are
always reported regardless (that is required by `criteria/test.md`), this rule only governs
when the *aggregate* is escalated into Short.

## Findings

One line per finding, across all lenses — `<ID>` · `severity` · `file:line` · <what it is
in a sentence>. A scannable index; the full write-up for each lives in Detail below.

A review whose lenses found nothing worth fixing writes one line saying so and keeps the
heading — an empty index is a real result, not a thin review
([the reporting bar](../lenses/README.md#finding-nothing-is-a-good-result--report-what-must-or-should-be-fixed)).

## Pre-existing Debt

Debt this review revealed but that this change did not cause — `minor` and `nit` only, one line
each: `<ID>` · `severity` · `file:line` · <what it is>. Omitted entirely when there is
none.

## Detail

### <Lens> — <one-line per-lens verdict>

Severity-ordered. Each finding opens with its **ID**, then **severity** (blocker /
major / minor / nit), `file:line`, **why it matters**, **what to merge instead**. A
pre-existing blocker/major is marked **pre-existing** and says what the change does to it.

### <next lens…>

## Coverage

Which lenses ran, and which were skipped (planned, or no request); whether the PR
description was read; whether the code-review line scan was sampled; the file count for a
branch review.
```

### Finding IDs

**Every finding carries a short ID, and it is the same ID in every section the finding reaches.**
A reader who meets a problem in Summary
finds its one-line entry in the index and its full write-up in Detail
by searching the report for one token, instead of matching a `file:line` by eye across four sections
— which fails outright when two findings sit in the same file.

**Shape** — a lens prefix followed straight by a number, with **no separator**: `E1`, `CTO2`.
A hyphen would split the ID into two words, so a double-click grabs half of it;
`E1` selects whole, which is how a reader carries it from Summary to the search box.
The prefix is the shortest that stays unambiguous once every lens is active,
so it is **fixed by this table** rather than re-derived per report:

| Lens | Prefix | | Lens | Prefix |
|------|--------|-|------|--------|
| Engineering | `E` | | Security | `SEC` |
| Code-review skill | `CR` | | Adversarial security | `ADV` |
| Scope | `SCP` | | Feature regression | `FR` |
| Architecture | `ARC` | | CTO | `CTO` |

**[Pre-existing Debt](#pre-existing-debt) has no prefix of its own**
— a debt item keeps the ID the lens that raised it gave it,
and is simply listed in that section instead of in Findings.
Where a finding lands is decided by its origin and severity, and that is not a reason to rename it:
a debt line reading `CTO4` says which lens turned it up,
and anything written about it before synthesis still points at the right ID.

- **One counter per lens**, starting at 1: first the lens's findings in **Detail** order
  — which is severity-ordered, so `E1` is the Engineering lens's most severe finding —
  then its debt items, in severity order, on the numbers that follow.
  So a lens's Findings-index numbers run contiguously and its debt takes the tail.
  Numbers are never reserved or skipped:
  a finding dropped at verification simply doesn't exist, and the rest close up.
- **IDs identify findings and debt, nothing else.**
  Anything that gets an ID has a line in the [Findings](#format) index
  or in [Pre-existing Debt](#pre-existing-debt);
  the verdicts, the per-lens summaries,
  the scope lens's delivery and Beyond-request blocks
  and the CTO lens's path map are not findings and carry none.
- **A merged finding gets exactly one ID.**
  Where two lenses flag the same line and synthesis merges them,
  the lens **earliest in [execution order](../lenses/README.md#lenses)** owns the ID;
  the other lens's Detail section does not restate the finding, it points at that ID.
  One defect, one number, one write-up.
- **Assigned at synthesis, and local to one report.**
  IDs are handed out at [Step 4](../lenses/README.md#step-4--synthesize-the-report)
  and fixed for that file's life.
  A re-run of the same change numbers independently,
  so citing an ID across reports means naming the report it came from.
- **Letters and digits only — never a `#` prefix.**
  A finished report is posted as a GitHub comment,
  where `#3` renders as an issue or PR link and `@name` as a mention.
  A bare `E1` is inert on GitHub, in a plain-text reader, and in a search box.

**Where the ID goes**, one convention per section:

- **Summary** — in parentheses after the clause it belongs to: `…so no avatar is rendered (E1).`
  A line resting on several findings lists them: `(E1, CTO2)`.
- **Short** — leading its bolded lead-in:
  `- **E1 · Major — the presence handler drops stanzas that arrive before the room snapshot.**`
  Where Short is prose rather than a list, in parentheses after the clause, as in Summary.
- **Findings** and **Pre-existing Debt** — the first field of the line,
  in backticks like the severity beside it: `` `E1` · `major` · `src/…:88` · … ``.
- **Detail** — the first field of the finding's opening line,
  bolded like the severity beside it: `` **E1** · **major** · `src/…:88` · *introduced* · … ``.

### The header list

Five lines, directly under the `#` heading, always all five and always in this order:

- **Change** — for a PR, the title linked to the pull request;
  the URL is the `url` field `gh pr view <N> --json url` prints,
  e.g. `https://github.com/lupuslabs/n3qExt/pull/92`.
  For a branch review, the branch name in backticks and `(no PR)`.
- **Intent** — the stated request in one line, followed by its source in parentheses:
  `(trigger)`, `(PR description)` or `(commits)`.
  Write `Intent: -` when none could be found; the scope lens is then skipped.
- **Branch** — the source branch, in backticks,
  so the branch under review is on the record and can be checked out from the report.
  For a PR read it from `gh pr view <N> --json headRefName`.
- **Runtime** — how long the whole run took, from the trigger to writing the report,
  in whole minutes: `1 h 47 min`, or `38 min` under an hour.
  The clock starts at the first command of
  [Step 1](../lenses/README.md#step-1--gather-the-change-and-check-out-the-branch-once-shared-by-every-lens),
  which records `date +%s`, and stops when the report is written at Step 5.
  It measures the run, not the reading:
  a review whose lenses ran in parallel shows the wall-clock span, not the sum of the lenses.
- **Verdict** — the overall advisory verdict, `merge` / `merge-with-nits` / `needs-work` / `reject`.
  It sits in the header so the answer to the one question the review exists for is the first thing after the links;
  the Summary below it then carries only the per-lens lines.

### Pre-existing Debt

Every finding is classified *introduced* or *pre-existing* by the lens that raised it
— the rule, and how the two are told apart, lives in
[`lenses/README.md`](../lenses/README.md#every-finding-carries-an-origin--introduced-or-pre-existing).
This section is where the pre-existing `minor`s and `nit`s go, and it is the **only** place they appear.

- **It sits directly below the [Findings](#format) index** and above Detail,
  so a reader meets everything the change has to answer for first and the surrounding debt second
  — and the two one-line lists, which share a shape, sit next to each other.
- **Only pre-existing `minor` and `nit`.**
  Anything *introduced* belongs in Findings/Detail whatever its severity,
  and a pre-existing `blocker` or `major` stays in Findings/Detail too — marked **pre-existing**,
  because a serious problem in code this change touches is still in front of the merge decision.
  Where it also reaches Summary or Short, it carries a `(pre-existing)` suffix there too
  — see [below](#marking-pre-existing-findings-in-summary-and-short).
- **It never moves the verdict**, and nothing in it may appear in Verdict, Summary or Short.
  A clean change that happens to sit next to twenty old nits is still a clean change.
- **Different framing: not "what to merge instead".**
  These are not this change's to fix, so the line says what the debt *is*
  and, where it is cheap to say, what clearing it would take — a follow-up, not a merge condition.
  Never phrase one as a change the author must make before merging.
- **One line each**, in the Findings-index shape so the file reads consistently:
  `<ID>` · `severity` · `file:line` · what it is · (optionally) what clearing it would take.
  Group by lens only when that makes it easier to read; severity order otherwise.
  A debt item keeps the [ID](#finding-ids) its lens gave it
  and is never renumbered on the way into this section
  — the prefix is how a reader sees which lens turned it up.
- **Omit the whole section when it is empty** — a standing empty heading trains readers to skip it.
  Say nothing rather than "None".

```markdown
## Pre-existing Debt

Debt this review revealed; not caused by this change and not a condition of merging it.

- `E7` · `minor` · `ChromeExt/src/background/WebsocketManager.ts:119` · the reconnect delay is a literal in two places; worth a follow-up
- `CTO4` · `nit` · `Items/n3q.Grains/UserGrain.cs:88` · a `#{Id}` prefix in log messages the logger already adds
```

### Marking pre-existing findings in Summary and Short

Only a pre-existing `blocker` or `major` ever reaches Summary or Short
— the pre-existing `minor`s and `nit`s stay in [Pre-existing Debt](#pre-existing-debt)
and appear nowhere else.
When one does reach them it carries a plain **`(pre-existing)`** suffix,
so a reader skimming the top of the report sees immediately
which problems this change caused and which it only sits next to,
instead of opening Detail to find out.

- **In Short**, the suffix follows the item's lead-in, and the [ID](#finding-ids) leads it:
  `- **CTO1 · Major — no test runs the background worker's reconnect path.** (pre-existing)`
- **In Summary**, it follows the per-lens line that rests on the finding:
  `- CTO: needs-work — no test runs the reconnect path (CTO1), so the two reconnect fixes are guaranteed by nothing. (pre-existing)`
  When the line carries introduced findings as well,
  attach the suffix to the clause it belongs to rather than the whole line
  — the verdict itself is never "pre-existing", only a finding is.
- **Unmarked means introduced.** Never write `(introduced)`; the absence of the suffix is the mark.
- **The word alone** — `(pre-existing)`, not `(pre-existing debt)` or `(pre-existing, major)`,
  and not a link.
  Detail carries the full origin explanation, including what the change does to it.

### Plain words — Summary and Short

**Summary and Short are written for the person deciding whether to merge**,
skimming before they open a single file.
Write them the way you would explain the change to a colleague out loud:
short sentences, ordinary words, no inflated wording.

- **Effect first, mechanism second.**
  "Avatars of other users never appear after a tab wakes up" — then, if it adds anything, *why*
  ("the presence handler drops stanzas that arrive before the room snapshot").
  Not the other way round.
- **Keep the precise words, drop the impressive ones.**
  Identifiers, `file:line`, error names and the project's own domain terms stay — they are exact.
  What goes is the padding around them: *load-bearing*, *blast radius*, *surface*, *leverage*,
  *non-trivial*, *orthogonal*, *idiomatic*, *sane defaults*, *cognitive overhead*, *first-class*, *robust*.
  If a plainer word means the same thing, the plainer word is the right one.
- **No hedging or preamble.**
  Not "it is worth noting that this is arguably problematic" — "this breaks X".
  Say it, or leave it out.
- **Call a thing what it is, not what a profession calls it.**
  A term that needs a second question to unpack is jargon, however common it is in testing or software circles.
  Not "the golden-file description" — "the stored description the cached rule is regenerated from".
  Where the project's own name for the thing is plain (`ItemProperties`), use that.
- **Literal, not metaphoric.**
  Code does not drive, pin, kill, or reach.
  "Nothing drives a second room" means "there is no test with two rooms" — write that.
- **Leave out context the reader already has.**
  A review is always of a change on a branch, so "on the branch" adds nothing;
  "no test on the branch drives it" is "there is no test for it".
- **State the positive fact and what is missing, not a double negation.**
  Not "nothing drives a second room" —
  "there is only a single-room test; a test with a second room is missing".
  A sentence the reader has to invert in their head has not been written for them.
- **Say who is affected and when.**
  "Every user on an old extension version loses their backpack items after the server deploys"
  is better than "introduces a deployment-ordering constraint".
- **Detail may stay technical.**
  A reader who gets that far wants the mechanism in full,
  and precision matters more than simplicity there.
  This rule governs **Summary** and **Short** only.

**The words that actually recur.**
A scan of 64 finished reports in the workspace this one was adapted from found about 1,500 cases,
and most of them are one small vocabulary.
These are the ten most frequent, with the plain word to use instead:

| Written | Meant |
|---|---|
| a test *pins* X / X is *pinned by* a test / *unpinned* | X has a test / X is checked by a test / X has no test |
| the suite *stays green* / the test *goes red* | the tests still pass / the test fails |
| the test *drives* the real class | the test runs the real class |
| the websocket *surface* / the error *surfaces* | the websocket interface / the error is shown |
| *load-bearing* | matters / the fix depends on it |
| *fails loudly* / *dies silently* | raises an error / stops with no log line |
| the value *reaches* the user | the value is shown to the user |
| *blast radius* | what else is affected |
| *rides along* / a *rider* | is included / an extra change |
| *hand-rolled* | written by hand / its own copy |

The full list, with every recurring word and its replacement, is [`criteria/plain-words.md`](../criteria/plain-words.md);
it is loaded into every review and read once more before a report is posted.

The **statistics table** is filled by the Engineering lens
(it always runs and owns scope/blast-radius),
per [`lenses/engineering.md`](../lenses/engineering.md#step-1b--count-the-change-fill-the-statistics-table).
The row set above is fixed — keep every row even when a count is `0`.

- Order the per-lens **Detail** sections by lens execution order
  — the order lenses are listed in the [lenses](../lenses/README.md#lenses) table
  (Engineering → Code-review → Scope → … → CTO).
- Where two lenses flag the same line, **merge** the findings — into one entry under one [ID](#finding-ids).
- **Every finding written up in Detail must appear as a line
  in either the Findings index or Pre-existing Debt**
  — the index is the complete inventory, not a selection of highlights.
  A Detail entry with no index line is invisible to a reader who scans the index and trusts it.
  The [IDs](#finding-ids) make the check mechanical — before saving, confirm all four:
  every ID in Detail has exactly one Findings line and every Findings line has its Detail entry;
  every debt ID appears in [Pre-existing Debt](#pre-existing-debt) and nowhere else;
  each lens's numbers run `1..n` across the two lists together, with no gap and no number used twice;
  and every ID cited in Summary or Short is one of the Findings IDs
  — an ID that resolves to a debt line means a pre-existing `minor`/`nit` reached a section it may never appear in.
- Keep nitpicks at `nit` severity so they stay separate from the findings that matter.
- The verdict is **advisory** — the human makes the merge decision.

## Posting to the pull request

The **posted** report is what the PR author actually reads,
so every finished **PR** review posts its report to the pull request it reviewed, **as the body of a comment**
— never as an uploaded file.
This is the closing action of the run
(the runbook's [Step 5](../lenses/README.md#step-5--save-the-report-and-post-it-to-the-pr))
and it happens **automatically, without asking**
— the report is written for the author, and a report nobody reads was not worth running.
GitHub renders a comment body as Markdown,
so the headings, the statistics table, the Findings index and every inline `file:line` come out formatted.
A **branch** review has no pull request; skip this step and say so.

Before posting, read Summary and Short once more against
[`criteria/plain-words.md`](../criteria/plain-words.md) and rewrite every word on that list.
Post the file, verbatim — do not re-summarize, re-order or trim it for the comment.
Run it from the weblin root, substituting the repo (`n3qExt` or `nine3q`), the PR number and the report path:

```bash
cd C:/Heiner/weblin                                                            # the root repo, not the clone
"/c/Program Files/GitHub CLI/gh.exe" pr comment <N> -R lupuslabs/<repo> \
    --body-file review/reports/<YYMMDD-HHMM>-<repo>-PR-<N>…-<model>.md
```

`gh pr comment` prints the comment's URL on success and exits non-zero on failure;
read the URL back into the terminal summary.
If `gh` is missing or not signed in, say the report was **not** posted and leave it at the saved file.

**Known limitation — relative links don't resolve.**
A report links back into this workspace (`../lenses/…`, `../criteria/…`);
in a comment on the *reviewed* repo those paths point at nothing.
Only a handful of links per report are affected, all in Summary and Coverage,
and every `file:line` citation in the findings is plain text and unaffected
— so post as-is by default, and only inline those link texts if the user asks.

Skip the post — and say so — when the trigger carried a **`throwaway`** modifier
([Trigger](../lenses/README.md#trigger)), or when the user asked you not to post.
A plain **`re-run`** posts like any other review;
a second report on the same change is usually the point of running it again.

## Committing

Reports are kept in git as the record of past reviews, but **a review never commits its own report**.
The run writes it, posts it, and leaves it in the working tree of the weblin root repository;
the closing summary says it is uncommitted and names the path,
so the user knows it is waiting for them.
Committing happens in a **later turn, only when the user asks for it**
— the same rule the workspace applies to its own docs ([AGENTS.md](../AGENTS.md#committing)):
they inspect first, then authorize.

When they do ask, stage **only the report(s) they asked for**, by explicit path
— never `git add .` / `git add -A`,
so unrelated working changes and other runs' reports are never swept in.
Run it from the weblin root, not the clone, and never in a code repo:

```bash
cd C:/Heiner/weblin                                                 # the root repo, not the clone
git add review/reports/<YYMMDD-HHMM>-<repo>-<ref>…-<model>.md       # the one report — never `git add .`/`-A`
git commit -m "review: <repo> <PR n | branch> (<slug>, <suffix>, <model>)"
```

The message stays on one line; the root `CLAUDE.md` allows two at most and forbids a `Co-Authored-By` line.

A **`throwaway`** report is not committed either
— but that is not a special case any more, it just never gets asked for.
To keep reports out of git entirely, add `review/reports/` to the root `.gitignore` and drop this section.
