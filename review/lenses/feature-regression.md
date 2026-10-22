# Feature-Regression Review

**Status: planned — stub.**

A lens focused on how the change interacts with **already-shipped features** — the risk that fixing or extending one thing quietly breaks another.

Intended scope (to be developed):

- Which existing features touch the same code paths, state, or data?
- Does the change alter shared behavior that other features rely on?
- Are there implicit contracts (ordering, timing, side effects) being changed?
- Is existing test coverage sufficient to catch a regression here?
- Trace callers/callees of changed symbols to find affected features.

**Prerequisite — descriptions of the features that might be interfered with.**
This lens can only judge interference if it knows what already-shipped features exist and what each one depends on.
It needs, per candidate feature: what it does, the code paths / state / data it owns, and the implicit contracts (ordering, timing, side effects) it relies on.
Without such descriptions the review is blind to anything it can't reach by tracing callers/callees of the changed symbols.
Maintaining this catalog — as per-feature documents under the weblin root `docs/`, which the project already requires before any code change — is part of activating this lens.

Flesh this out before first use; removing the `planned` status line makes the [default review](README.md) run this lens automatically.
