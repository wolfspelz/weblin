# Scope Review (solution vs. the stated request)

Checks the change **against the request that asked for it** in two ways:
does it **deliver** what was asked — is each requested behaviour actually implemented —
and does the change go **beyond** the request, shipping things nobody asked for?

This is the requirements-side complement to the engineering review's proportionality lens
([`criteria/engineering.md`](../criteria/engineering.md) §1 *Proportionality*, §3 *Blast radius*).
The engineering review judges diff size against the *problem*;
this review judges the solution against the *written request*.

**Where the request comes from.**
weblin has no ticket tracker; the request is, in order of preference:

1. the quoted **intent** on the trigger (`review ext 92 "make badge items droppable on the avatar"`);
2. the **PR description** and its discussion (`gh pr view <N> --comments`);
3. for a branch review without an intent, the **commit messages** (`git log $BASE..$HEAD`).

Record which source was used; the report header's `Intent:` line names it.
If none of the three says what the change is for — an empty PR body and commit messages like "wip" —
report *skipped — no request* and stop; this review is meaningless without the request to compare against.
A missing request is itself worth one line in Coverage: the next reader will otherwise re-derive the same question.

**When it runs:** as part of every [default review](README.md) that has a request.
Run it *alone* with `review <repo> <change> scope`.

**Criteria & patterns this lens applies:**
[`criteria/engineering.md`](../criteria/engineering.md) §1 *Proportionality* and §3 *Blast radius*,
plus the *Gratuitous rewrite / scope creep* [anti-pattern](../criteria/anti-patterns.md)
— all from the requirements side.
The *facts-over-declarations* principle below governs how every judgement is made.
Delivery is judged by the inline standard in Step 2 — it can yield `major` findings.
Output is advisory — the human decides whether out-of-request work is acceptable here or should be split out.

---

## Principle — facts over declarations

**Judge by what the code and the request actually are, never by what the developer *says* about them.**
Developer statements — in the PR description, commit messages, or review comments —
are **hints to verify**, not evidence.
This applies in both directions:

- A developer **admitting** scope ("these are follow-ups, separate from this PR") is *not* a finding on its own.
  Such an admission may be unintended or imprecise, and is not held against the author.
  Check it against the data: is that work genuinely *separable* and *not required* by a requested item
  (separable commits, no code dependency)?
  Only then does it count
  — and it counts because the **facts** show it's beyond the request, not because the author said so.
  If the facts don't back the admission, drop it.
- A developer **claiming necessity** ("this change is needed for the fix") is likewise not proof.
  If the requested item demonstrably ships without that change,
  the claim is not true and does not hold
  — the change is **Beyond request** regardless of the assertion.

So a declaration only ever moves a finding when the code independently confirms it.
The verdict must stand on facts in the diff and the request alone, with the declaration removed.

**One caveat for commit-message requests.**
When the request *is* the commit messages (source 3), the request and the declaration are the same text.
Then this lens can only check the second front — does the change ship *more* than its own commits claim —
and says so in its verdict.

---

## Step 0 — Gather both sides

The change (inside this run's clone `$WORK` — see the orchestrator's Step 1; never the developer's checkout.
Substitute the literal SHAs from Step 1):

```bash
cd "$WORK"
git diff --stat <BASE>..<HEAD>                                                     # the size of the solution
git log --format='%h %s%n%b' <BASE>..<HEAD>                                         # the commits
"/c/Program Files/GitHub CLI/gh.exe" pr view <N> --comments -R lupuslabs/<repo>     # PR review: the author's *claimed* scope and the discussion
```

The request: the trigger's intent sentence, else the PR body, else the commits — see above.

**Read the discussion carefully**, but as *hints, not verdicts*.
It often records an apparent scope decision
— e.g. "the core fix ships here; the other items are *follow-ups, separate from this PR*".
Treat that as a **lead to check against the code**, not a fact.
See "Facts over declarations" above.

## Step 1 — Build the request inventory

From the request, extract a flat list of **what was actually asked for**:

- each requested behaviour, as one short observable statement;
- any acceptance criteria the PR description states, where it states them;
- any scope *boundaries* stated ("X is out of scope", "follow-ups separate from this PR").

Keep each item short and atomic.
This list is the reference; everything in the change is measured against it.

## Step 2 — Is each requested behaviour delivered?

For every inventory item, trace it to the change:
which code in the diff makes it true, and — ideally — which test covers it.

- **Flag `major`** when a requested behaviour has **no implementation** in the change
  — no code path makes it hold.
  Cite the request text and the gap (`file:line` where it should live, or *absent*)
  and say **what must ship** to meet it.
  A behaviour that is only *partially* met, or met but covered by **no** test, is a `minor` note — not a major.
- **A request with no testable statement is not a finding by itself.**
  weblin's requests are a sentence on the trigger or a PR body, not a story with acceptance criteria;
  where the request is too vague to trace ("improve the backpack"), say so in the verdict
  and judge the delivery against what the PR description and commits claim instead.
- **Before filing a gap, look for the explanation** in the PR discussion and commit bodies.
  An explanation that *invalidates* the requested approach (tried, shown not to work) removes the finding:
  report it as ***explained*** with its source in one sentence.
  An explanation that merely *defers* the work keeps the finding at its normal severity,
  with the explanation attached.
  No explanation anywhere → file it as-is, and say so.

Judge from the **facts** — the request text, the diff, the tests —
never from the author's claim that something is "done" (see *facts over declarations*).

