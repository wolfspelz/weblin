# weblin Review Workspace

An AI-assisted code-review setup for the two weblin code repos:
`github-n3qExt` (the browser extension, GitHub `lupuslabs/n3qExt`)
and `github-nine3q` (the server, GitHub `lupuslabs/nine3q`).
Point it at a pull request or a branch and it runs a panel of independent review **lenses**
— engineering quality, request scope, long-term stewardship, and more —
then synthesizes them into one report that cites every finding by `file:line`,
says **why it matters**, and proposes **what to merge instead**.

**The report is advice, not a decision.**
It is input for a human's merge call; the review never decides whether to merge — the human reviewer does.

This README explains *what the workspace is and how it works*, for a human.
Two companion docs go deeper:

- **[AGENTS.md](AGENTS.md)** — the operating rules an AI agent follows when running a review.
- **[lenses/README.md](lenses/README.md)** — the step-by-step orchestration
  (fetch the change, fan out to the lenses, verify, synthesize).

## What it's good for

- **A thorough second opinion before a human spends attention on a change**
  — several reviewers' worth of angles in one pass,
  each grounded in the actual code on the branch, not just the diff.
- **Catching what a quick skim misses** — scope creep against the stated request, tests that guard nothing,
  how much else a change affects, a `CLAUDE.md` rule broken without notice,
  an architecture document in `docs/` left stale
  — each owned by a dedicated lens so no concern is mixed into another.
- **A durable, comparable record**
  — every review is saved as a timestamped report under [`reports/`](reports/),
  so past reviews stay searchable
  and two runs of the same change (e.g. under different models) can be compared.

## How it works

### One change → many lenses → one report

A review is a short loop between you and the agent:

