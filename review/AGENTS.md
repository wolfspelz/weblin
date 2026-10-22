# AGENTS.md — weblin Review Workspace

The **operating rules an agent follows** when reviewing a pull request or branch of the two weblin code repos, `github-n3qExt` (the extension) and `github-nine3q` (the server).
What the workspace is and how it works, for a human: [README.md](README.md).
Step-by-step review orchestration: [lenses/README.md](lenses/README.md).

## Orientation
- **This workspace** is `C:/Heiner/weblin/review` (`criteria/ lenses/ bin/ reports/`).
  It lives inside the weblin root repository and is committed there, next to `docs/` and `CLAUDE.md`.
  The two code repos are separate git repositories under the same root and are gitignored by it.
- **The code repos** `C:/Heiner/weblin/github-n3qExt` and `C:/Heiner/weblin/github-nine3q` are the developer's **own checkouts**.
  A review never checks out, edits, merges or fast-forwards anything in them.
  `bin/review-clone` only fetches into them and then hardlinks a disposable per-run clone into `work/`.
  All `git` commands of a review run **inside that clone (`$WORK`)**.
- **Reach the workspace's own tools by absolute path** — `C:/Heiner/weblin/review/bin/review-clone`, never `./bin/review-clone`.
  A review's working directory is the clone, where `./bin/` does not exist, and shell variables never survive from one command to the next.
- **`gh` resolves the repository from the current directory's git remote.**
  Run it inside `$WORK` or pass `-R lupuslabs/n3qExt` / `-R lupuslabs/nine3q`; from this workspace it would hit the weblin root, which has no GitHub remote.
  `gh` is not installed by default on this machine: `winget install GitHub.cli`, then `gh auth login`.
  Branch reviews need no `gh` at all.
- **Shell is Git Bash on Windows.**
  Write paths with forward slashes (`C:/Heiner/...`); `date +%s`, `find -printf`, and hardlink clones all work.

## The job
- Trigger: **`review <repo> <PR-number-or-URL-or-branch> [lens] [modifiers] ["<intent>"]`** → execute the runbook in [lenses/README.md](lenses/README.md) directly (there is no skill; the trigger authorizes the heavy multi-agent run).
  `<repo>` is `ext` or `server`; a PR URL carries the repo and may replace it.
  No lens or `all` → run **all active lenses** in separate contexts, in the order the runbook lists them; a single lens → only that lens.
- A review is **advisory input for the human merge decision** — never decide merge.
- Every finding cites `file:line`, says **why it matters**, proposes **what to merge instead**.
- Save the report to [`reports/`](reports/); its file name and format are defined in [`reports/README.md`](reports/README.md).

## Always loaded into a review
- [criteria/engineering.md](criteria/engineering.md) · [criteria/desired-patterns.md](criteria/desired-patterns.md) · [criteria/anti-patterns.md](criteria/anti-patterns.md) · [criteria/test.md](criteria/test.md)
- [criteria/project.md](criteria/project.md) — the pointer to the reviewed repo's own `CLAUDE.md`, the weblin root `CLAUDE.md`, and the architecture docs in `C:/Heiner/weblin/docs/`.

## Criteria upkeep
Do **not** propose criteria changes as part of a review — a review ends with its findings.
The criteria files above are edited **only when the user explicitly asks** to add a pattern, anti-pattern, or engineering principle.
Learnings are not tracked.

## Writing markdown in this workspace

**One sentence per line.**
Every markdown file this workspace owns — `AGENTS.md`, `CLAUDE.md`, the READMEs, `lenses/`, `criteria/`, `docs/`, and the reports — puts each sentence on its own line, with no wrapping inside a sentence.
A paragraph is a run of such lines with no blank line between them; blank lines separate paragraphs as usual.
This keeps diffs honest: editing one sentence rewrites one line, instead of reflowing the whole paragraph and burying the change.
The full convention, including semantic line breaks for long sentences, is the weblin root's [docs/README.md](../docs/README.md); it applies here unchanged.

Rendering is unaffected — CommonMark and GitHub join the lines back into a flowing paragraph.

**Inside a list item, split too — and indent the continuation lines** to the item's content column (two spaces after `- `, three after `1. `).
The item then renders as one flowing item, exactly like a paragraph, and nested sub-bullets underneath still work.

```markdown
- **Flag `major`** when the stated request names a behaviour the change does not deliver.
  Cite the request text and the gap.
  Without it, the merge decision has nothing to verify against.
```

**Where not to split — leave the text on one line:**

- **When the next sentence would start the line with a block marker.**
  A line beginning with `-`, `*`, `+`, `#`, `>`, a code fence, or `1.` / `1)` becomes a list item, heading, quote, or code block as soon as it owns a line — inside a list item it becomes a *nested* one.
  Keep such a sentence on the previous sentence's line.
- **Tables** — a row is one line, however long.
- **Code fences and inline code blocks** — the content is literal.
- **YAML frontmatter** and **link reference definitions**.
- **Deliberate hard breaks** (two trailing spaces).

Where a sentence ends is a judgement call, not a regex — `e.g.`, `i.e.`, `v1.2`, and `Config.ts.` all end in a period without ending a sentence.
When the boundary is unclear, leave the sentences together; an over-eager split is worse than an unsplit line.

## Committing

- **Reports are committed when the review that produces them finishes** — into the **weblin root repository** (`C:/Heiner/weblin`), as its own commit, that report's explicit path only, never `git add .`/`git add -A`.
  Full procedure and guardrails: [`reports/README.md`](reports/README.md#committing).
  Commit messages follow the root `CLAUDE.md`: short, at most two lines, no `Co-Authored-By` line.
- **Never commit changes to the workspace's own docs automatically** — instructions (`AGENTS.md`, `CLAUDE.md`), lenses (`lenses/`), READMEs, criteria (`criteria/`), and anything else describing how the workspace operates.
  Make the edit, surface the diff, and **wait for the user to inspect it and explicitly say to commit**.
  A "commit" earlier in a conversation authorizes that one commit, not later edits.
  This rule covers every doc except a finished review's own report output file (the bullet above).
- **`docs/notes.md` — never edit it, always carry it along.**
  The file says so on its own line 2: *"AI: do not edit this file, but commit it with others"*.
  It is the user's own prompt queue, edited by hand between runs.
  Stage it by explicit path next to whatever you are committing (`git add review/reports/<report>.md review/docs/notes.md`) — that is the one addition the "explicit path only" rule above expects, and it keeps the queue from sitting in a permanently dirty working tree.
  If there is nothing else to commit, leave it alone; it rides along, it does not get its own run.
- **Never commit in the code repos.**
  A review reads them through its clone and writes nothing back; the only git that a review commits to is the weblin root.
