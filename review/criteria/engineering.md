# Core Engineering Criteria

The baseline lens for every review.
Derived from traditional good-engineering principles, oriented toward catching the failure modes of AI-generated changes (over-reach, bloat, exaggeration).

Each criterion lists **what good looks like** and **red flags** to call out.

---

## 1. Proportionality — change size vs. problem

The size of the diff should match the size of the problem it solves.

- **Good:** a small problem yields a small, surgical change.
- **Red flags:** large diffs for small problems; many files touched to fix one bug; new abstractions introduced to solve a one-off.

## 2. Simplicity — simplest *solid* solution, not bloated

Prefer the simplest solution that is still robust.
Simple ≠ fragile.

- **Good:** straightforward control flow; no layer that doesn't earn its place; the solution a careful engineer would reach for first.
- **Red flags:** speculative generality ("we might need it later"); configuration for things that never vary; wrappers around wrappers; clever code where plain code would do.

## 3. Blast radius — keep it as small as possible

Resist the tendency (especially AI's) to rewrite surrounding code.

- **Good:** changes are local to what the task requires; unrelated code is left alone; refactors, if any, are intentional and called out — not silent.
- **Red flags:** drive-by reformatting; renaming across files unrelated to the fix; "while I was here" changes; reflowing imports/whitespace that bloats the diff.

## 4. Clean code & naming — accurate, not exaggerated

Names should describe what a thing *is*, no more.

- **Good:** a getter is `get_x` / `x`, not `status_retriever_and_reconciler`; small functions, single responsibility, meaningful names.
- **Red flags:** names that overstate behavior; ceremony nouns (`Manager`, `Handler`, `Service`, `Coordinator`) where a plain noun fits; functions doing more than their name claims.

## 5. Engineering patterns

Standard principles, applied with judgment.

- **Single source of truth** — no duplicated state/logic; one place owns each fact.
- **Separation of concerns** — distinct responsibilities live in distinct units.
- **SOLID** — especially Single Responsibility and Dependency Inversion.
- **Red flags:** copy-pasted logic; the same constant/rule defined twice; a class that knows too much; concerns leaking across boundaries.

---

> This file holds the *current* rules. New criteria are added by hand when the user
> asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)). Keep entries concise.
> Test-specific criteria and patterns live in [`test.md`](test.md).