1. **You** point the agent at a PR or branch (see [Triggering a review](#triggering-a-review)),
   optionally adding context — the problem it should solve, anything you already suspect.
2. **The agent** fetches the change once, then runs each applicable [lens](#the-lenses) in its **own** context.
   Lenses don't see each other's work, which is intentional,
   and each reads *beyond the diff*, opening the full touched files and surrounding code
   to judge how the change fits the codebase as it stands.
   Findings are then verified by adversarial passes (to filter confident-but-wrong ones)
   and synthesized into one report.
3. **You** push back, add context, or ask for a deeper pass on a file or concern; the agent revises.
   Repeat until you're satisfied.
4. **You** make the merge call on GitHub.

The orchestration mechanics — fan-out, cross-lens synthesis, verification —
live in [lenses/README.md](lenses/README.md).

### It never touches your own checkouts

The workspace sits inside the weblin root next to the two code repos, and uses them only as a hardlink base:

```
C:/Heiner/weblin/            ← the weblin root (its own git: docs, CLAUDE.md, this workspace)
  docs/                      ← architecture docs — the review reads them as project rules
  github-n3qExt/             ← your extension checkout (own git, ignored by the root) — fetched into, never checked out
  github-nine3q/             ← your server checkout (own git, ignored by the root)    — fetched into, never checked out
  review/                    ← this workspace
    README.md  AGENTS.md  criteria/  lenses/  reports/  bin/  docs/
    work/                    ← per-run disposable clones — gitignored
```

Each review runs in its own disposable clone under `work/`,
which hardlinks the code repo's object store (fast, ~1 MB extra disk)
and is removed when the run finishes,
so several reviews can run in parallel without racing on one working tree.
Your checkouts only ever receive a `git fetch`;
you can switch branches, rebase or keep uncommitted work in them mid-review with no effect.

To refresh both repos by hand:

```bash
C:/Heiner/weblin/review/bin/review-clone --sync
```

### Reading the change

The code needs no credentials: GitHub exposes every PR as the git ref `refs/pull/<n>/head`,
and a branch is fetched by name.
The agent diffs it against the **merge-base with `origin/develop`** (not the `develop` tip),
which keeps the "size of change" figure correct by excluding commits that landed on `develop` after the author branched.

Two optional sources need the GitHub CLI, `gh`, installed at `C:/Program Files/GitHub CLI/gh.exe` and signed in:

- **PR metadata** (description, review threads), which the [scope lens](lenses/scope.md) uses as the stated request.
- **Posting the finished report** as a PR comment.

A branch review uses neither:
the request comes from the sentence you put on the trigger, or from the commit messages,
and the report is only saved.

## Triggering a review

The grammar:

```
review <repo> <PR-number-or-URL-or-branch> [lens] [modifiers] ["<intent>"]
```

- **`<repo>`** is `ext` (github-n3qExt) or `server` (github-nine3q).
  A full PR URL names the repo itself and may replace the token.
- **PR or branch only, or `all`** → the **default review**:
  every active [lens](#the-lenses) runs and is synthesized into one report.
- **+ a quoted intent** → the stated request the [scope](lenses/scope.md) lens judges against;
  without it the lens uses the PR description, else the commit messages.
- **+ a single lens name** → run only that lens.
- **+ `throwaway`** (or `temp`, `tmp`, `transient`, `scratch`, `local`, `nosave`, `dontsave`, `nocommit`)
  → run everything and show you the report, but keep it off the PR.
- **+ `re-run`** (or `rerun`, `redo`, `repeat`, `again`) → the repeat is deliberate,
  so the agent won't stop to ask whether you meant it; the report is posted as usual.

The recognised words are a convenience, not a syntax to memorise
— plain language works just as well ("just show me this one", "yes, do it again").

Examples:

- `review ext 92` — full default review of PR 92 on the extension (same as `review ext 92 all`).
- `review server feature/aw` — full default review of the server branch `feature/aw` against `develop`.
- `review ext 92 engineering` — the engineering lens only.
- `review ext 92 throwaway` — the full review, report shown but not posted to the PR.
- `review server feature/aw "make badge items droppable on the avatar"`
  — a branch review with the request spelled out for the scope lens.
- `https://github.com/lupuslabs/nine3q/pull/92` works in place of `server 92`.

## The lenses

Each lens is a focused pass run in its own context.
Omitting the lens (or passing `all`) runs every *active* lens and synthesizes them;
naming one runs only that lens.
Lenses run in the order listed — dispatched, and their report sections ordered, top-to-bottom.
A planned lens joins the default automatically once its status is dropped, slotting into its position.

| Lens | File | Default | Status |
|------|------|---------|--------|
| Engineering (core) | [lenses/engineering.md](lenses/engineering.md) | ✓ | active |
| Code-review skill (built-in) | [lenses/code-review-skill.md](lenses/code-review-skill.md) | ✓ | active |
| Scope (solution vs. stated request) | [lenses/scope.md](lenses/scope.md) | ✓ | active |
| Architecture | [lenses/architecture.md](lenses/architecture.md) | — | planned |
| Security | [lenses/security.md](lenses/security.md) | — | planned |
| Adversarial security | [lenses/security-adversarial.md](lenses/security-adversarial.md) | — | planned |
| Feature regression | [lenses/feature-regression.md](lenses/feature-regression.md) | — | planned |
| CTO (long-term stewardship) | [lenses/cto.md](lenses/cto.md) | ✓ | active |

## Output

Every review is saved as timestamped Markdown under [`reports/`](reports/)
— a verdict and per-lens summary, the change statistics, a short narrative,
a one-line **Findings** index with a short ID per finding (`E1`, `CTO2`),
**Pre-existing Debt** the change did not cause, then **Detail** per lens ordered by severity.
The file-name convention, the full format,
and the closing step — posting the report to the pull request as a rendered comment —
are all defined in [`reports/README.md`](reports/README.md).
The report itself is **left uncommitted**:
it stays in the weblin root's working tree until you look at it and say to commit.

## The criteria — the standard changes are judged against

Loaded into every review:

- [criteria/engineering.md](criteria/engineering.md) — core engineering criteria.
- [criteria/desired-patterns.md](criteria/desired-patterns.md) — patterns we *want* to see.
- [criteria/anti-patterns.md](criteria/anti-patterns.md) — patterns we reject.
- [criteria/test.md](criteria/test.md) — test criteria and patterns (wanted and rejected).
- [criteria/project.md](criteria/project.md) — a **pointer** to the project's *own* rule files
  (each repo's `CLAUDE.md`, the weblin root `CLAUDE.md`, the architecture docs in `docs/`), re-encoding nothing.
- [criteria/assumptions.md](criteria/assumptions.md) — facts about the **operating environment**
  every lens takes as given (the clock is monotonic, a platform call that reports success did its job),
  so findings that describe a world we don't run in are dropped instead of reported.
- [criteria/plain-words.md](criteria/plain-words.md) — the review jargon a report does not use,
  each with the plain word to write instead.

## Updating the criteria and lenses

The criteria, patterns and lenses are maintained **by hand**
— a review ends with its findings and never proposes criteria changes on its own.
To add a lesson, ask the agent directly, naming the target:

- a recurring problem → a new entry in [criteria/anti-patterns.md](criteria/anti-patterns.md)
- a good practice worth enforcing → [criteria/desired-patterns.md](criteria/desired-patterns.md)
- a sharper or new judgment rule → [criteria/engineering.md](criteria/engineering.md)
- a test-quality rule or test shape → [criteria/test.md](criteria/test.md)
- a false positive that was right about the code and wrong about the world
  → a new entry in [criteria/assumptions.md](criteria/assumptions.md)
- a weblin-specific code rule or idiom → **the code repo's own** `CLAUDE.md`
  (mirrored sections go into both repos); an architecture fact → a document in `docs/`;
  [criteria/project.md](criteria/project.md) only defers to those files
- a new lens worth its own pass → a new file under `lenses/`

Nothing here is committed automatically
— neither a change to these workspace docs nor a finished review's report.
Everything is left for a human to inspect and commit (see [AGENTS.md](AGENTS.md)).
