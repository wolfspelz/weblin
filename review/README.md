# weblin Review Workspace

An AI-assisted code-review setup for the two weblin code repos: `github-n3qExt` (the browser extension, GitHub `lupuslabs/n3qExt`) and `github-nine3q` (the server, GitHub `lupuslabs/nine3q`).
Point it at a pull request or a branch and it runs a panel of independent review **lenses** — engineering quality, request scope, long-term stewardship, and more — then synthesizes them into one report that cites every finding by `file:line`, says **why it matters**, and proposes **what to merge instead**.

**The report is advice, not a decision.**
It is input for a human's merge call; the review never decides whether to merge — the human reviewer does.

This README explains *what the workspace is and how it works*, for a human.
Two companion docs go deeper:

- **[AGENTS.md](AGENTS.md)** — the operating rules an AI agent follows when running a review.
- **[lenses/README.md](lenses/README.md)** — the step-by-step orchestration (fetch the change, fan out to the lenses, verify, synthesize).

## What it's good for

- **A thorough second opinion before a human spends attention on a change** — several reviewers' worth of angles in one pass, each grounded in the actual code on the branch, not just the diff.
- **Catching what a quick skim misses** — scope creep against the stated request, tests that guard nothing, a change's blast radius, a code-style rule from `CLAUDE.md` silently broken, an architecture doc in `docs/` left stale — each owned by a dedicated lens so no concern bleeds into another.
- **A durable, comparable record** — every review is saved as a timestamped report under [`reports/`](reports/), so past reviews stay searchable and two runs of the same change (e.g. under different models) can be compared.

## How it works

### One change → many lenses → one report

A review is a short loop between you and the agent:

1. **You** point the agent at a PR or branch (see [Triggering a review](#triggering-a-review)), optionally adding context — the problem it should solve, anything you already suspect.
2. **The agent** fetches the change once, then runs each applicable [lens](#the-lenses) in its **own** context.
   Lenses don't see each other's work — that independence is the point — and each reads *beyond the diff*, opening the full touched files and surrounding code to judge how the change fits the codebase as it stands.
   Findings are then verified by adversarial passes (to filter confident-but-wrong ones) and synthesized into one report.
3. **You** push back, add context, or ask for a deeper pass on a file or concern; the agent revises.
   Repeat until you're satisfied.
4. **You** make the merge call.

The orchestration mechanics — fan-out, cross-lens synthesis, verification — live in [lenses/README.md](lenses/README.md).

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

Each review runs in its own disposable clone under `work/`, which hardlinks the code repo's object store (fast, ~1 MB extra disk) and is removed when the run finishes, so several reviews can run in parallel without racing on one working tree.
Your checkouts only ever receive a `git fetch`; you can switch branches, rebase or keep uncommitted work in them mid-review with no effect.

To refresh both repos by hand:

```bash
C:/Heiner/weblin/review/bin/review-clone --sync
```

### Reading the change

The code needs no credentials: GitHub exposes every PR as the git ref `refs/pull/<n>/head`, and a branch is fetched by name.
The agent diffs it against the **merge-base with `origin/develop`** (not the `develop` tip), which keeps "size of change" honest by excluding commits that landed on `develop` after the author branched.

Two optional sources need the GitHub CLI, `gh` (`winget install GitHub.cli`, then `gh auth login`):

- **PR metadata** (description, review threads), which the [scope lens](lenses/scope.md) uses as the stated request.
- **Posting the finished report** as a PR comment.

A branch review uses neither: the request comes from the sentence you put on the trigger, or from the commit messages, and the report is only saved.

## Triggering a review

The grammar:

```
review <repo> <PR-number-or-URL-or-branch> [lens] [modifiers] ["<intent>"]
```

- **`<repo>`** is `ext` (github-n3qExt) or `server` (github-nine3q).
  A full PR URL names the repo itself and may replace the token.
- **PR or branch only, or `all`** → the **default review**: every active [lens](#the-lenses) runs and is synthesized into one report.
- **+ a quoted intent** → the stated request the [scope](lenses/scope.md) lens judges against; without it the lens uses the PR description, else the commit messages.
- **+ a single lens name** → run only that lens.
- **+ `throwaway`** (or `temp`, `tmp`, `transient`, `scratch`, `local`, `nosave`, `dontsave`, `nocommit`) → run everything and show you the report, but neither commit it nor post it to the PR.
- **+ `re-run`** (or `rerun`, `redo`, `repeat`, `again`) → the repeat is deliberate, so the agent won't stop to ask whether you meant it; the report is committed and posted as usual.

The recognised words are a convenience, not a syntax to memorise — plain language works just as well ("don't commit this", "yes, do it again").

Examples:

- `review ext 12` — full default review of PR 12 on the extension.
- `review server feature/aw` — full default review of the server branch `feature/aw` against `develop`.
- `review ext 12 engineering` — the engineering lens only.
- `review ext 12 throwaway` — the full review, report shown but not committed and not posted.
- `review server feature/aw "make badge items droppable on the avatar"` — a branch review with the request spelled out for the scope lens.
- `https://github.com/lupuslabs/nine3q/pull/7` works in place of `server 7`.

## The lenses

Each lens is a focused pass run in its own context.
Omitting the lens (or passing `all`) runs every *active* lens and synthesizes them; naming one runs only that lens.
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

Every review is saved as timestamped Markdown under [`reports/`](reports/) — a verdict and per-lens summary, the change statistics, a short narrative, a one-line **Findings** index, then **Detail** per lens ordered by severity.
The file-name convention, the full format, and the two closing steps — commit the report into the weblin root, then post it to the pull request as a comment — are all defined in [`reports/README.md`](reports/README.md).

## The criteria — the standard changes are judged against

Loaded into every review:

- [criteria/engineering.md](criteria/engineering.md) — core engineering criteria.
- [criteria/desired-patterns.md](criteria/desired-patterns.md) — patterns we *want* to see.
- [criteria/anti-patterns.md](criteria/anti-patterns.md) — patterns we reject.
- [criteria/test.md](criteria/test.md) — test criteria and patterns (wanted and rejected).
- [criteria/project.md](criteria/project.md) — a **pointer** to the project's *own* rule files (each repo's `CLAUDE.md`, the weblin root `CLAUDE.md`, the architecture docs in `docs/`), re-encoding nothing.

## Updating the criteria and lenses

The criteria, patterns and lenses are maintained **by hand** — a review ends with its findings and never proposes criteria changes on its own.
When you want to fold a lesson in, ask the agent directly, naming the target:

- a recurring problem → a new entry in [criteria/anti-patterns.md](criteria/anti-patterns.md)
- a good practice worth enforcing → [criteria/desired-patterns.md](criteria/desired-patterns.md)
- a sharper or new judgment rule → [criteria/engineering.md](criteria/engineering.md)
- a test-quality rule or test shape → [criteria/test.md](criteria/test.md)
- a weblin-specific code rule or idiom → **the code repo's own** `CLAUDE.md` (mirrored sections go into both); an architecture fact → a document in `docs/`; [criteria/project.md](criteria/project.md) only defers to those files
- a new lens worth its own pass → a new file under `lenses/`

Changes to any of these workspace docs are staged for a human to inspect and are **not** committed automatically (see [AGENTS.md](AGENTS.md)).