## Step 3 — Map the solution onto the request

Walk the change's substantive edits (group by feature/subsystem, not by file)
and classify each against the inventory:

| Class | Meaning |
|-------|---------|
| **Requested** | Directly implements an inventory item. |
| **Supporting** | Not named in the request, but genuinely required to deliver a requested item (a config key the feature needs, a test for it, the `_Changes.ts`/`Changes.cs` entry). Necessary supporting work, not scope creep. |
| **Beyond request** | Real, shippable behaviour the request did not ask for and that isn't required by a requested item. New windows, new grain methods, new config, "while we're here" features. |
| **Contradicts** | Cuts against a stated boundary (does work the request explicitly deferred or excluded). |

Decide each class from the **code**:
trace whether a requested item depends on the change (can it ship without it?)
and whether the change is separable from the core fix.
The developer's own labelling — "necessary", "separate", "while I was here" — does not decide the class;
the dependency in the diff does (see *facts over declarations*).

Be strict about **Supporting** — this class can be used to excuse anything.
A change is Supporting only if a requested item *cannot* land without it,
*and you can show why* from the code.
If the requested item would ship fine without the change, it's **Beyond request**
— even if the author called it necessary.

**Legacy conversion on touched lines is Supporting.**
The project's `CLAUDE.md` says legacy code is converted to the current style when touched,
so a style conversion of a line the change edits for its own reasons is expected supporting work.
Conversion of lines the request gave no reason to touch is **Beyond request**,
and the engineering lens's *Drive-by churn* dimension flags the same lines; merge at synthesis.

**Reuse does not shrink scope.**
A capability the request never asked for is **Beyond request**
even when it is built from an *already-established* mechanism
— an existing grain, aspect, window base class, or shared helper.
Familiar building blocks make new behaviour look like routine supporting work
and hide that it is new functionality.
The test is unchanged and it is about *behaviour*, not *parts*:
does a requested item depend on this capability in the code?
If not, reusing an established component to build it does not make it Supporting
— the capability is still unrequested, and the reuse only hides that.
Judge what the change *does*, not what it is *made of*.

## Step 4 — Verify (adversarial), same as the default review

For every **Beyond request** / **Contradicts** call, a verifier argues the opposite:
"this *is* required by requested item X because…",
**citing the code dependency**, not a developer statement.
Default to **downgrading to Supporting** when the necessity is genuinely arguable from the diff
— the finding that survives is work with no traceable connection to the request *in the code*.
A verifier may **not** downgrade on the grounds
that the capability reuses an established mechanism ("it's just the existing grain"):
a requested item either depends on the capability or it doesn't, and reuse doesn't create that dependency
— hold the **Beyond request** call unless a code dependency is shown.
The verifier must also strip out any finding that rests on a declaration the data doesn't confirm
(an "admission" of separateness that the commits don't actually show as separable).
This keeps the review from calling necessary supporting work scope creep,
and from reacting to the author's wording instead of the code.

The same adversarial pass covers the **delivery** findings from Step 2:
for a *not-delivered* flag, a verifier argues the diff/tests *do* satisfy it,
**citing the code**, not the author's "done".
A finding that survives stays `major`; one that does not is dropped or downgraded to `minor`.

## Step 5 — Report: the coverage verdict

- **Request source** — trigger / PR description / commits, in one line.
- **Coverage verdict** — one of:
  - *In scope* — everything traces to the request or its necessary support;
  - *Over-scope* — the change carries material Beyond-request work (list it);
  - *Diverges* — work contradicts a stated boundary.
- **Delivery verdict** — one of: *Delivered* (every requested behaviour implemented);
  *Gaps* (**major** — one or more requested behaviours went unimplemented).
  List each `major`: the request text, the gap, and **what must change**.
  These majors hold even when the coverage verdict is *In scope*
  — a change can be clean on scope yet still fail to deliver the request.
  Add ***explained*** for each requested behaviour that did not ship
  but whose absence the discussion or the commits account for (Step 2):
  name it, and give the explanation in **one sentence** with its source.
  This belongs to the lens's Detail section only —
  an explained gap is not a problem and never reaches Summary
  ([an excused check is a cleared check](../reports/README.md#format)).
- **Beyond-request inventory** — each item: what it is (`file:line` / subsystem),
  why it isn't traceable to the request, and the **disposition to recommend**:
  *split into its own PR*, *acceptable to keep — call it out in the PR description*,
  or *record it in `_Changes.ts`/`Changes.cs` as its own entry*.
  (Recommend; the human decides.)
- **Origin on every finding** — *introduced* or *pre-existing*,
  per [`README.md`](README.md#every-finding-carries-an-origin--introduced-or-pre-existing).
  Almost everything this lens raises is *introduced* by construction:
  a delivery gap, a Beyond-request item and a divergence
  are all statements about **this** change's delivery against **this** request.
  The exception is code the change merely touches while delivering
  — a pre-existing `minor`/`nit` noticed on the way is **debt**,
  and it never weakens the coverage verdict.
- **The reporting bar** — only what must or should be fixed,
  per [`README.md`](README.md#finding-nothing-is-a-good-result--report-what-must-or-should-be-fixed).
  It applies most to the Beyond-request inventory: work that is traceable to the request is not a finding,
  and a change that delivers its request and nothing else earns *In scope* with an empty inventory.
- The verdict is **advisory.**
  Going beyond a request is sometimes the right call;
  this review makes the over-scope *visible and deliberate* instead of unstated.
