# Scope Review (solution vs. the stated request)

Checks the change **against the request that asked for it** on two fronts: does it **deliver** what was asked — is each requested behaviour actually implemented — and does it go **beyond** the request, shipping things nobody asked for?

This is the requirements-side complement to the engineering review's proportionality lens ([`criteria/engineering.md`](../criteria/engineering.md) §1 *Proportionality*, §3 *Blast radius*).
The engineering review judges diff size against the *problem*; this review judges the solution against the *written request*.

**Where the request comes from.**
weblin has no ticket tracker; the request is, in order of preference:

1. the quoted **intent** on the trigger (`review ext 12 "make badge items droppable on the avatar"`);
2. the **PR description** and its discussion (`gh pr view <N> --comments`);
3. for a branch review without an intent, the **commit messages** (`git log $BASE..$HEAD`).

Record which source was used; the report header's `Intent:` line names it.
If none of the three says what the change is for — an empty PR body and commit messages like "wip" — report *skipped — no request* and stop; this review is meaningless without the request to compare against.
A missing request is itself worth one line in Coverage: the next reader will otherwise re-derive the same question.

**When it runs:** as part of every [default review](README.md) that has a request.
Run it *alone* with `review <repo> <change> scope`.

**Criteria & patterns this lens applies:** [`criteria/engineering.md`](../criteria/engineering.md) §1 *Proportionality* and §3 *Blast radius*, plus the *Gratuitous rewrite / scope creep* [anti-pattern](../criteria/anti-patterns.md) — all from the requirements side.
The *facts-over-declarations* principle below governs how every call is judged.
Output is advisory — the human decides whether out-of-request work is acceptable here or should be split out.

---

## Principle — facts over declarations

**Judge by what the code and the request actually are, never by what the developer *says* about them.**
Developer statements — in the PR description, commit messages, or review comments — are **hints to verify**, not evidence.
This cuts both ways:

- A developer **admitting** scope ("these are follow-ups, separate from this PR") is *not* a finding on its own.
  Check it against the data: is that work genuinely *separable* and *not required* by a requested item (separable commits, no code dependency)?
  Only then does it count — and it counts because the **facts** show it's beyond the request, not because the author confessed.
- A developer **claiming necessity** ("this change is needed for the fix") is likewise not proof.
  If the requested item demonstrably ships without that change, the change is **Beyond request** regardless of the assertion.

So a declaration only ever moves a finding when the code independently confirms it.
The verdict must stand on facts in the diff and the request alone, with the declaration removed.

**One caveat for commit-message requests.**
When the request *is* the commit messages (source 3), the request and the declaration are the same text.
Then this lens can only check the second front — does the change ship *more* than its own commits claim — and says so in its verdict.

---

## Step 0 — Gather both sides

