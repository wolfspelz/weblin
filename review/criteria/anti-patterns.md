# Anti-Patterns

Patterns we **reject** in a change.
Each entry: the pattern, why it's bad, and what to do instead.
Added by hand when the user asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)).

When a review flags one of these, cite the entry by its heading.

---

## Gratuitous rewrite / scope creep

Rewriting working code that the task didn't require touching.

- **Why bad:** inflates blast radius, hides the real change, risks regressions in code that was fine.
- **Instead:** change only what the task needs.
  If a refactor is genuinely warranted, make it a separate, clearly-labeled PR.

## Over-naming / exaggerated identifiers

Names that overstate what the code does (`status_retriever_and_reconciler` for a getter; `*Manager`/`*Coordinator` for a plain helper).

- **Why bad:** misleads readers about responsibility and complexity.
- **Instead:** name for what it *is*.
  A getter is a getter.

## Premature abstraction / speculative generality

Layers, generics, config or hooks added for needs that don't exist yet.

- **Why bad:** complexity with no current payoff; harder to read and change.
- **Instead:** solve today's problem.
  Abstract when the second real caller appears.

## Drive-by churn

Reformatting, import reordering, whitespace or rename changes unrelated to the fix.

- **Why bad:** pollutes the diff, obscures the substantive change, complicates review.
- **Instead:** keep formatting-only changes out of feature MRs.

## Duplicated source of truth

The same constant, rule, or piece of state defined in more than one place.

- **Why bad:** breaks single-source-of-truth; the copies drift.
- **Instead:** define once, reference everywhere.

---

<!-- Add new anti-patterns below this line. -->

## Unbounded blocking — I/O without a timeout

A call that can wait forever: a network request, websocket/XMPP round-trip, grain call, storage call, or lock acquisition issued with no timeout or bounded wait.

- **Why bad:** in a silo one stuck call pins a grain activation indefinitely and in a service worker it pins a pending message; under load these accumulate until the service hangs — a single slow dependency becomes a total outage.
  The failure is invisible until it strikes at a peak.
- **Instead:** put a bounded timeout on every blocking I/O, decide the behaviour when it fires (retry with backoff, fail closed, degrade — see [Fail closed, degrade gracefully](desired-patterns.md)), and free the resource.
  A bounded wait that fails is recoverable; an unbounded one is not.

## Swallowed errors / silent failure

Catching an exception and ignoring it — an empty `catch`, a `catch (Exception)` that logs nothing actionable, a fire-and-forget `.catch()` that drops the error instead of routing it to `app.onError`, or continuing past a corrupt/partial result as if it were valid.

- **Why bad:** the failure disappears.
  State silently diverges, the true cause is lost, and in an extension running in a stranger's browser or a silo running for every user the symptom surfaces far from the cause — undiagnosable and unrecoverable.
  It also defeats *fail closed*: the code proceeds on bad data instead of stopping.
- **Instead:** handle the error you can genuinely recover from; let the rest propagate to a boundary that decides the safe outcome (see [Fail closed, degrade gracefully](desired-patterns.md)).
  Catch narrowly, preserve the cause, and never read "no error raised" as "operation succeeded."

## Hidden temporal coupling (must-call-in-order, unenforced)

Methods or steps that only work if invoked in a specific, undocumented order — `init()` before use, `set_x()` before `get_y()`, "configure then start" — with nothing in the code that enforces or signals the requirement.

- **Why bad:** the constraint lives only in the author's head.
  A new developer or a future extension wires the steps in a different order and gets a silent, half-initialised state rather than an error.
  The [CTO lens](../lenses/cto.md) flags this under *ordering assumptions* and *can a new developer extend this safely* — here they can't.
- **Instead:** make the order impossible to get wrong — construct fully-initialised objects, fold the prerequisite into the dependent call, or guard with an explicit state check that fails loudly.
  If a sequence is genuinely required, encode and assert it rather than leaving it to a comment.

## Completeness assumed from a timeout (snapshot-by-deadline)

A bounded wait whose **expiry is treated as "all the data has now arrived"** — reading an items snapshot or presence stanzas into room state after (re)connect, a startup warm-up — after which whatever showed up is taken as the complete picture.
Late-but-valid data that arrives after the deadline is silently dropped, so the state is quietly **wrong**.

- **Why bad:** this is the opposite failure to [Unbounded blocking](#unbounded-blocking--io-without-a-timeout) — the wait *is* bounded (good), but its *expiry is misused as a completeness signal* (bad).
  The system **fails open**: it proceeds on partial state as if whole, with no error.
  The defect is timing-dependent — a slow broker, a cold cache, a busy start — so it hides in development and strikes intermittently in production, exactly the long-running reliability the [CTO lens](../lenses/cto.md) guards.
  Across every tab of every install, the odds that *some* room renders with wrong state on any given reconnect approach certainty (*"small risks compound across the install base"*).
  The extension's `CLAUDE.md` *server notification ordering* rule is the house statement of this hazard: an items snapshot is not an arrival barrier for presence.
- **Instead:** derive completeness from a **positive** signal, not the clock — a known key set fully received, a version/watermark reached, an end-of-snapshot marker, an expected count.
  If completeness genuinely can't be proven in bounded time, treat the post-deadline state as **provisional** and *reconcile* as late data arrives (see [Idempotent, restart-safe operations](desired-patterns.md)), or **fail closed** rather than continue on a state you can't vouch for (see [Fail closed, degrade gracefully](desired-patterns.md)).
  A clock may bound *how long you wait*; it must never decide *that you are done*.

## PII / secrets in logs & over-collection

Logging, persisting, or transmitting user data — visited URLs, room names, chat text, nicknames — credentials, tokens, or session identifiers — or collecting more personal data than the feature needs, or widening who and what can see data that already exists.

- **Why bad:** logs and telemetry are long-lived, widely readable, and often shipped off-box; a token or personal record written there is an exposure that outlives the request and breaks the trust a user placed in an extension that sees every page they visit.
  Over-collection enlarges the blast radius of any future breach.
- **Instead:** log identifiers and outcomes, not payloads or secrets; redact or hash what must be referenced; collect and retain only the minimum the feature needs.
  Apply the test "would I be comfortable if this log left the building?"
  Deep control review belongs to the [security lens](../lenses/security.md); this is the privacy-as-user-trust slice the [CTO lens](../lenses/cto.md) owns.
