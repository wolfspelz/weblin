# Desired Patterns

Patterns we **want** to see and will positively note in a review.
Each entry: the pattern and why it's good.
Added by hand when the user asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)).

This list starts small on purpose — entries are added from real, observed-good MRs rather than invented up front.
Project-wide conventions already live in the two repos' `CLAUDE.md` files and in `docs/`; this file captures *review-specific* preferences beyond those.

---

## Surgical, proportional change

The diff matches the problem: minimal files touched, no unrelated edits, the intent obvious from the change alone.

## Intentional, called-out refactors

When a refactor is needed, it's separated from the feature change and explicitly justified — not smuggled in.

---

<!-- Add new desired patterns below this line. -->

## Idempotent, restart-safe operations

An operation that may be retried or replayed — after a service-worker restart, an XMPP or websocket reconnect, a grain reactivation, a redelivered notification, or a re-run on the same input — converges to the **same** state and produces **no** duplicate side effects.
Writes are keyed or guarded so a second application is a no-op; startup re-derives or reconciles state rather than assuming a clean slate.

- **Why good:** the service worker is suspended and restarted by the browser and the silo reactivates grains at will, so restarts, reconnects and redeliveries are normal, not exceptional.
  Idempotent operations make those events safe — recovery is just "run it again" — instead of corrupting state or double-acting (double-write, double-emit, double-charge).
  This is the reliability the [CTO lens](../lenses/cto.md) demands under *error recovery / restart & persistence*.
- **Look for:** writes guarded by a key/version/existence check; startup that reconciles to the desired state; consumers that tolerate at-least-once delivery; no "assume this runs exactly once" assumption baked into the path.

## Fail closed, degrade gracefully

On an error, timeout, or unexpected input, the change defaults to the **safe** state and continues with **reduced** function — it does not take the whole process down, and it does not silently proceed on bad data as if nothing happened.

- **Why good:** a browser extension and an always-on silo must survive partial failure without dropping the whole service or quietly corrupting state.
  Failing closed keeps the safe default; degrading gracefully keeps core function alive while a non-essential part is down — the reliability posture the [CTO lens](../lenses/cto.md) asks for ("does it fail closed and degrade gracefully?").
- **Look for:** the error path lands on a *defined* safe state (deny / last-known-good / skip-with-signal), not an undefined one; a failed non-essential dependency disables only its own feature; failures are surfaced, not swallowed — the inverse anti-pattern is [Swallowed errors / silent failure](anti-patterns.md).