The change (inside this run's clone `$WORK` — see the orchestrator's Step 1; never the developer's checkout):

```bash
cd "$WORK"
git diff --stat <BASE>..<HEAD>                               # the surface area of the solution (literal SHAs from Step 1)
git log --format='%h %s%n%b' <BASE>..<HEAD>                   # the commits
gh pr view <N> --comments -R lupuslabs/<n3qExt|nine3q>        # PR review: the author's *claimed* scope and the discussion
```

The request: the trigger's intent sentence, else the PR body, else the commits — see above.

## Step 1 — Build the request inventory

From the request, extract a flat list of **what was actually asked for**:

- each requested behaviour, as one short observable statement;
- any scope *boundaries* stated ("X is out of scope", "follow-ups separate from this PR").

Keep each item short and atomic.
This is the yardstick; everything in the change is measured against it.

## Step 2 — Is each requested behaviour delivered?

For every inventory item, trace it to the change: which code in the diff makes it true, and — ideally — which test pins it.

- **Flag `major`** when a requested behaviour has **no implementation** in the change — no code path makes it hold.
  Cite the request text and the gap (`file:line` where it should live, or *absent*) and say **what must ship** to meet it.
  A behaviour that is only *partially* met, or met but covered by **no** test, is a `minor` note — not a major.
- **Before filing a gap, look for the explanation** in the PR discussion and commit bodies.
  An explanation that *invalidates* the requested approach (tried, shown not to work) retires the finding: report it as ***explained*** with its source in one sentence.
  An explanation that merely *defers* the work keeps the finding at its normal severity, with the explanation attached.
  No explanation anywhere → file it as-is, and say so.

Judge from the **facts** — the request text, the diff, the tests — never from the author's claim that something is "done".

## Step 3 — Map the solution onto the request

Walk the change's substantive edits (group by feature/subsystem, not by file) and classify each against the inventory:

| Class | Meaning |
|-------|---------|
| **Requested** | Directly implements an inventory item. |
| **Supporting** | Not named in the request, but genuinely required to deliver a requested item (a config key the feature needs, a test for it, the `_Changes.ts`/`Changes.cs` entry, a `docs/` update). Necessary plumbing, not scope creep. |
| **Beyond request** | Real, shippable behaviour the request did not ask for and that isn't required by a requested item. New windows, new grain methods, new config, "while we're here" features. |
| **Contradicts** | Cuts against a stated boundary (does work the request explicitly deferred or excluded). |

Decide each class from the **code**: trace whether a requested item depends on the change (can it ship without it?) and whether the change is separable from the core fix.
The developer's own labelling — "necessary", "separate", "while I was here" — does not decide the class; the dependency in the diff does (see *facts over declarations*).

Be honest about **Supporting** — it is the escape hatch that can excuse anything.
A change is Supporting only if a requested item *cannot* land without it, *and you can show why* from the code.
If the requested item would ship fine without the change, it's **Beyond request** — even if the author called it necessary.

**Legacy conversion on touched lines is Supporting.**
The project's `CLAUDE.md` says legacy code is converted to the current style when touched, so a style conversion of a line the change edits for its own reasons is expected plumbing.
Conversion of lines the request gave no reason to touch is **Beyond request**, and the engineering lens's *Drive-by churn* dimension flags the same lines; merge at synthesis.

**Reuse does not shrink scope.**
A capability the request never asked for is **Beyond request** even when it is built from an *already-established* mechanism — an existing grain, aspect, window base class, or shared helper.
Familiar building blocks make new behaviour read as idiomatic plumbing and mask that it is new surface area.
The test is unchanged and it is about *behaviour*, not *parts*: does a requested item depend on this capability in the code?
If not, reusing an established component to build it does not make it Supporting.

## Step 4 — Verify (adversarial), same as the default review

For every **Beyond request** / **Contradicts** call, a verifier argues the opposite: "this *is* required by requested item X because…", **citing the code dependency**, not a developer statement.
Default to **downgrading to Supporting** when the necessity is genuinely arguable from the diff — the finding that survives is work with no honest line back to the request *in the code*.
A verifier may **not** downgrade on the grounds that the capability reuses an established mechanism; a requested item either depends on the capability or it doesn't.
The verifier must also strip out any finding that rests on a declaration the data doesn't confirm.

The same adversarial pass covers the **delivery** findings from Step 2: for a *not-delivered* flag, a verifier argues the diff/tests *do* satisfy it, **citing the code**, not the author's "done".
A finding that survives stays `major`; one that does not is dropped or downgraded to `minor`.

## Step 5 — Report: the coverage verdict

- **Request source** — trigger / PR description / commits, in one line.
- **Coverage verdict** — one of:
  - *In scope* — everything traces to the request or its necessary support;
  - *Over-scope* — material Beyond-request work rides along (list it);
  - *Diverges* — work contradicts a stated boundary.
- **Delivery verdict** — one of: *Delivered* (every requested behaviour implemented); *Gaps* (**major** — one or more requested behaviours went unimplemented).
  List each `major`: the request text, the gap, and **what must change**.
  Add ***explained*** for each requested behaviour that did not ship but whose absence the discussion accounts for (Step 2): name it, and give the explanation in **one sentence** with its source.
- **Beyond-request inventory** — each item: what it is (`file:line` / subsystem), why it isn't traceable to the request, and the **disposition to recommend**: *split into its own PR*, *acceptable to keep — call it out in the PR description*, or *record it in `_Changes.ts`/`Changes.cs` as its own entry*.
  (Recommend; the human decides.)
- The verdict is **advisory.**
  Going beyond a request is sometimes the right call; this review makes the over-scope *visible and deliberate* instead of silent.
