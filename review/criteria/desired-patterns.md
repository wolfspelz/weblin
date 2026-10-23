# Desired Patterns

Patterns we **want** to see and will note positively in a review.
Each entry: the pattern and why it's good.
Added by hand when the user asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)).

This list starts small on purpose — entries come from real changes that did it well, not from invention up front.
Project-wide conventions already live in the two repos' `CLAUDE.md` files and in `docs/`;
this file holds *review-specific* preferences beyond those.

---

## Surgical, proportional change

The diff matches the problem: minimal files touched, no unrelated edits, the intent obvious from the change alone.

## Intentional, called-out refactors

When a refactor is needed, it is separated from the feature change and explicitly justified,
not mixed into it.

---

<!-- Add new desired patterns below this line. -->

## Idempotent, restart-safe operations

An operation that may be retried or replayed
— after a service-worker restart, an XMPP or websocket reconnect, a grain reactivation,
a redelivered notification, or a re-run on the same input —
ends in the **same** state and produces **no** duplicate side effects.
Writes are keyed or guarded so a second application does nothing;
startup re-derives or reconciles state rather than assuming a clean slate.

- **Why good:** the browser suspends and restarts the service worker at will and the silo reactivates grains at will,
  so restarts, reconnects and redeliveries are normal, not exceptional.
  Idempotent operations make those events safe — recovery is "run it again" —
  instead of corrupting state or acting twice (double-write, double-emit, double-charge).
  This is the reliability the [CTO lens](../lenses/cto.md) requires
  under *error recovery / restart & persistence*.
- **Look for:** writes guarded by a key/version/existence check;
  startup that reconciles to the desired state; consumers that tolerate at-least-once delivery;
  no "runs exactly once" assumption built into the path.

## Fail closed, degrade gracefully

On an error, timeout, or unexpected input,
the change defaults to the **safe** state and continues with **reduced** function
— it does not stop the whole process, and it does not continue on bad data without reporting it.

- **Why good:** a browser extension and an always-on silo must survive partial failure
  without stopping the whole service or corrupting state unnoticed.
  Failing closed keeps the safe default;
  degrading gracefully keeps core function running while a non-essential part is down
  — the reliability the [CTO lens](../lenses/cto.md) asks for
  ("does it fail closed and degrade gracefully?").
- **Look for:** the error path ends in a *defined* safe state
  (deny / last-known-good / skip-with-signal), not an undefined one;
  a failed non-essential dependency disables only its own feature;
  failures are reported, not swallowed
  — the inverse anti-pattern is [Swallowed errors / silent failure](anti-patterns.md).
