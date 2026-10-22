# Adversarial Security Review

**Status: planned — stub.**

A deeper, attacker-minded counterpart to the [security review](security.md): not "are the obvious controls present" but "how would someone abuse this change".

Intended scope (to be developed):

- Threat-model the change: who benefits from breaking it, and how?
- Hunt for abuse cases, not just missing checks: race conditions, TOCTOU, boundary confusion, privilege escalation paths opened by the change.
- Adversarial verification: each candidate weakness is probed for a concrete exploit path before it's reported.

Authorized, defensive use only — reviewing our own code for weaknesses.

Flesh this out before first use; removing the `planned` status line makes the [default review](README.md) run this lens automatically.
