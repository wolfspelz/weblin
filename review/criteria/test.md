# Tests — criteria & patterns

Everything test-specific in one place: the **criterion** that decides whether a test earns its keep (moved out of [engineering.md](engineering.md)), plus the **desired** and **rejected** test shapes (moved out of [desired-patterns.md](desired-patterns.md) and [anti-patterns.md](anti-patterns.md)).
The desired pattern is the positive form of the anti-pattern that follows it.

Added by hand when the user asks (see [Updating the criteria](../README.md#updating-the-criteria-and-lenses)).
When a review flags one of these, cite the entry by its heading.

---

## Meaningful unit tests

Unit tests must earn their place by guarding behavior or an interface that could plausibly break in the future.
(Integration tests are checked separately — this criterion is about unit tests only.)

- **Good — tests that catch a future failure:**
  - a **changed interface** would make callers fail;
  - a **behavior change** would give callers different results, state, or effects than they expect;
  - an **edge case** in input arguments (boundaries, empty, null, invalid combos).
- **Red flags:**
  - **TDD relics** — tests that only exist because something was written red-then- green, asserting trivia that can't realistically regress;
  - tests that restate the implementation (change the code → must change the test in lockstep) rather than pinning observable behavior;
  - tests of getters, setters, constructors, dataclasses, and wiring with no behavior to protect — including generated members like `__repr__`/equality and default values;
  - high test count, low protective value.

Ask of each test: *what future breakage does this catch?*
If there's no honest answer, the test is noise, not coverage.

## Useless tests — concrete shapes to flag

Specific forms of the *Red flags* above: a test that runs green but guards no plausible future breakage.
Each **must be reported as a finding** that says **delete it** (or replace it with a test that pins real behavior) — a review does not pass over one silently because the test is harmless, cheap, or green.
When a review flags one, cite it by name.

- **Mirrors a trivial line** — the whole assertion restates one trivial line of production code: a constant or default assignment, a field initializer, an insertion into a map or list, a pass-through getter or setter.
  It can turn red only when someone edits that line or edits the test — never when behaviour changes — so it guards the source, not the program.
  The **tautological variant** counts too: the test computes its expected value by calling the very production code it is testing, so the assertion compares the implementation against itself.
  Whether the mirrored line is *correct* is not the question; whether anything else could ever turn the test red is.
  The next two entries are this shape's most common concrete forms.
  Keep the test only when that line carries a contract an outside caller could break (a defensive copy, an exactly-once callback, a guard branch); otherwise delete it, or re-anchor it on an **independent oracle** — a hand-computed value, an invariant, or an observable effect.
- **Constant / config assertion** — asserts a config value, default, or constant equals a hardcoded literal (`expect(Config.get('timeoutSec')).to.equal(30)`, `Assert.AreEqual(30, Timeout)`); it restates the source and only goes red when someone edits the constant.
  Keep it only if a *computation or invariant* produces the value.
- **Store-and-retrieve round-trip** — writes a value into a dict, list, or plain attribute and asserts it reads back unchanged; it exercises the language and its containers, not your code.
  Test the logic that *decides* what to store, not that storage works.
- **Tests the framework or stdlib** — the assertion guards behavior owned by the language, serializer, or framework (record equality, JSON round-trips, Orleans persisting a grain state); that is tested upstream.
  Test only your code layered on top of it.
- **Assertion-free test** — no real assertion, or the only check is "it didn't throw"; it passes as long as the lines execute, so it can never turn red on a logic regression.
  Assert an observable outcome or delete it.
- **Coverage-only via privates** — exists to lift a coverage number and reaches into private state (reflection, `as any` casts, `InternalsVisibleTo` used only to reach private state) to touch otherwise-unreachable lines; brittle to refactors and guards nothing observable.
  Drive the line through the public path — if it can't be reached that way, suspect dead code.
- **Mockery — asserts the mock** — so much is mocked that the assertion only re-checks what a mock was told to return, or merely that a mock was called, so the production logic never runs (tautological; cf. *Test-only production path* below).
  Mock only true boundaries (I/O, network, clock, the communicator, storage) and let the real SUT compute.

Ask of each: *would this test ever fail for a reason other than someone editing the test or the line it mirrors?*
If not, it is noise.

### Verifying a "useless test" finding — do not refute-drop it

A review's adversarial verification defaults to **refuted when uncertain**, to kill confident-but-wrong findings.
**That default is suspended for a finding that invokes one of the shapes above.**
Such a finding must survive verification whenever the shape is really present, so the verifier checks only these two things — and may lower severity but **not** drop it once the shape is confirmed:

1. **Is the shape correctly identified?**
   The finding is refuted **only** if the citation is factually wrong — the test actually uses an **independent oracle** (a hand-computed value, an invariant, a separate reference implementation), or the mirrored line carries an **outside-observable contract** a caller could break (a defensive copy, an exactly-once callback, a guard branch), or the named shape simply isn't there.
   Absent that, the shape stands.
2. **"The behaviour is covered elsewhere" is not a refutation — it confirms the finding.**
   That the real behaviour is pinned by another, honest test does not rescue a tautological or mirror duplicate; it proves the duplicate adds no protection, so the disposition is **delete it**.
   Likewise "there is no better, non-tautological rewrite" argues for deletion, not for keeping.
   A verifier may **not** downgrade or drop on either ground.

Severity is usually `minor` (a lone weak case, `nit`); the finding is dropped only when the shape was misidentified per (1).
Passing over such a test silently because it is green, cheap, or redundant is the exact failure this criterion exists to prevent.

## Desired — Test the real production code, isolated by dependency injection

A test exercises the **production class** as its system under test (SUT) and calls a **production method** in its `// Act` section (the server's `// Arrange` / `// Act` / `// Assert` markers; a Chai `it()` body in the extension) — not a parallel re-implementation, a test-only seam, or a rebuilt twin.
Where the SUT would otherwise touch the environment (I/O, network, broker, clock, filesystem), those collaborators are **dependency-injected away** with fake/dummy/in-memory/simulator implementations (inversion of control), so the real production logic runs while its side effects are contained in the test.

- **Why good:** the test guards the code that actually ships — its branching, ordering, wiring and edge handling — so a regression in the production path turns a test red.
  DI/IoC keeps the test fast and deterministic without forking a separate code path that drifts from production.
  This is the positive form of the *Test-only production path* anti-pattern below.
- **Look for:** the SUT in the act step is a production type; the only test-shaped objects are the injected dependencies (a fake broker, a simulated clock, a recording client), not the SUT or its entry point.

## Anti-pattern — Test-only production path (test drives a parallel seam, not the real one)

A test exercises a method, branch, or entry point that exists **only to serve the test harness** — so the production code path it claims to cover is never actually run.
Common shape: production grows a public method to paper over a harness limitation (a fake communicator, a test-only reset of grain state), and the test calls *that* instead of the real bootstrap/hydration/startup path.

- **Why bad:** the test is green but guards nothing real — the production path it advertises (ordering, timing, wiring) could regress silently and every test would still pass.
  It also widens the production contract for test-only reasons, and reflective `hasattr/getattr` calls to such seams no-op silently on a rename.
  This is the "parallel rebuilt twin that tests itself" failure.
- **Instead:** make the test drive the **real** production path — inject a harness-aware collaborator (communicator, storage provider, clock) rather than a second entry point, and delete the test-only method.
  If a seam is unavoidable, it belongs in the test-support layer, not on the shipped production API.
  A test must answer: *does this run the code that actually runs in production?*

## Anti-pattern — Test helpers shipped in the production code (blocker)

Test-only helper code or test provisioning — fixture builders, synthetic-data packers/generators, fake or simulator collaborators, harness utilities — placed in the **shipped production code** (`ChromeExt/src/lib|background|contentscript`, or a non-test `.csproj`) instead of `ChromeExt/src/test/`, a `*.Test` project, or `n3q.TestTools`.
Tell-tale shape: a new module under the production source tree whose only importers are test files.

- **Why bad:** it ships test scaffolding in the production artifact and instantly becomes importable public API — production code can silently start depending on it, and it must then be maintained, reviewed, and deprecated like production code.
  It widens the shipped surface for a need that is entirely test-local, mixes concerns (test provisioning is a test concern), and often publishes **unverified or speculative knowledge** (an assumed data layout, a convenient fake) with the authority of shipped API.
  Tooling constraints never justify it — a uniquely named module inside the test folder resolves those just as well.
- **Instead:** put the helper with the tests it serves — `ChromeExt/src/test/`, the owning `*.Test` project, or `n3q.TestTools` for shared server test machinery — in any case **not in the production code**.
  Promote to a shipped testing module only deliberately, in the change that introduces a real consumer outside those test folders.
- **Severity:** flag as a **blocker** — this prevents the merge until the helper is moved; it is not a placement nit.

---

<!-- Add new test criteria or patterns below this line. -->
