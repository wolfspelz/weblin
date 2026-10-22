# Project Criteria — weblin (`github-n3qExt`, `github-nine3q`)

The project-specific lens for the [engineering review](../lenses/engineering.md).
**It encodes no rules of its own.**
The criteria are the **project's own rule files**, read fresh at review time and applied to the changed code.
This file only says *where those rules live* and *how to apply them in a review* — so when a rule changes in the project, there is nothing to update here.

---

## Principle — defer to the project's own rules, never copy them

The project carries its house rules in-tree, and **those files are the single source of truth.**
This lens reads and applies them directly.
Do **not** restate, summarise, or pin any specific rule in this file: a copy drifts from the original the moment the project edits it, and the review would then judge against a stale standard.
If you feel the urge to write "the project requires X" here, link to the rule file instead.
The only reason to edit this file is to change *how* rules are discovered or applied — never to record a rule.

## Step 1 — Discover every rule source

The rules live in **three places**, two of them outside the reviewed repo:

| Source | Path | Holds |
|--------|------|-------|
| The reviewed repo's root `CLAUDE.md` | `$WORK/.wt/<ref>/CLAUDE.md` (the branch worktree, so the rules as they apply to *this* change) | architecture overview, build commands, **General / TypeScript / C# Code Style**, changelog rule, repo-specific hazards (timers in background code, notification ordering, grain type placement) |
| The weblin root `CLAUDE.md` | `C:/Heiner/weblin/CLAUDE.md` | the **documentation gate** (no undocumented area is edited), commit rules, project structure, the index of architecture docs |
| The architecture docs | `C:/Heiner/weblin/docs/*.md` | the design of record per area; `docs/README.md` holds the writing conventions |

```bash
cat "$WORK/.wt/<ref>/CLAUDE.md"
cat C:/Heiner/weblin/CLAUDE.md
ls C:/Heiner/weblin/docs/
```

Read both `CLAUDE.md` files in full, then every `docs/` document whose area the change touches — decide that from the document titles and headings against the changed paths, and open any you are unsure about.
Treat all of them as the criteria.

**The code-style sections are mirrored.**
The *General Code Style* and *TypeScript Code Style* sections are, by their own statement, identical in both repos' `CLAUDE.md`.
Read the one in the reviewed repo; if the change edits such a section, also read the other repo's copy (`C:/Heiner/weblin/github-n3qExt/CLAUDE.md` or `C:/Heiner/weblin/github-nine3q/CLAUDE.md` — read-only) to check the mirror.

## Step 2 — Apply them to the change

Check the changed code against the discovered rules.
A rule violation **introduced or touched by this change** is a finding; a pre-existing violation in code the change doesn't touch is not this change's job (unless the change *is* the cleanup).
The `CLAUDE.md` files mark some old forms as *legacy*: legacy on untouched lines is not a finding, legacy left on a line the change edits is.
Each finding cites the rule by its **file and heading in the project** (e.g. `n3qExt/CLAUDE.md › TypeScript Code Style › Naming`), the `file:line` in the diff, why it matters, and what to merge instead — the same shape as every other lens.

**Three project-level rules are checked as rules, not as style:**

- **The documentation gate** (root `CLAUDE.md`, *Hard Rule*): every edited area must be covered by a document in `docs/`.
  The review asks the same question the rule asks before coding — which document covers the touched area? — and files a finding when none does, or when the change makes the covering document untrue.
  Severity follows the rule's own weight: `major` for an undocumented area, `minor` for a stale sentence in an existing document.
  The docs live in the weblin root repo, not in the reviewed repo, so a change cannot ship its doc update in the same diff; the finding then says what the doc must now say.
- **The mirror rule** (both `CLAUDE.md`, *General Code Style*): an edit to a mirrored section lands in both repos.
- **The changelog rule** (`n3qExt/CLAUDE.md › Changelog`, `nine3q/CLAUDE.md › Feature Change Log`): a user-visible change is recorded in `ChromeExt/src/lib/_Changes.ts` or `Items/n3q.Common/Changes.cs`.

## Step 3 — Report drift, don't fix it here

If a project rule looks wrong, outdated, or self-contradictory — or the two mirrored copies have already drifted apart — that is an observation for the human; note it.
Do **not** silently override it, and do **not** "correct" it by encoding a different rule in this file.
The project's rule files remain authoritative.

---

> This file is a **pointer**, not a rule store: the live criteria are the two `CLAUDE.md` files and the `docs/` documents.
> To add or change a weblin rule, edit those files **in the project** (mirrored sections in both repos), not here.
> Maintained by hand — see [Updating the criteria](../README.md#updating-the-criteria-and-lenses).
