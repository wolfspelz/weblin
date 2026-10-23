# Assumptions — what every review takes as given

Facts about the environment the reviewed system runs in, accepted without proof.

They remove one class of false positive:
the finding that is correct about the code but describes a situation the deployment rules out
— a clock jumping backwards, a storage write that reported success and did nothing.
Written down here, the situation is ruled out once for every lens instead of re-argued in each review.

## What an assumption is

A property of the platform, browser, deployment or operating procedure
that holds for every change unless the request says otherwise.
It would be true even if none of the reviewed code existed.

It bounds the environment; it never excuses the code.
Not assumptions: "the code is correct", "the caller passes valid input",
"covered by a test elsewhere", "that path is unlikely".

## How an assumption is applied

1. **Drop a finding only when violating the assumption is its only failure path.**
   Test: does the problem still happen if the assumption holds?
   If yes, keep the finding at unchanged severity
   and reword it so it no longer relies on the assumed-away situation.
2. **Switch, not discount.**
   Dropped whole or kept whole — never a lowered severity, never a hedge inside a finding.
3. **The request overrides.**
   When the trigger's intent, the PR description or a `docs/` document puts the case in scope,
   the assumption does not apply to that change.
4. **Code that handles the case is still reviewed**, like any other code.
   An assumption removes the obligation to handle a case; it says nothing about handling that ships anyway.
5. **In doubt, keep the finding.**
   Only what an entry plainly covers is suppressed.
6. **A suppressed finding leaves no trace** — not in the report, not in Coverage, not as a suppression note.

## Adding an assumption

Trigger: a concrete false positive, right about the code and wrong about the environment.
Name the fact that would have prevented it,
check it against [What an assumption is](#what-an-assumption-is), add an entry.

- One fact per entry, in the group it belongs to.
  Groups organise reading only; add groups as the list grows.
- IDs are permanent — next free number, regardless of group.
  Rewording or moving an entry does not renumber it; a retired number is never reused.
- Write all four fields: *Assume*, *Therefore not a finding*, *Still a finding*,
  *Added* (date plus the false positive that prompted the entry).
  *Still a finding* is what keeps an entry from becoming a blanket excuse;
  an entry with no real carve-outs is too broad.
- Retiring an entry is a normal delete.
  Its findings return on the next review.

> Added by hand when the user asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)).
> Loaded into every review — the mechanism is
> [Assumptions about the operating environment](../lenses/README.md#assumptions-about-the-operating-environment)
> in the orchestrator.
> The first two entries were carried over from the workspace this one was adapted from;
> their false positives occurred there, not on weblin.

---

## Time & clocks

### A1 — System time is reliable and monotonic

**Assume:** a host's system clock — the user's machine running the extension, and the server hosts —
runs forward monotonically.
It never jumps — not set back, wound forward or reset,
neither during operation nor across a service worker, silo or machine restart.
A timestamp from an earlier run therefore always lies in the past.

**Therefore not a finding:** state, caches, `browser.storage` entries, logs or table rows
keyed, ordered or expired by wall-clock timestamps being corrupted by a backwards jump;
a restart misreading timestamps written by a previous run;
a negative duration measured across a restart;
an expiry or scheduling window a backwards jump would reopen or skip.

**Still a finding:** anything wrong under a well-behaved clock
— a comparison with the wrong sign, a unit or timezone confusion, a timestamp never updated,
wall-clock used where the code's own contract requires a monotonic source.
Clock skew between hosts is not covered: a finding resting on the extension's clock
and the server's clock disagreeing stands.

**Added:** 2026-10-23, carried over; the original false positive was a finding about timestamps
from an earlier run being misread when a service restarted with the clock set back.

## Platform & framework calls

### A2 — A system or framework call that reports success did what it says

**Assume:** a call into the browser, the operating system, Orleans, Azure Table Storage,
the XMPP or websocket client or a comparable library
is trusted at its word: success returned and no error signalled means the promised effect happened.
A grain state write that returns without failure has persisted the state;
a `browser.storage` write that resolves has stored the value;
a websocket `send` that returns has handed the frame to the socket.

**Therefore not a finding:** the absence of a read-back that only re-confirms what the call already reported
— reading the row after a successful write, re-reading storage after a successful set.
The missing test for such a read-back is not a finding either.

**Still a finding:** a success signal that is never checked — a result ignored, an exception swallowed,
a rejected promise with no handler.
An API whose own contract makes success mean *accepted*, not *done*, is not covered:
a queued message, a fire-and-forget grain call, a send on a socket that may already be closing.
Neither is a call that succeeds at the wrong thing — wrong grain key, wrong storage key, wrong payload —
nor state a second actor can change after the call returns.
Confirmation is required only where a `docs/` document or the request demands it;
then its absence is a finding, per rule 3, never because a reviewer considers it safer.

**Added:** 2026-10-23, carried over; the original false positives asked for a follow-up read
to confirm a database write and a message-broker deletion that had both already reported success.
