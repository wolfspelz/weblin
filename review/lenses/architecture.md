# Architecture Review

**Status: planned — stub.**

A lens focused on how the change fits the system's structure, beyond local code quality.

Intended scope (to be developed):

- Does the change respect existing boundaries (modules, bounded contexts, layers)?
- Does it introduce coupling across boundaries that should stay separate?
- Are new dependencies pointed in the right direction (Dependency Inversion)?
- Does it duplicate or contradict an existing architectural decision (see the architecture docs in `C:/Heiner/weblin/docs/` and the architecture sections of both `CLAUDE.md` files)?
- Is this the right *place* for the change, or is it bolted onto the wrong layer?

Flesh this out before first use; removing the `planned` status line makes the [default review](README.md) run this lens automatically.
