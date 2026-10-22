# CTO Review (long-term stewardship lens)

Reviewed from the chair of a **seasoned CTO** accountable for weblin as a long-lived product: a browser extension installed on strangers' machines that **runs in every tab, all day, for years of releases**, and a server — WebEx, WebIt, and the Orleans silo — that **runs always-on for every user at once** and holds their items and identity.
The question is never "is this line clean" but: *does this change keep the system **manageable, maintainable, reliable, and private** for years and hundreds of future extensions — or does it quietly make those goals harder?*

This is one of the [lenses](../README.md#lenses) the [default review](README.md) runs in its own context.
Select it alone with `review <repo> <change> cto`.
Like the [engineering lens](engineering.md), it is a **heavy multi-agent** pass: independent reviewers each own one of the CTO's accountabilities, findings are adversarially verified, then synthesized into a CTO verdict.

## The CTO's charter — what this lens is accountable for

For a long time, and for users who let the product into every web page they visit, the system must:

- **Run reliably** — the extension survives service-worker suspension, tab sleep, XMPP and websocket reconnects, and hostile host pages; the silo survives restarts, storage hiccups and load peaks; both keep working across version skew, because users update the extension days after the server deploys.
- **Stay manageable, maintainable & extensible** — a solid architecture that *hundreds* of future extensions can build on, that **new** developers (many of them, later) can read, understand the relevant parts of, and extend safely — which is why the project gates every code change on a document in `docs/`.
- **Protect privacy** — the product sees which pages its users visit, who else is there, and what they say; that data is collected, stored, logged and transmitted in a way that honours the trust the user placed in an extension.

The CTO **despises any change that makes these goals harder**, and weighs every change by whether it *advances or erodes* them.
That is the through-line of every finding.

**"Small risks compound across the install base."**
A defect that is individually minor but runs in every tab of every user is a **systemic reliability finding, not a nit**.
A leaked timer, an unbounded cache, a reconnect without backoff, a stanza handler that assumes arrival order — each one is harmless on a developer's machine and a support queue across the install base.
Weigh a finding by its aggregate cost across all installs and all tabs, not just its local blast radius — do **not** let adversarial verification downgrade it merely because one instance looks harmless.

## Principle — two acceptable ways to guarantee functionality

The CTO insists the system "**runs reliably and does what it is supposed to do, long-term — no matter how the code was created.**"
Functionality may be guaranteed by **either** path below; the CTO accepts either, but demands **at least one**:

- **A · Engineered for humans** — code quality, good engineering, good architecture, and an architecture document in `docs/`: a human can find the relevant part, read it, understand it, and trust and extend it.
  The traditional path; still considered essential.
- **B · Proven by behaviour** — the code's behaviour is pinned by tests that exercise *all expected behaviours* — including reconnect, restart, version skew, and edge cases.
  Here the code itself may be opaque or machine-generated, yet its behaviour is guaranteed and a regression turns the harness red.

A change that is **neither** cleanly engineered for a human reader **nor** behaviourally proven is the worst case — **unverifiable functionality the CTO cannot stand behind long-term**; flag it.
Name which path a change relies on; "neither" is a finding.

## Criteria & patterns this lens applies

- **Engineering criteria**, read at portfolio altitude ([`criteria/engineering.md`](../criteria/engineering.md)) — §2 *Simplicity*, §3 *Blast radius* and §5 *Engineering patterns* are the maintainability/architecture backbone; §1 *Proportionality* bounds long-term cost of ownership.
- **Tests** ([`criteria/test.md`](../criteria/test.md)) — *Meaningful unit tests* is path-B evidence, and path B only counts when the harness drives the **real** production code (*Test the real production code, isolated by DI*), not a parallel twin (*Test-only production path* — a green test that guards nothing is **not** path-B evidence).
- **Anti-patterns** ([`criteria/anti-patterns.md`](../criteria/anti-patterns.md)) — *Premature abstraction* (speculative layers are future maintenance debt, not extensibility); *Unbounded blocking*; *Swallowed errors*; *Hidden temporal coupling*; *Completeness assumed from a timeout*; *PII / secrets in logs*.
- **Desired patterns** ([`criteria/desired-patterns.md`](../criteria/desired-patterns.md)) — *Idempotent, restart-safe operations*; *Fail closed, degrade gracefully*.
- **The project's own architecture** ([`criteria/project.md`](../criteria/project.md)) — the `docs/` documents and the architecture sections of both `CLAUDE.md` files describe the system this change must fit; the CTO reads them as the design of record.

The charter and the two-paths principle above govern how every call is weighed.

## Boundary with the other lenses

The CTO lens is **higher-altitude** and must not re-litigate what other lenses own:

- It does **not** nitpick naming or restate per-line engineering findings (the [engineering lens](engineering.md) does that); it asks whether the *whole change* leaves the system maintainable and extensible.
- It does **not** check individual `CLAUDE.md` style rules (engineering dimension 6 does); it asks whether the change respects the *architecture* those files and `docs/` describe.
- It does **not** run the full security control checklist (the [security lenses](security.md) do); it raises privacy only as a **user-trust** obligation — data exposure, over-collection, leaky logs.
- Where it overlaps another lens, it cites the same `file:line` so synthesis can merge — but it always speaks to **long-term stewardship**, not local correctness.

## Input

The shared change set + the stated intent, gathered once by the [default review](README.md) (Step 1), plus the paths of the `docs/` documents covering the touched area.
Running this lens standalone?
Gather them first via that step — long-term cost is judged against what the change is *for*.

## Step 1 — Fan out (one reviewer per accountability)

Run as a Workflow: spawn the reviewers in parallel, each given the diff + intent + worktree + rule-file paths and one accountability.
Each reviewer cites the criterion / pattern it invokes **by its heading** and every finding by `file:line`.

1. **Maintainability & extensibility at scale** — Will hundreds of future extensions build on this without fighting it?
   Can a *new* developer find, read and safely extend the relevant part — is the area documented in `docs/`, and does the change keep that document true?
   Does it add lasting complexity, hidden coupling across the content-script/background boundary or the WebIt/silo boundary, or a load-bearing shortcut that future work must route around?
   (engineering §2, §3, §5; anti-pattern *Premature abstraction*; project — the documentation gate.)
2. **Reliability in long-term operation** — Deterministic behaviour; correct under load and edge cases; safe for always-on operation.
   Extension: service-worker suspension (no raw timers in background code), tab sleep/wake, reconnect storms, host pages that mutate the DOM, memory growth in long-lived tabs.
   Server: grain reactivation and persistence, deadlock-safe grain calls, storage failures, version skew between deployed services and installed extensions.
   Does it fail closed and degrade gracefully?
   Probe **state-recovery** specifically: when the change loads or rebuilds state at startup/reconnect, is that state **provably complete** (a positive completeness signal) or only **probably complete after a timeout** — where late data is silently dropped and the state is quietly wrong?
   The extension's `CLAUDE.md` *server notification ordering* rule is the house statement of this hazard.
   (engineering §5; anti-patterns *Completeness assumed from a timeout*, *Unbounded blocking*, *Swallowed errors / silent failure*, *Hidden temporal coupling*; desired-patterns *Fail closed, degrade gracefully*, *Idempotent, restart-safe operations*.)
3. **Guarantee of functionality (the two paths)** — Apply the *two-paths* principle: is this change's behaviour guaranteed by **A** (clean, documented, human-understandable engineering) **or B** (tests covering its behaviours incl. reconnect / restart / edge cases)?
   Name the path it relies on; flag changes that satisfy **neither**.
   (tests *Meaningful unit tests*, *Test the real production code, isolated by DI*, *Test-only production path*.)
4. **Privacy & data stewardship** — Does the change collect, store, log, or transmit user data it shouldn't — visited URLs, room membership, chat text, identity tokens — widen who can see it, or weaken a privacy guarantee users rely on?
   Raise concrete exposure; defer deep control review to the security lens.
   (anti-pattern *PII / secrets in logs & over-collection*.)

## Step 2 — Verify (adversarial)

Every non-trivial finding is checked by independent verifier(s) prompted to **refute** it (default to "refuted" when uncertain) — e.g. "a new developer *can* extend this safely because…", "these behaviours *are* covered because…", **citing the code / tests / docs, not the author's claims**.
Findings that don't survive are dropped or downgraded.
This is the guard against confident-but-wrong AI findings — do not skip it.

**Exception — test-triviality findings are not refute-dropped.**
When a path-B judgement turns on a test that invokes [`criteria/test.md`](../criteria/test.md) *Useless tests — concrete shapes to flag*, verify it per that file's *Verifying a "useless test" finding* rule, **not** the default-refuted bias: a green test that only mirrors the code it tests is **not** path-B evidence, and "the behaviour is covered by another test" confirms the duplicate should be deleted rather than refuting the finding.

## Step 3 — CTO verdict

- **Verdict** — merge / merge-with-nits / needs-work / reject + one sentence, framed as the CTO's stewardship call: *does this keep the system manageable, reliable, and private for the long haul?*
- **Findings** grouped by accountability, severity-ordered, each with `file:line`, **why it matters for the long-term / at-scale goal**, and **what to merge instead**.
  Each path-B judgement states which behaviours are (or aren't) covered.
  Nitpicks separated so they don't drown the strategic signal.

This verdict feeds the combined report the [default review](README.md) synthesizes across lenses.
It is **advisory** — the human makes the merge decision.
