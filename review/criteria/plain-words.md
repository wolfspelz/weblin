# Plain words — the vocabulary a report does not use

The words and phrases the plain-words rule
([`reports/README.md`](../reports/README.md#plain-words--summary-and-short)) rejects, with what to write instead.
Loaded into every review, and read once more before a report is posted
([`lenses/README.md`](../lenses/README.md#step-5--save-the-report-and-post-it-to-the-pr)).

The list comes from a scan of 64 finished reports in the workspace this one was adapted from.
About 1,500 cases were found, and nearly all of them are the words below, repeated.
Summary and Short must be free of them; Detail may stay technical, but the metaphors do not belong there either.

Where the plain replacement depends on what was meant, the row says so.
A word that is literal in its context stays: a *dead* branch in code is a dead branch.

## Tests

| Written | Meant |
|---|---|
| a test *pins* X, X is *pinned by* a test, *pinning* | X has a test, X is checked by a test |
| *unpinned*, *pinned by nothing*, *pinned by no test* | has no test |
| the suite *stays green*, *ships green*, *leaves the suite green* | the tests still pass |
| the test *goes red*, *turns red*, *turns no test red* | the test fails, no test fails |
| a *green* build, a *green* run | a passing build |
| a *false green* | a test that passes without checking anything |
| the test *drives* the real class, *driven by* a test | the test runs the real class, tested by |
| *exercises*, *exercised by* | runs, tests, tested by |
| *vacuous*, passes *vacuously* | checks nothing, passes without checking anything |
| an independent *oracle*, no *oracle* for done | an expected value computed by hand, no way to check when it is done |
| *golden* file, *golden* test | the stored expected output, the test comparing against stored output |
| *mutants* died, mutation testing | a deliberate code change was caught by a test |
| *tripwire*, *guard rail*, *safety net* | test, check, warning |
| *smoke test* | say what it checks |
| *hand-computed oracles* | expected values computed by hand |

## Code and failure

| Written | Meant |
|---|---|
| *load-bearing* | matters, the fix depends on it, essential |
| *blast radius* | what else is affected, the extent of the change |
| the websocket *surface*, the config *surface*, a dead *surface* | the websocket interface, the config fields, unused endpoints |
| the error *surfaces*, *surfaced by* | the error is shown, is reported, reported by |
| a *seam*, the DI *seam* | an injection point, a hook |
| *hand-rolled*, *hand-rolls* | written by hand, its own copy, reimplements |
| a *knob* | a setting |
| an *escape hatch* | an override, a way to disable |
| the *happy path* | the successful path, when nothing throws |
| the *hot path* | the frequently called path |
| a *choke point* | the single write path, one place every value passes through |
| *plumbing*, *machinery*, *glue* | the configuration code, the code, the code that passes it on |
| *churn*, *drive-by churn*, *drive-by* changes | unrelated changes, how often it changes |
| *hygiene* | cleanup, bookkeeping |
| *wedged* | hung, unresponsive |
| *flapping* | keeps dropping and returning |
| *busy-spins*, *hot spin*, *pegs a core* | loops without waiting, keeps one core fully busy |
| *fire-and-forget* | with no acknowledgement, a task nobody waits on |
| *surgical* | small |
| *idiom*, *idiomatic*, the house *idiom* | the pattern the repository uses, usual |
| *robust*, *robustness* nits | say what holds, error-handling nits |
| *non-trivial*, *orthogonal*, *first-class*, *cognitive overhead*, *sane defaults*, *leverage* | say what it is, independent, its own, harder to read, reasonable defaults, use |
| *check-then-act*, *TOCTOU* | a check and a write with no lock between them |
| *fail-open*, *fail-closed* | say what happens: continues without the room, rejects the value |
| *rehydrate*, *hydration* | reload, delivery of the stored state |
| *egress* | data leaving the system |
| *hoisting* | moving up |

## Metaphors for code, tests and failure

| Written | Meant |
|---|---|
| *fails loudly*, a *loud* failure | raises an error, is reported |
| *dies silently*, *silently* drops | stops with no log line, drops with no warning; say which |
| *swallows* an error, *swallowed* | catches and ignores, discards |
| *kills* the task, the stream *dies*, a *dead* write | stops the task, the stream ends, a write nothing reads |
| a thread is *parked*, *strands* a message | a thread waits, leaves a message undelivered |
| *leaks* a timer, the key state *leaks* | leaves a timer running, the entries are never removed |
| *rides along*, a *rider*, a *ride-along* | is included, an extra change |
| the fix *lands*, the stack trace *lands* three times | the fix is merged, the stack trace is written three times |
| the value *reaches* the user, no test *reaches* it | the value is shown to the user, no test covers it |
| the fix *turns on*, *rests on*, *hangs on* | depends on |
| *bites* | matters, affects |
| *papers over* | hides |
| the PR *sells*, *buys* nothing, *burns* an hour | the PR claims, has no effect, uses an hour |
| *self-heals* | is corrected by the next message |
| *goes dark*, *gone deaf* | is hidden, receives no notifications |
| *fires* (a diagnostic, a timer) | is shown, runs, starts |
| *arms* a timer, *armed* | starts a timer, enabled |
| *flips* the default, *on the flip* | changes the default, once the default changes |
| *bricks*, *launders*, *cries wolf*, *sharp edges*, *bandaid*, *bolted onto*, *blessed*, *lone straggler*, *belt-and-braces*, *dodge* | breaks, casts, fails on valid input, known defects, workaround, added to, recommended, the only one left, thorough, avoid |
| a *deployment story*, the *privacy story* | the documented deploy path, no private data is exposed |

## Redundant context

| Written | Meant |
|---|---|
| *on the branch*, *anywhere on the branch* | drop it: a review is always of a branch |
| *in this PR* | drop it unless another PR is in play |
| *in-tree*, *in the codebase* | drop it, or name the file |

## Double negation

| Written | Meant |
|---|---|
| *nothing pins* X, *nothing drives* X, X is *asserted by none*, *exercised by nothing* | there is no test for X |
| X *is enforced by nothing* | no code enforces X |
| *no test drives a second* room | there is only a single-room test; a test with a second room is missing |

State the positive fact and what is missing; a sentence the reader has to invert in their head has not been written for them.
