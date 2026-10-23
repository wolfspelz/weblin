# CTO Review (long-term stewardship lens)

Reviewed from the position of a **CTO** responsible for weblin as a long-lived product:
a browser extension installed on strangers' machines that **runs in every tab, all day, for years of releases**,
and a server — WebEx, WebIt and the Orleans silo — that **runs always-on for every user at once**
and holds their items and identity.
The question is never "is this line clean" but:
*does this change keep the system **manageable, maintainable, reliable, and private**
for years and hundreds of future extensions — or does it make those goals harder?*

This is one of the [lenses](../README.md#the-lenses) the [default review](README.md) runs in its own context.
Select it alone with `review <repo> <change> cto`.
Like the [engineering lens](engineering.md), it is a **heavy multi-agent** pass:
independent reviewers each own one of the CTO's accountabilities,
findings are adversarially verified, then synthesized into a CTO verdict.

## The CTO's charter — what this lens is accountable for

For a long time, and for users who let the product into every web page they visit, the system must:

- **Run reliably** — the extension survives service-worker suspension, tab sleep,
  XMPP and websocket reconnects, and host pages that rewrite their DOM;
  the silo survives restarts, storage failures and load peaks;
  both keep working across version skew, because users update the extension days after the server deploys.
- **Stay manageable, maintainable & extensible**
  — a solid architecture that *hundreds* of future extensions can build on,
  that **new** developers (many of them, later) can read, understand the relevant parts of, and extend safely
  — which is why the project gates every code change on a document in `docs/`.
- **Protect privacy** — the product sees which pages its users visit, who else is there, and what they say;
  that data is collected, stored, logged and transmitted in a way that honours the trust the user placed in an extension.

**A change that makes these goals harder is a finding.**
Every change is weighed by whether it *advances or weakens* them,
and that judgement is what every finding rests on.

**"Small risks compound across the install base."**
A defect that is individually minor but runs in every tab of every user
is a **systemic reliability finding, not a nit**.
A leaked timer, an unbounded cache, a reconnect without backoff, a stanza handler that assumes arrival order
— each one is harmless on a developer's machine and a support queue across the install base.
Weigh a finding by its total cost across all installs and all tabs, not just its local effect
— do **not** let adversarial verification downgrade it because one instance looks harmless.

## Principle — two acceptable ways to guarantee functionality

The CTO requires that the system
"**runs reliably and does what it is supposed to do, long-term — no matter how the code was created.**"
Functionality may be guaranteed by **either** path below;
either is accepted, but **at least one** is required:

- **A · Engineered for humans** — code quality, good engineering, good architecture,
  and an architecture document in `docs/`:
  a human can find the relevant part, read it, understand it, and trust and extend it.
  The traditional path; still considered essential.
- **B · Proven by behaviour** — the code's behaviour is covered by tests
  that exercise *all expected behaviours* — including reconnect, restart, version skew and edge cases.
  Here the code itself may be opaque or machine-generated,
  yet its behaviour is guaranteed and a regression makes a test fail.

A change that is **neither** cleanly engineered for a human reader **nor** behaviourally proven
is the worst case — **functionality nobody can verify and the CTO cannot vouch for long-term**; flag it.
Opaque code that arrives *with* tests covering its behaviours is acceptable on this axis,
and so is plain, readable, documented code with modest tests.
Name which path a change relies on; "neither" is a finding.

## Criteria & patterns this lens applies

- **Engineering criteria**, read at system level rather than line level
  ([`criteria/engineering.md`](../criteria/engineering.md))
  — §2 *Simplicity*, §3 *Blast radius* and §5 *Engineering patterns*
  carry the maintainability and architecture judgement;
  §1 *Proportionality* bounds long-term cost of ownership.
- **Tests** ([`criteria/test.md`](../criteria/test.md)) — *Meaningful unit tests* is path-B evidence,
  and path B only counts when the harness runs the **real** production code
  (*Test the real production code, isolated by DI*), not a copy
  (*Test-only production path* — a passing test that guards nothing is **not** path-B evidence).
- **Anti-patterns** ([`criteria/anti-patterns.md`](../criteria/anti-patterns.md))
  — *Premature abstraction* (speculative layers are future maintenance debt, not extensibility);
  *Unbounded blocking*; *Swallowed errors*; *Hidden temporal coupling*;
  *Completeness assumed from a timeout*; *PII / secrets in logs*.
- **Desired patterns** ([`criteria/desired-patterns.md`](../criteria/desired-patterns.md))
  — *Idempotent, restart-safe operations*; *Fail closed, degrade gracefully*.
- **The project's own architecture** ([`criteria/project.md`](../criteria/project.md))
  — the `docs/` documents and the architecture sections of both `CLAUDE.md` files
  describe the system this change must fit; the CTO reads them as the design of record.

The charter and the two-paths principle above govern every judgement this lens makes.

## Boundary with the other lenses

The CTO lens looks at the change as a whole and must not repeat what other lenses cover:

- It does **not** flag naming or restate per-line engineering findings
  (the [engineering lens](engineering.md) does that);
  it asks whether the *whole change* leaves the system maintainable and extensible.
- It does **not** check individual `CLAUDE.md` style rules (engineering dimension 6 does);
  it asks whether the change respects the *architecture* those files and `docs/` describe.
- It does **not** run the full security control checklist (the [security lenses](security.md) do);
  it raises privacy only as an obligation to the user
  — data exposure, over-collection, data written to logs.
- Where it overlaps another lens, it cites the same `file:line` so synthesis can merge
  — but it always speaks to **long-term stewardship**, not local correctness.

## Input

The shared change set + the stated intent, gathered once by the [default review](README.md) (Step 1),
plus the paths of the `docs/` documents covering the touched area.
Running this lens standalone?
Gather them first via that step — long-term cost is judged against what the change is *for*.

## Step 1 — Fan out (one reviewer per accountability)

Run as a Workflow: spawn the reviewers in parallel,
each given the diff + intent + worktree + rule-file paths and one accountability.
Each reviewer cites the criterion / pattern it invokes **by its heading** and every finding by `file:line`.

1. **Maintainability & extensibility at scale**
   — Can hundreds of future extensions build on this without working around it?
   Can a *new* developer find, read and safely extend the relevant part
   — is the area documented in `docs/`, and does the change keep that document true?
   Does it add lasting complexity, hidden coupling across the content-script/background boundary
   or the WebIt/silo boundary, or a shortcut that later work has to work around?
   (engineering §2, §3, §5; anti-pattern *Premature abstraction*; project — the documentation gate.)
2. **Reliability in long-term operation** — Deterministic behaviour;
   correct under load and edge cases; safe for always-on operation.
   Extension: service-worker suspension (no raw timers in background code), tab sleep/wake,
   reconnect storms, host pages that mutate the DOM, memory growth in long-lived tabs.
   Server: grain reactivation and persistence, deadlock-safe grain calls, storage failures,
   version skew between deployed services and installed extensions.
   Does it fail closed and degrade gracefully?
   Check **state-recovery** specifically:
   when the change loads or rebuilds state at startup/reconnect,
   is that state **provably complete** (a positive completeness signal)
   or only **probably complete after a timeout**
   — where late data is dropped without notice and the state is wrong with nothing reporting it?
   The extension's `CLAUDE.md` *server notification ordering* rule is the house statement of this hazard.
   (engineering §5; anti-patterns *Completeness assumed from a timeout*, *Unbounded blocking*,
   *Swallowed errors / silent failure*, *Hidden temporal coupling*;
   desired-patterns *Fail closed, degrade gracefully*, *Idempotent, restart-safe operations*.)
3. **Guarantee of functionality (the two paths)** — Apply the *two-paths* principle:
   is this change's behaviour guaranteed by **A** (clean, documented, human-understandable engineering)
   **or B** (tests covering its behaviours incl. reconnect / restart / edge cases)?
   Name the path it relies on; flag changes that satisfy **neither**.
   (tests *Meaningful unit tests*, *Test the real production code, isolated by DI*,
   *Test-only production path*.)
4. **Privacy & data stewardship**
   — Does the change collect, store, log, or transmit user data it shouldn't
   — visited URLs, room membership, chat text, nicknames, identity tokens —
   widen who can see it, or weaken a privacy guarantee users rely on?
   Report concrete exposure; leave detailed control review to the security lens.
   (anti-pattern *PII / secrets in logs & over-collection*.)

## Step 2 — Verify (adversarial)

Every non-trivial finding is checked by independent verifier(s) prompted to **refute** it
(default to "refuted" when uncertain)
— e.g. "a new developer *can* extend this safely because…",
"these behaviours *are* covered because…",
**citing the code / tests / docs, not the author's claims**.
Findings that don't survive are dropped or downgraded.
This is the guard against confident-but-wrong AI findings — do not skip it.

**Exception — test-triviality findings are not refute-dropped.**
When a path-B judgement turns on a test that invokes [`criteria/test.md`](../criteria/test.md)
*Useless tests — concrete shapes to flag* (tautological/mirror/etc.),
verify it per that file's *Verifying a "useless test" finding* rule, **not** the default-refuted bias:
a passing test that only mirrors the code it tests is **not** path-B evidence,
and "the behaviour is covered by another test" confirms the duplicate should be deleted
rather than refuting the finding.

## Step 3 — CTO verdict

- **Verdict** — merge / merge-with-nits / needs-work / reject + one sentence,
  framed as the CTO's stewardship judgement:
  *does this keep the system manageable, reliable, and private over years?*
- **Findings** grouped by accountability, severity-ordered, each with `file:line`,
  **why it matters for the long-term / at-scale goal**, and **what to merge instead**.
  Each path-B judgement states which behaviours are (or aren't) covered.
  Nitpicks kept separate from the stewardship findings.
- **Origin on every finding** — *introduced* or *pre-existing*,
  per [`README.md`](README.md#every-finding-carries-an-origin--introduced-or-pre-existing).
  This lens looks at the whole system, so it finds pre-existing weaknesses more often than the others:
  a pre-existing `minor`/`nit` is reported as **debt**, never as a merge condition,
  and a pre-existing `major`/`blocker` stays a finding and says what this change does to it.
- **The reporting bar** — only what must or should be fixed,
  per [`README.md`](README.md#finding-nothing-is-a-good-result--report-what-must-or-should-be-fixed).
  Judged over years, almost any change can be argued to make *something* marginally harder,
  which makes speculative findings cheap to produce here:
  raise a stewardship risk only where a good engineer would change the code for it,
  and report a sound change as sound.

This verdict feeds the combined report the [default review](README.md) synthesizes across lenses.
It is **advisory** — the human makes the merge decision.
