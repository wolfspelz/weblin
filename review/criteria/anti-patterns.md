# Anti-Patterns

Patterns we **reject** in a change.
Each entry: the pattern, why it's bad, and what to do instead.
Added by hand when the user asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)).

When a review flags one of these, cite the entry by its heading.

---

## Gratuitous rewrite / scope creep

Rewriting working code that the task didn't require touching.

- **Why bad:** enlarges the blast radius, hides the real change, risks regressions in code that was fine.
- **Instead:** change only what the task needs.
  If a refactor is genuinely warranted, make it a separate, clearly-labeled PR.

## Over-naming / exaggerated identifiers

Names that overstate what the code does
(`statusRetrieverAndReconciler` for a getter; `*Manager`/`*Coordinator` for a plain helper).

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

- **Why bad:** adds noise to the diff, hides the substantive change, makes review harder.
- **Instead:** keep formatting-only changes out of feature PRs.
  The project's own legacy rule is the one exception: a line the change edits anyway is converted to the current style.

## Duplicated source of truth

The same constant, rule, or piece of state defined in more than one place.

- **Why bad:** breaks single-source-of-truth; the copies drift apart.
- **Instead:** define once, reference everywhere.

---

<!-- Add new anti-patterns below this line. -->

## Unbounded blocking — I/O without a timeout

A call that can wait forever:
a network request, a websocket or XMPP round-trip, a grain call, a storage call, or a lock acquisition
issued with no timeout or bounded wait.

- **Why bad:** in the silo one stuck call occupies a grain activation indefinitely,
  and in the service worker it leaves a message pending until the browser suspends the worker;
  under load these accumulate until the service stops responding,
  so a single slow dependency becomes a total outage.
  Nothing shows the problem until it happens under peak load.
- **Instead:** put a bounded timeout on every blocking I/O,
  decide the behaviour when it fires
  (retry with backoff, fail closed, degrade — see [Fail closed, degrade gracefully](desired-patterns.md)),
  and free the resource.
  A bounded wait that fails is recoverable; an unbounded one is not.

## Swallowed errors / silent failure

Catching an exception and ignoring it
— an empty `catch`, a `catch (Exception)` that logs nothing actionable,
a fire-and-forget `.catch()` that discards the error instead of routing it to `app.onError`,
or continuing past a corrupt/partial result as if it were valid.

- **Why bad:** the failure is recorded nowhere.
  State diverges unnoticed, the original cause is lost,
  and in an extension running in a stranger's browser or a silo serving every user the symptom appears far from the cause,
  which makes it hard to diagnose and hard to recover from.
  It also defeats *fail closed*: the code proceeds on bad data instead of stopping.
- **Instead:** handle the error you can genuinely recover from;
  let the rest propagate to a boundary that decides the safe outcome
  (see [Fail closed, degrade gracefully](desired-patterns.md)).
  Catch narrowly, preserve the cause, and never read "no error raised" as "operation succeeded."

## Hidden temporal coupling (must-call-in-order, unenforced)

Methods or steps that only work if invoked in a specific, undocumented order
— `init()` before use, `setX()` before `getY()`, "configure then start" —
with nothing in the code that enforces or signals the requirement.

- **Why bad:** the constraint is written down nowhere.
  A new developer or a future extension calls the steps in a different order
  and gets a half-initialised state instead of an error.
  The [CTO lens](../lenses/cto.md) flags this under *ordering assumptions*
  and *can a new developer extend this safely* — here they can't.
- **Instead:** make the order impossible to get wrong — construct fully-initialised objects,
  fold the prerequisite into the dependent call,
  or guard with an explicit state check that raises an error.
  If a sequence is genuinely required, encode and assert it rather than leaving it to a comment.

## Completeness assumed from a timeout (snapshot-by-deadline)

A bounded wait whose **expiry is treated as "all the data has now arrived"**
— reading an items snapshot or presence stanzas into room state after a reconnect, a startup warm-up —
after which whatever arrived is taken as complete.
Valid data arriving after the deadline is dropped without notice, so the state is **wrong**
and nothing says so.

- **Why bad:** this is the opposite failure to [Unbounded blocking](#unbounded-blocking--io-without-a-timeout)
  — the wait *is* bounded (good), but its *expiry is misused as a completeness signal* (bad).
  The system **fails open**: it proceeds on partial state as if it were complete, with no error.
  The defect is timing-dependent — a slow server, a cold cache, a busy start —
  so it does not appear in development and fails intermittently in production,
  which is the long-running reliability the [CTO lens](../lenses/cto.md) covers.
  Across every tab of every install,
  the odds that *some* room renders with wrong state on any given reconnect approach certainty
  (*"small risks compound across the install base"*).
  The extension's `CLAUDE.md` *server notification ordering* rule is the house statement of this hazard:
  an items snapshot is not an arrival barrier for presence.
- **Instead:** derive completeness from a **positive** signal, not the clock
  — a known key set fully received, a version/watermark reached, an end-of-snapshot marker, an expected count.
  If completeness genuinely can't be proven in bounded time,
  treat the post-deadline state as **provisional** and *reconcile* as late data arrives
  (see [Idempotent, restart-safe operations](desired-patterns.md)),
  or **fail closed** rather than continue on a state you can't vouch for
  (see [Fail closed, degrade gracefully](desired-patterns.md)).
  A clock may bound *how long you wait*; it must never decide *that you are done*.

## PII / secrets in logs & over-collection

Logging, persisting, or transmitting user data — visited URLs, room names, chat text, nicknames —
credentials, tokens, or session identifiers
— or collecting more personal data than the feature needs,
or widening who and what can see data that already exists.

- **Why bad:** logs and telemetry are kept for a long time, readable by many people,
  and often copied to other systems;
  a token or personal record written there stays exposed long after the request
  and breaks the trust a user placed in an extension that sees every page they visit.
  Over-collection enlarges the blast radius of any future breach.
- **Instead:** log identifiers and outcomes, not payloads or secrets;
  redact or hash what must be referenced; collect and retain only the minimum the feature needs.
  Apply the test "would this be acceptable if the log were read outside the company?"
  Detailed control review belongs to the [security lens](../lenses/security.md);
  this is the privacy part the [CTO lens](../lenses/cto.md) owns,
  because it is what the user trusts us with.
