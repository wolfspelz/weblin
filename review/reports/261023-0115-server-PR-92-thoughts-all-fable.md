# Review — server · PR 92 · all · fable

- Change: [Feature/aw thoughts](https://github.com/lupuslabs/nine3q/pull/92)
- Intent: Implements the Thoughts feature: users leave persistent public thoughts and comments in the room they are in, with reactions that drive the sort order (PR description; detail taken from the two design docs the PR adds)
- Branch: `feature/aw_thoughts`
- Runtime: 39 min

## Summary

Verdict: needs-work
- Engineering: needs-work — well built and well tested, but a three-grain wait ring, a faulted config task cached for the WebIt lifetime, and the unmet documentation gate should be fixed before merge.
- Code-review: needs-work — four confirmed bugs: the wait ring, a DataVersion mix-up that stops clients reconciling once a thread passes the comment cap, stale room subscriptions after a silo shutdown, and a destination that is never persisted.
- Scope: over-scope, delivered — the no-client mode for item frames (commit 5a0b6fb9, about 1250 lines) is a second feature with no code dependency from Thoughts; every requested behaviour has a code path.
- CTO: needs-work — the grain layer is the best-guaranteed code this repo has received in a long time, but the merge as it stands adds 30-second stalls, a cluster-wide fan-out on every scoring deploy, note text in a persistent log, a presence leak, and a production-reachable dev harness.

| Stat | Count |
|-|-|
| Files changed | 110 |
| Code files changed | 70 |
| Code files changed (excluding tests) | 59 |
| Code lines delta (excluding tests) | +10015 |
| Scaffolding files changed (non .ts/.cs) | 40 |
| Code files new (excluding tests) | 34 |
| Code files deleted (excluding tests) | 0 |
| Unit test files | 3 |
| Unit test functions/methods | 32 |
| Grain/integration test files | 8 |
| Grain/integration test functions/methods | 227 |
| Docs in `docs/` touched | 0 |

## Short

**What it is.**
A room-agnostic notes layer of four Orleans grains (board, thread, note, per-user reactor) with vote-weighted sorting, culling, notifications and repair, a Thoughts item that `RoomGrain` keeps in the room inventory, a WebIt Razor page with a server-side HTML allow-list, and a 14-file TypeScript single-page app with a TipTap editor.
The grain layer is pinned by 227 grain tests on a real silo against Azurite, the page by 24 HTTP tests with real grains, and both come with substantial design docs inside the repo.
A third commit adds a "no-client mode" that emulates the extension's client API whenever an item-frame page is not framed.

**What turns the call.**
- A wait ring Board → User → Room → Board: the board awaits `IUser.SendSystemInstantMessage` while holding its turn (`NoteBoardNotificationSender.cs:347`), the user grain awaits the room, and the room's `Maintain` awaits the board.
  When a notified user is entering or leaving the room whose board is sending, three activations freeze for the 30 s Orleans timeout, and that room's chat and presence freeze with them.
- Clients stop seeing other users' changes once a thread passes 300 comments: `GetComments` returns the thread's own `DataVersion` (`NoteBoardGrain.cs:1068`), the page mixes it into the board version with `Math.Max` (`Thoughts.cshtml.cs:176`), and the thread bumps twice per culled comment, so the client's watermark runs ahead of the board it polls.
- One transient silo failure on the first Thoughts request after a WebIt start leaves the page broken until WebIt restarts (`Thoughts.cshtml.cs:108-117`).
- Every thought, edit and reaction is written to the persistent request log as "user U said X on page R" (`ItemFrameModel.cs:59-73`), and every author's live online status and language are shipped to anyone who opens any room board (`ThoughtsWire.cs:77`).
- The first request after a scoring-coefficient change, and every post on a full board, do unbounded sequential grain fan-outs inside one user request (`NoteBoardGrain.cs:657-661`, `NoteThreadGrain.cs:525-581`).
- No document in the weblin `docs/` folder covers any of the touched areas; the PR's own docs live in the repo's `Docs/Features/`, which the gate does not name.

**Coverage caveats.**
The SPA has no test runner in the repo, so its behaviour rests on reading alone.
Nine useless-test shapes were found among the 259 changed test methods; the count is above the five-finding threshold, so it is called out here, but it is under 4 % and does not change the verdict.
The very large test files were sampled, not read line by line; the Coverage section says what was read.

## Findings

- major · `Items/n3q.Grains/NoteBoardNotificationSender.cs:347` · Board → User → Room → Board wait ring freezes three activations for the Orleans timeout.
- major · `Items/n3q.Grains/NoteBoardGrain.cs:1068` · thread `DataVersion` passed through to the client; polling stops detecting changes once a thread passes the comment cap.
- major · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:108` · a faulted `NotesConfig` task is cached for the WebIt process lifetime.
- major · `Items/n3q.Grains/NoteBoardGrain.cs:657` · scoring-config drift rebuild fans out to every thread and every note inside one user request.
- major · `Items/n3q.Grains/NoteThreadGrain.cs:525` · post culling reads and deletes up to 300 notes sequentially inside the `CreatePost` turn.
- major · `Items/n3q.WebIt/ItemFrameModel.cs:59` · note text, reaction ids and room id of every write land in the persistent log next to the user id.
- major · `Items/n3q.WebIt/Pages/ItemFrame/ThoughtsWire.cs:77` · authors' `OnlineStatus` and `Lang` are shipped to any authenticated user who opens any board.
- major · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/itemFrame.ts:42` · no-client mode auto-activates for every unframed item frame; untested, undocumented, beyond the request.
- major · documentation gate · no document in `C:/Heiner/weblin/docs/` covers the notes grains, room items, item frames, the WebIt SPA or `ClientConfig`.
- minor · `Items/n3q.Grains/ThoughtsItemManager.cs:37` · subscriptions left behind by a silo shutdown are never removed; the doc says they are.
- minor · `Items/n3q.Grains/NoteBoardGrain.cs:629` · `SetDestinationIfUnset` changes state in memory only; the externally adopted destination is lost on deactivation.
- minor · `Items/n3q.WebIt/ItemFrameModel.cs:126` · `OnPostGetOwnItems` lists whatever inventory the context names (pre-existing exposure, maintainer already noted it).
- minor · `Items/n3q.Grains/NoteBoardGrain.cs:884` · three bare, unlogged `catch` blocks silently drop notifications.
- minor · `Items/n3q.Grains/NoteBoardGrain.cs:320` · `LastNoteId` advances in memory before the write; a failed write re-issues an id after reactivation.
- minor · `Items/n3q.Grains/NoteBoardGrain.cs:982` · a post whose thread is gone is rendered to everyone as a ghost note with id 0.
- minor · `Items/n3q.Grains/ThoughtsItemManager.cs:112` · a failing board subscribe is retried on every client message with no backoff.
- minor · `Items/n3q.Grains/NoteThreadGrain.cs:364` · `CreateComment` accepts a soft-deleted parent; only the client hides the Reply button.
- minor · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsNoteView.ts:386` · a moderator-edited note shows no Reply or Delete although the grain permits both.
- minor · `Items/n3q.Grains/NoteBoardNotificationSender.cs:336` · the `reaction` notification names the reactor, including downvoters, while the UI shows anonymous counts.
- minor · `Items/n3q.Grains/NoteBoardNotificationSender.cs:342` · deleted text is forwarded as `originalText` and persisted in instant-message grains after the author deleted it.
- minor · `Items/n3q.Grains/NoteThreadGrain.cs:1514` · per-user reaction rows outlive the note and nothing offers user-level erasure.
- minor · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsDraftStore.ts:15` · unsent drafts are stored without a user key and never expire.
- minor · `Items/n3q.Items/Templates/Thoughts.cs:81` · window geometry and frame URL suffix duplicated in `ConfigController.cs:16`.
- minor · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:365` · sanitize-then-length gate duplicated in `NotesCommandlineCommands.cs:137`; the seven moderator commands are untested.
- minor · `Items/n3q.Grains/NoteBoardGrain.cs:360` · the per-user reactions LRU cache is implemented twice with diverging semantics.
- minor · `Items/n3q.Grains/NoteThreadGrain.cs:1314` · `"upvote"`/`"downvote"` literals in scoring, config and the SPA.
- minor · `Items/n3q.WebIt/AppPageModel.cs:86` · a 60-entry import map lives in the base model of every WebIt page and is stat'ed on every render.
- minor · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsEditor.ts:262` · the toolbar and source-view regex hard-code the vocabulary the server serves "once".
- minor · `Items/n3q.Grains/NoteBoardNotificationSender.cs:306` · `EnqueueImmediateIm` sends; `GetOrFetchUserReactions` and `GetPostOrThrow` write; one concept spelled `ReactionMeaningCounts`/`meaningCounts`/`MeaningCounts`; `NoteCountObservers` for post-count observers.
- minor · `Items/n3q.Grains/NoteThreadGrain.cs:405` · the cascade-delete loop and the post-order DFS are each copied three times; `GetPosts` is an 86-line method.
- minor · `Items/n3q.WebIt/wwwroot/css/default/itemframe.css:2273` · 746 lines of client CSS for windows no item frame renders.
- minor · `Items/n3q.WebIt/Scripts/is.ts:70` · `stringsObject` deprecated while all three callers stay on it.
- minor · `Items/n3q.WebIt/Pages/ItemFrame/Shared/_Layout.cshtml:5` · a referrer policy added to every item frame with no stated reason.
- minor · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:67` · the doc says the registry rebuilds once on a miss; the code never does.
- minor · `Items/n3q.GrainInterfaces/INoteBoard.cs:10` · the observer doc block names a nonexistent method and the wrong firing rule.
- minor · both design docs, "File-Level Implementation Plan" · the file tables describe a plan, not this diff.
- minor · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsModel.ts:412` · the comments-reconcile anchor and preview rules are pinned by nothing and absent from the doc.
- minor · `Items/n3q.GrainTests/RoomGrainTest_Notes.cs:51` · the room lifecycle is pinned for enter and callbacks, never for leave or deactivation.
- minor · `Items/n3q.GrainTests/NoteBoardGrainTest.cs:1090` · `NotesConfigMatchesSiloConfig` is tautological; `StateSurvivesDeactivation` (`:1250`) never deactivates.
- minor · tight `new()` on 126 added lines against the C# style rule, split 50/50 with the compliant `new ()`.
- minor · `Items/n3q.Grains/NoteGrain.cs:88` · untyped `throw new Exception`.
- minor · `Items/n3q.Grains/NoteBoardGrain.cs:129` · `null!` in own code where the field can be constructed in the constructor.
- minor · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:8` · `any` on five lines, unbraced switch cases at `:204-247`, boolean fields that do not read as booleans at `:62-65`.
- minor · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/itemFrame.ts:42` · missing semicolons; `SiteCommunicator.ts:24` keeps legacy `Secs` on an edited line.
- minor · behaviour coverage · no test pins the Thoughts item badge, the changed-profile branch of `UpdateUserProfile`, the new tools helpers, the WebEx `thoughts` config section, `OnPostGetOwnItems`, or a write with `clientRoomId`.
- minor · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml:195` · the base64 context token is embedded in a script through a Razor `@` expression, which HTML-encodes `+`; pre-existing pattern, needs one manual check.
- nit · `Items/n3q.Common/Changes.cs:10` · "No-client mode for item frames" is developer vocabulary, not the user's view.
- nit · `Base/n3q.WebEx/ClientConfig.cs:166` · a `notifications` config section no server code reads; its consumer is the extension repo.
- nit · useless-test shapes in seven further test methods, listed in Detail.
- nit · about twenty naming, comment and whitespace nits, listed in Detail.

## Detail

### Engineering — needs-work: well built and well tested, but the wait ring, the faulted-task cache and the unmet documentation gate should be fixed before merge

**Correctness & risk**

- **major** · `Items/n3q.Grains/NoteBoardNotificationSender.cs:347` with `NoteBoardGrain.cs:698,720-721,748-749,803,868`, `IUser.cs:106`, `User.cs:186`, `UserClients.cs:137-142`, `RoomGrain.cs:121,263`, `ThoughtsItemManager.cs:62,116,130`.
  The board awaits `IUser.SendSystemInstantMessage` inside its own mutation turn and inside timer callbacks registered with `Interleave = false` (`NoteBoardNotificationSender.cs:74-79`), which in Orleans 8.2 hold the grain.
  `SendSystemInstantMessage` carries neither `[OneWay]` nor `[AlwaysInterleave]`; `UserGrain`, `RoomGrain` and `NoteBoardGrain` are all non-reentrant, and `Items/n3q.Silo` enables no call-chain reentrancy.
  A user turn awaits `IRoom.HandleUserRequest` or `HandleClientGone`; the room's `finally { Maintain }` awaits the board's `SubscribePostCountChangedObserver`, `UnsubscribePostCountChangedObserver` or `SetDestinationIfUnset`.
  Why it matters: when a delayed reply or reaction notification for user X fires on board r-R while X is the first user entering or the last leaving room R, Board waits on User X, X waits on Room, Room waits on Board; all three stall until the 30 s response timeout, the room's chat and presence freeze, and the enter fails.
  The authors made the observer callback `[OneWay]` to avoid exactly this class (Notes doc line 451) and `ActivationSubscribeDoesNotDeadlock` (`NoteBoardGrainTest.cs:1784`) covers only that case.
  Merge instead: mark `IUser.SendSystemInstantMessage` `[OneWay]` (its failure is already only logged at `:348-351`; `RegisterInstantMessageForUser` at `IUser.cs:99` already interleaves), or send instant messages from a zero-delay one-shot timer so no mutation turn ever awaits a user grain; add a grain test for the ring.
  `nine3q/CLAUDE.md › Deadlock Prevention`; anti-patterns › Unbounded blocking.
  Verification: survived; the code-review and CTO verifiers confirmed every edge independently.
- **major** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:33,108-117`, consumed at `:93`.
  `GetCachedNotesConfig` memoises a static `Task<NotesConfig>` and never evicts a faulted one; the comment at `:27-32` says a faulted task stays cached by design.
  Why it matters: one transient silo failure or timeout on the first page load after a WebIt start makes every later `OnGetAsync` rethrow until WebIt restarts; the silo recovers, the page does not.
  Merge instead: cache only successful results (on fault, `CompareExchange` back to null and rethrow), or a plain `_notesConfigTask ??=` with a reset on fault, or an app-level cached accessor on `WebItApp`.
  desired-patterns › Fail closed, degrade gracefully; anti-patterns › Hidden temporal coupling.
  Verification: survived.
- **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:884-886`, `NoteBoardNotificationSender.cs:212-214`, `:278-280`.
  Three untyped, log-free catches on the notification path: `catch { return null; }` makes a storage failure indistinguishable from "note deleted", `catch { return; }` drops a reaction notification, and `catch { break; }` in `DispatchToDirectChildren` drops every remaining child's notification.
  Merge instead: `catch (Exception ex)` with `_app.Log.Warning(...)` and `GetLogData()`, then degrade; let `GetNoteOrNull` distinguish unknown from failed.
  anti-patterns › Swallowed errors / silent failure.
  Verification: survived.
- **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:629-636`, caller `ThoughtsItemManager.cs:55-66` from `RoomGrain.cs:61-69`.
  `SetDestinationIfUnset` is a public `INoteBoard` method that sets `State.Destination` in memory and returns `true` without a write; the internal caller `SubscribePostCountChangedObserver` writes afterwards, the external caller never does.
  The room latches `_destination` (`RoomGrain.cs:63`) and never re-sends, so a destination adopted while the room is already subscribed is lost if the board idle-deactivates before an unrelated write, and `GetDestinationOrNull` (`:607-610`) then returns null.
  Effect: the Thoughts header link back to the page is occasionally missing; cosmetic and self-healing.
  `RoomGrainTest_Notes.cs:171-196` drives this exact path but reads from the live activation, so it passes on in-memory state.
  Merge instead: a private non-persisting helper for the subscribe path, and the public method awaits `WriteOrDeleteIfUnused()` when it returns true; extend the test to read the stored state as `NoteBoardGrainTest.cs:682` does.
  desired-patterns › Idempotent, restart-safe operations.
  Verification: survived; the code-review verifier confirmed.
- **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:320-324,687,696,707,718`.
  `MakeNoteId` bumps `LastNoteId` in memory before the thread persists the note; a failed board write leaves memory diverged until deactivation, and after reactivation the id is reused once so `NoteGrain.Initialize` throws "Already initialized!" (`NoteGrain.cs:88`); `Repair` reaps orphans but never raises `LastNoteId`.
  Merge instead: `DeactivateOnIdle()` after a failed `WriteStateAsync`, or persist `LastNoteId` before fanning out.
  desired-patterns › Idempotent, restart-safe operations.
  Verification: survived; self-limiting to one failed user action per reactivation.
- **minor** · `Items/n3q.WebIt/ItemFrameModel.cs:126-131` with `ItemContext.cs:102-145`.
  `OnPostGetOwnItems` opens `IInventory(Context.InventoryId)` and returns every item with `PidSet.All`; the HMAC binds the context to the caller's token but nothing binds `inventoryId` to `userId`, so a user can mint a context naming another user's id as inventory and list that backpack.
  The exposure is pre-existing through `ItemApiController.GetItemIds`, and the maintainer's PR comment says it is not counted against the PR; the CTO lens still rated the new copy major because the endpoint sits on the base class every future frame inherits, and its only caller is the no-client emulation (`WeblinClientApi.ts:105`).
  Merge instead: read `Grains.GetGrain<IInventory>(Context.UserId)` for "own items" (do not reject `InventoryId != UserId` outright, room-item contexts are legitimate), and pin it with a `WebIt.Test`; a central ownership check in `EvaluateContext` covering `ItemApiController` is a separate PR.
  desired-patterns › Fail closed, degrade gracefully.
  Verification: survived; severity merged to minor on the maintainer's stated stance.
- **minor** · `Items/n3q.Grains/NoteGrain.cs:88` · `throw new Exception("Already initialized!")` in new grain code; `:162` of the same file uses `InvalidOperationException`.
  `nine3q/CLAUDE.md › C# Code Style › Failure signaling`.
  Verification: confirmed by rule text.
- **nit** · `Items/n3q.GrainInterfaces/INoteBoard.cs:10,15` and `NoteBoardGrain.cs:597,697,719,867`.
  The observer doc names a nonexistent `SubscribeHasPostsObserver` and says the callback fires only when the count "reached 0 or increased from 0"; the code fires on every post create or delete and on every comment create or delete, where the post count does not change (harmless: `[OneWay]`, and `ThoughtsItemManager.cs:76` dedups), and the log text "HasPosts notification threw" is stale.
  Merge instead: fire only when `Posts.Count` changed and rewrite both doc blocks to the as-built contract.
- **nit** · `ThoughtsController.ts:495-526` · the optimistic reaction toggle reverts to `activeBefore` of this call on failure, clobbering an overlapping toggle until the next refresh; flicker only.

**Scope & blast radius**

- **minor** · `itemFrame.ts:42,55-60`, `WeblinClientApi.ts:22,207,213`, `ItemFrameModel.cs:126`, `_Layout.cshtml:5`, `itemframe.css:29,37,56`, `Changes.cs:10`.
  Commit 5a0b6fb9 "No-client mode" is a second feature; it auto-activates when `window === window.top` for all 19 item-frame pages that construct `ItemFrameUtils`, ships three `Todo` stubs and the `OnPostGetOwnItems` endpoint, and no Thoughts source or doc references it.
  The engineering lens kept this minor because it is a clearly labelled own commit with its own changelog line and the framed path is unchanged; the CTO lens rates it major (see its section) and the scope lens lists it as beyond request.
  Merge instead: cherry-pick the commit into its own PR, or at least name it in the PR description.
  anti-patterns › Gratuitous rewrite / scope creep; desired-patterns › Intentional, called-out refactors.
- **minor** · `Items/n3q.WebIt/wwwroot/css/default/itemframe.css:77-91,2273-3014`.
  The "Default client style" re-sync appends 746 lines that did not exist at BASE (test window, XMPP debug window, tutorial, about window with a 12.5 KB base64 PNG, changes, backpack, persons, chat, settings, toasts), none referenced by any item-frame script; Thoughts needs about 19 lines of tokens.
  Every item frame now downloads about 40 % more CSS and the diff hides the tokens Thoughts relies on.
  Merge instead: trim the copy to the shared token and base sections, or make the sync its own commit; the header at `:77-91` documents a wholesale-copy procedure, so this is a process decision for the human.
  engineering §3 Blast radius; anti-patterns › Drive-by churn.
- **minor** · `Items/n3q.WebIt/Scripts/is.ts:70-78`, callers `WebsocketMessage.ts:43,61,85`.
  `stringsObject` is marked `@deprecated` and reimplemented via a new generic `mapObject<T>`/`stringMapObject`, but all three pre-existing callers stay on the deprecated name and `mapObject` has one caller.
  Merge instead: rename the three callers in the same change, or add `stringMapObject` without deprecating.
  anti-patterns › Drive-by churn, Premature abstraction.
- **minor** · `Items/n3q.WebIt/Pages/ItemFrame/Shared/_Layout.cshtml:5`.
  `<meta name="referrer" content="origin">` is added to every item frame in the no-client commit with no comment, commit message or doc mention; it is a privacy tightening (keeps `?context=` tokens out of the Referer), so the defect is an unexplained global change, not harm.
  Merge instead: a one-line comment stating the reason.
- **nit** · `RoomGrain.cs:249-264` · `Maintain` restructured from early return to nested `if`, so `_roomItems.Maintain` now runs on every processed message; harmless because of its own interval guard (`RoomItems.cs:90-93`), but a silent refactor.
- **nit** · `Base/n3q.WebEx/ClientConfig.cs:166,191`, `ConfigController.cs:31` · the `notifications { enabled }` section has no server reader and no doc mention; its consumer is the extension repo (`ContentInstantMessageManager.ts:48`, `lib/Config.ts:452`); state the cross-repo companion in the PR.
- **nit** · `Items/n3q.WebIt/Scripts/site.ts:175-179` · a genuine shared-helper fix (`clipPath` never applied, string negation yielding `--x`) that also changes the pre-existing `Person.cshtml` avatar frame and is nowhere called out; the scope lens verified Thoughts depends on it through `ThoughtsNoteView.ts:320` → `domImageTools.ts:40`.
- **nit** · whitespace-only churn on untouched lines: `WebItConfigDefinition.cs:46,49`, `ClientConfig.cs:86,165`, `RoomItems.cs:95` (the file's only change), `n3q.WebIt.csproj:17`, UTF-8 BOM removed on line 1 of nine files; `itemframe.css:4` `* {}` → `:root {}` changes the font of ten legacy frames with markup outside `#n3qD`.

**Simplicity**

- **minor** · `NoteBoardGrain.cs:360-393` and `NoteThreadGrain.cs:1571-1606`.
  Two hand-rolled LRU caches (`LinkedList` plus node index plus eviction) for `NotesUserReactionsCacheLimit = 30`, with five "≤ 0 disables" branches whose semantics differ: the board clears all three structures (a branch its callers at `:404,461,479` make dead), the thread returns early; the board moves the node, the thread removes and re-adds.
  Merge instead: one small `LruCache<TKey,TValue>` in `n3q.Tools` or a `UserReactionsLruCache` in `n3q.Grains`, and drop the ≤ 0 branches.
  anti-patterns › Duplicated source of truth; engineering §2.
- **minor** · `ThoughtsEditor.ts:262-276,356` vs `thoughtsSchema.ts:4-8`, `NotesVocabulary.cs:16-17`.
  The vocabulary is served part by part so it is "defined ONCE, server-side", but the toolbar literal and the source-view regex hard-code bold, italic, strike, link, code, code block and both lists independently; dropping a part server-side leaves a button targeting an unloaded extension.
  Merge instead: derive the toolbar from the served parts, or keep a fixed TS constant mirrored by the C# allow-list with a test asserting equality.
  anti-patterns › Premature abstraction.
- **nit** · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:82-103`, sole caller `NoteBoardGrain.cs:579` · a 151-line reflection registry that force-loads every referenced assembly on first observer dispatch, for one observer type; the design doc gives a sound reason, so an observation ("heavier than a manifest lookup"), not a defect.
- **nit** · `NoteBoardGrain.cs:182-203,1127-1130` · SHA-256 `ReactionConfigHash` of a 30-character canonical string compared once in `GetStats`, while the sibling scoring drift check value-compares the record; the canonical string itself would be cheaper and debuggable.
- **nit** · `NoteThreadGrain.cs:1234-1303` · incremental best-descendant maintenance with a `NegativeInfinity` sentinel and a subtree-scan fallback where a full bottom-up rebuild over ≤ 300 comments per mutation would cost microseconds; documented design, so a simplification offer.
- **nit** · `INoteBoard.cs:67` `HasPosts()` and `INoteThread.cs:89` `GetDataVersion()` · public grain-interface methods with no production caller (tests only); remove until a consumer exists.
- **nit** · `domSchedulingUtils.ts:9` `waitForRenderComplete` has no caller; `Templates/Thoughts.cs:47-58` a commented-out "Debug grid" sits inside the SVG that is base64-encoded into every Thoughts item's `ImageUrl`; `ThoughtsItemManager.cs:30-34` `Init` returns `Task.CompletedTask` for a field assignment.

**Naming & clean code**

- **minor** · `NoteBoardNotificationSender.cs:306-350` · `EnqueueImmediateIm` enqueues nothing; it awaits `IUser.SendSystemInstantMessage` at `:347`. Merge instead: `SendNotificationIm`.
- **minor** · `NoteThreadGrain.cs:1514-1542` · `GetOrFetchUserReactions` is a getter that writes (`reactor.RemoveNotes(...)` at `:1536`) and is reached from every comment read. Merge instead: split `DropStaleUserReactions` out of the read path. `CLAUDE.md › General Code Style` ("`Get` promises side-effect-freedom").
- **minor** · `NoteThreadGrain.cs:985-997` · `GetPostOrThrow` mutates persisted state (`State = State with { Post = post }` at `:995`) as an adoption side effect across five call sites (`:171,363,387,408,497`). Merge instead: `AdoptPostIfMissing()` once, a pure getter otherwise.
- **minor** · `INote.cs:40,63`, `ThoughtsWire.cs:36`, `ThoughtsModel.ts:47` · one concept, three spellings: `ReactionMeaningCounts` (persisted), `meaningCounts` (parameter), `MeaningCounts` (wire and doc). Merge instead: `MeaningCounts` everywhere; the type is new, so no migration.
- **minor** · `NoteBoardGrain.cs:51,574` vs `:541,564` · `NoteCountObservers`/`InformNoteCountChangedObservers` next to the public `SubscribePostCountChangedObserver`; notes are not posts in this domain, and the doc's "earlier name kept for deserialization" cannot refer to production data for a grain new in this PR. Merge instead: `PostCountObservers`/`InformPostCountChangedObservers`.
- **minor** · `NoteThreadGrain.cs:405-474` vs `:602-662` · the "delete note grain → `UnregisterEntry` → walk up removing soft-deleted childless ancestors" loop appears twice, and the identical `"NoteGrain.Delete failed"` try/catch/log seven times. Merge instead: one `RemoveNoteAndChildlessSoftDeletedAncestors` and one `TryDeleteNoteGrain`.
- **minor** · `NoteThreadGrain.cs:668-691,1154-1181,1188-1220` · three copies of the same `Stack<(long Id, bool IsVisited)>` post-order DFS differing only in the per-node action. Merge instead: one `ForEachCommentPostOrder(Action<long>)`.
- **minor** · `NoteBoardGrain.cs:955-1040` · `GetPosts` is 86 lines that sort, page, fill caches, enrich authors (re-implementing `EnrichAuthors` at `:1092-1105` over a different element type), fetch reactions and build the truncation marker, with single-letter locals. Merge instead: reuse `EnrichAuthors`, extract `GetVisiblePostStates`, spell locals out.
- **nit** · `ThoughtsItemManager.cs:9,43-53,68-74` · the only `*Manager` in the grain layer beside the plain-noun sibling `RoomItems`; `SetPostCount` also rewrites the inventory; `Maintain(now)` ignores `now`; `UpdateThoughtsItem` is `internal` with no external caller.
- **nit** · `MoveUserReactionsToLruFront`/`MoveUserToLruFront` both `AddLast`; `Thoughts.cshtml.cs:375` `CheckAuthor` returns `IActionResult?`; `NoteBoardGrain.cs:233` private `Delete()` clears state and deactivates; `ThoughtsModel.ts:3` cites a nonexistent `INoteBoardGrain`, `:55` SPA `NoteState` differs from the server `NoteState`, `:89` `NotesConfigEntry` is the whole config; `NoteBoardGrain.cs:278` region label "NoteBoardNotifier" stale; `NoteBoardNotificationSender.cs:208` "pae revert check" unexplained; `DomUtils.ts:21` "where introduced" → "were"; `ThoughtsReactionsView.ts:100-107` dead re-checks of `config`; `WeblinClientApi.ts:56` `ItemHandling` → `OwnItemsPoller`; `NotesVocabulary.cs:21-22` `SchemaPart` is a key.

**Engineering principles**

- **minor** · `Items/n3q.Items/Templates/Thoughts.cs:81-90` ↔ `Base/n3q.WebEx/Controllers/ConfigController.cs:16-25`.
  The Thoughts window geometry (600×700, bottom 150, hidden, resizable) and the `Thoughts?context={context}` suffix are typed twice with reciprocal "update the other" comments; WebEx and WebIt deploy separately, so the room item and the participant-menu entry can open different sizes; both projects reference `n3q.AppInterfaces`.
  Merge instead: a small record in `App/n3q.AppInterfaces` consumed by both (the URL bases legitimately differ), or a test asserting equality.
  anti-patterns › Duplicated source of truth.
- **minor** · `Thoughts.cshtml.cs:365-373` and `NotesCommandlineCommands.cs:137-145`.
  The sanitize → `NotesMaxTextLength` → `ArgumentException` block is copied; the policy's own header says it runs "before the length check", so the length gate is the part that drifts, and the seven moderator commands have no test, so the page tests that pin the gate do not protect the moderator path.
  Merge instead: `NotesHtmlPolicy.SanitizeAndCheckLength(text, maxLength)` plus one commandline test per command family through `ItemCommandline`.
  anti-patterns › Duplicated source of truth; test.md › Meaningful unit tests.
- **minor** · `NoteThreadGrain.cs:1314,1316`, `SiloConfigDefinition.cs:107-108`, `ThoughtsModel.ts` vote column.
  `"upvote"`/`"downvote"` literals in scoring, config defaults and the SPA; scoring silently yields 0 if an operator renames the meanings (the doc says so at line 627, the code does not).
  Merge instead: `public const string UpvoteMeaning`/`DownvoteMeaning` beside `NoteReactionConfig`, served to the SPA with `notesConfig`.
  engineering §5 Single source of truth.
- **minor** · `Items/n3q.WebIt/AppPageModel.cs:60-152,183-192` with `Scripts/lib-shims/*.d.ts`.
  A 60-entry tiptap/ProseMirror import-map manifest with a four-step manual update procedure lives in the base page model of every WebIt page, and `File.Exists` plus `GetLastWriteTimeUtc` run per entry per render of every page through both layouts; 174 lines of hand-written ambient types must track the bundles.
  Vendoring is the repo's existing practice, so the finding is placement and per-page cost.
  Merge instead: an own static class or JSON next to `wwwroot/lib`, computed once (`Lazy<>`), and a script that produces the bundle set and alias map from one version list.
  engineering §5 Separation of concerns; anti-patterns › Duplicated source of truth.
- **nit** · `Thoughts.cshtml:10-38,91+` tooltip keys duplicate the reaction ids of `SiloConfigDefinition.cs:106-150` with no drift check; `ThoughtsWire.cs:26-87` ↔ `ThoughtsModel.ts:21-91` hand-mirrored DTOs with stale comments and no serialisation-shape test; `NoteBoardNotificationSender.cs:273,277` magic `pageSize = 100` and user id `"system"`; event names as bare strings.

**Project conventions**

- **major (per rule)** · documentation gate.
  `C:/Heiner/weblin/docs/` holds only the README and four documents on avatar upload, vidconf, backpack rendering and GUI styling; none covers the Notes grains, `RoomGrain` room items and `ThoughtsItemManager`, the `ItemFrameModel` page-call handlers, the WebIt item-frame SPA (`itemFrame.ts`, `WeblinClientApi.ts`, the import map), `SeparateJsonAzureTableStorageProvider.GetRecordsByRowKeyRange`, or WebEx `ClientConfig`/`ConfigController`.
  What the document must say, at minimum: the board → thread → note/reactor layering and the key derivation `{boardId}:{postId}[:{noteId|userId}]` with the colon-free invariant; the storage layout (PartitionKey = boardId, empty RowKey for the board); the `INoteBoardPostCountChangedObserver` plus `GrainInterfaceRegistry` contract with its persisted `(InterfaceFullName, GrainKey)` form; the five activation-scoped caches; the three `Repair` levels and the never-used-state cleanup; `RoomGrain` as sole writer of the Thoughts item and the `Maintain` subscribe/unsubscribe cycle; the page model → `ThoughtsWire` string-id boundary → SPA module graph; the one-vocabulary-two-gates HTML policy; the import-map vendoring procedure; the no-client mode; the two places defining the Thoughts frame URL and options.
  Observation for the human, not resolved here: the PR ships two substantial design docs inside the reviewed repo (`Docs/Features/Notes/…`, `Docs/Features/Thoughts/…`), referenced from source (`SiloConfigDefinition.cs:5,103`, `NoteBoardGrainTest.cs:1`, `NoteThreadGrain.cs:1`), and five spot checks (placement attributes, subscribe call, limits tables, row-key range, reactor persistence) all match the code.
  The gate names the weblin root `docs/`, the repo uses `Docs/Features/`; this is rule-versus-practice drift for the rule owner to settle, and the PR docs end in plan-tense "File-Level Implementation Plan" sections and do not follow `docs/README.md` conventions, which matters only if they move.
  `weblin/CLAUDE.md › Hard Rule: No Undocumented Area Is Edited`; `criteria/project.md › The documentation gate`.
  Verification: survived by rule text.
- **minor** · tight `new()` on 126 added lines against 127 spaced `new ()`, so the PR is split 50/50 against the rule.
  Representative: `NoteBoardNotificationSender.cs:49-51`, `NoteGrain.cs:15,91`, `NotesCommandlineCommands.cs:20,33`, `Thoughts.cshtml.cs:19`, `SiloConfigDefinition.cs:107` (26 times in the reaction table), `ConfigController.cs:32` directly under a compliant `:31`, `ItemFrameModel.cs:136,140`, `ThoughtsPageTest.cs` (61 times), `NoteBoardGrainTest.cs:38,40`; `NoteBoardGrain.cs` and `NoteThreadGrain.cs` are fully compliant.
  Merge instead: `new ()` throughout.
  `nine3q/CLAUDE.md › C# Code Style` ("the tight `new()` is legacy").
- **minor** · `NoteBoardGrain.cs:129` · `_notificationSender = null!; // Set in OnActivateAsync`; the sender constructor (`NoteBoardNotificationSender.cs:53`) needs only `(board, app, grainFactory)`, all available in the grain constructor (`:139-145`), so the field can be `readonly` and constructed there.
  `› C# Code Style` ("own-code null flow is proven to the compiler, not forgiven").
- **minor** · missing semicolons on added lines `WeblinClientApi.ts:101`, `itemFrame.ts:42,43,45,61`, `site.ts:175,179`. `› TypeScript Code Style › Formatting`.
- **minor** · `SiteCommunicator.ts:24` · `timeoutSecs?: Nil|number` on an edited line keeps the legacy `Secs`. `› General Code Style` ("plural `Secs` is legacy"; "converted when touched").
- **minor** · `WeblinClientApi.ts:8,27` (moved from `itemFrame.ts`, so touched) and `:68,72,199` (new) · `any` in a new file; the class already narrows with `is.array(ownItems, elem => is.stringMapObject(elem))` at `:110`, so `unknown` is reachable. `› Types & modules`.
- **minor** · `WeblinClientApi.ts:62-65` · private fields `handleOwnItems`, `inOwnItemsUpdate`, `ownItemsUpdateRequested`, `notifyAboutAllOwnItems` do not read as booleans (the imperative exception is for option objects only). Merge instead: `shouldHandleOwnItems`, `isOwnItemsUpdateRunning`, `isOwnItemsUpdateRequested`, `shouldNotifyAboutAllOwnItems`.
- Mirror rule: not applicable, `CLAUDE.md` is not in the diff.
- Changelog rule: mechanically compliant; the deployed top release on `origin/deployment` is "49", both entries sit in the existing higher release "50", and "Users can leave persisted public thoughts and comments for the current room" is well formed.
- **nit** · `Changes.cs:10` · "No-client mode for item frames" is developer vocabulary; it is arguably user-visible (an item-frame URL opened directly in a tab now renders instead of breaking), so reword as an effect or drop it if dev-only.
- Observation, pre-existing: release "50" has an empty name (`Changes.cs:9`) though the rule asks for a short PascalCase name.
- Observation, rule versus code: `› C# Code Style` prescribes `App.Log.SetValue(LogData.Key.ObjectId, …)` in `OnActivateAsync`, but `ICallbackLogger` exposes no `SetValue` and nothing in the repo calls it; the PR passes `LogData` per call (`NoteBoardGrain.cs:146-157`, with a comment saying so); the rule names an API the interface lacks, for the rule owner. Related nit: `ThoughtsItemManager.cs:17` creates its own `_logData` and never sets `ObjectId`, so its error logs carry no room identity.
- **nit** · `ThoughtsItemManager.cs:11` `_thoughtsItemSearchProperties` is `static readonly` in `_camelCase` (rule: PascalCase); about 35 new non-test C# lines over the soft 120-column limit; `AppPageModel.cs` still lacks a final newline though touched; `ThoughtsModel.ts:283` `callListeners(…, undefined)`; `ThoughtsController.ts:38-60` public fields interleaved among privates.

**Useless-test shapes** (`criteria/test.md`; confirmed by shape, not refute-dropped; nine methods)

- **minor** · `NoteBoardGrainTest.cs:1090` `NotesConfigMatchesSiloConfig` · Constant / config assertion, tautological: expected is `SiloConfig.NotesReactions`, and `GetNotesConfig` (`NoteBoardGrain.cs:1117-1123`) returns `_app.Config.NotesReactions` unchanged. Delete, or rewrite as a serialization round-trip with a hand-built config.
- **minor** · `NoteBoardGrainTest.cs:1250-1267` `StateSurvivesDeactivation` · no deactivation step; its own comment at `:1261` admits the re-fetch "may or may not reactivate the same instance"; duplicates `StatsMirrorState` (`:1113`); `INoteBoard` exposes no `Deactivate`. Add the `IDeactivatableGrain` hook the other note interfaces have, or rewrite to read the row with RowKey "" (the class name shows that intent), or delete.
- **nit** · `NoteGrainTest.cs:160` `GetReturnsPersistedState` · Store-and-retrieve round-trip on the same activation; `InitializePersistsSuppliedValuesWithOriginalStatus` (`:122`) already pins the fields. Delete, or insert `Deactivate()` so the name is true.
- **nit** · `NoteGrainTest.cs:145` `InitializeSetsParentIdForComment` · Mirrors a trivial line (`ParentId = parentId`, `NoteGrain.cs:98`). Fold into `:123` or delete.
- **nit** · `NoteGrainTest.cs:763` and `NoteThreadUserReactionsGrainTest.cs:321` `DeleteIsIdempotent` · Assertion-free: no `Assert` after the second `Delete()`. Idempotent delete is a plausible contract, so replace with an observable assertion (row absent, state empty) rather than delete.
- **nit** · `INoteBoardTest.cs:13,21,29` `Accepts*` · Assertion-free (one call under "// Act & Assert"); `:53-80` four colon tests mirror one guard line (`INoteBoard.cs:120`); a guard branch is a kept category, so collapse to one `DataRow` case.
- **nit** · `NoteThreadGrainTest.cs:573-597` `StatePersistsAcrossReactivation` · ends in the constant assertion `Assert.AreEqual(SiloConfig.NotesScoring, persisted.ScoringConfig)`; keep the structural asserts, drop that line.
- **nit** · `NoteBoardGrainTest.cs:261` and `NoteThreadGrainTest.cs:114` · Constant / default assertion (`DataVersion == 0` on a fresh grain). Merge into the adjacent rejection tests or delete.
- Refuted: `NoteBoardGrainTest.cs:703` `ObserversRoundTrip` pins the own `JsonDefaults.cs:13 IncludeFields = true` line protecting ValueTuple persistence, so an independent oracle exists.

**Behaviour coverage**

Pinned, with the production symbol each test drives: post create, edit and delete (`NoteBoardGrainTest.cs:305,389,428,480` → `CreatePost:678`/`AuthorEditPost:771`/`AuthorDeletePost:807`; `ThoughtsPageTest.cs:403,702,816` → the three page handlers over real HTTP with a signed context); comments (`NoteBoardGrainTest.cs:458`, `NoteThreadGrainTest.cs:200-367`; page `:739,778,854`); reactions and scoring (`NoteBoardGrainTest.cs:1033,1204,1896`, `NoteThreadGrainTest.cs:682,1435`, `NoteThreadUserReactionsGrainTest.cs:138,159,179` → `SetReactions:98`; page `:542,590,631` → `OnPostSetReactionsAsync:312`); culling (`NoteBoardGrainTest.cs:1935,1975,2007` → `CreatePost:683`/`CullLowestPost:725`; `NoteThreadGrainTest.cs:1340,1380` → `CullOne:602`); RoomGrain observer and Thoughts item (`RoomGrainTest_Notes.cs:53-247` → `RoomGrain.OnNoteBoardPostCountChanged:70` → `ThoughtsItemManager.SetPostCount:68` via real `RoomEnterRequest`s and `INoteBoard.CreatePost`); all twelve page handlers over HTTP including the 403/404 paths of `CheckAuthor:375` (`ThoughtsPageTest.cs:471,896`), the length cap (`:436`), the preview merge (`:135`) and the data version (`:23`); HTML policy (`NotesHtmlPolicyTest.cs:18-118` → `NotesHtmlPolicy.Sanitize`, write path pinned by `ThoughtsPageTest.cs:403,778`); notifications end to end on the real client stream (`NoteBoardGrainTest.cs:2055-2603` → `NoteBoardNotificationSender.OnNoteCreated:63`); `GrainInterfaceRegistry` (`GrainInterfaceRegistryTest.cs:14,27,41` plus eviction at `NoteBoardGrainTest.cs:1613,1664,1704`).
Not applicable: the TypeScript SPA, no-client mode, `Thoughts.cshtml` and the CSS; `Items/n3q.WebIt/package.json` has lint scripts only and no test runner exists.

- **minor** · `Items/n3q.Items/Templates/Thoughts.cs:9` `MakeThoughtsItemCreateOrUpdateProperties` and `ThoughtsItemManager.cs:105` · no test asserts any Thoughts item property (badge, `ImageUrl`) or the 1→2-post update; `RoomGrainTest_Notes.cs` only counts items. Add a second post after `PostCountChangeAddsAndRemovesItem` and assert `Pid.ImageUrl` changed, plus an `n3q.Items.Test` case on the `>999` badge text.
- **minor** · `NoteBoardGrain.cs:350` `UpdateUserProfile` changed-profile branch (the PR's only `IUser.cs` change, `HasSameIdAndNameAndAvatar`) · only the no-op branch is tested (`NoteBoardGrainTest.cs:1156`); inverting the comparison would stay green.
- **minor** · `Shared/n3q.Tools/StringExtensions.cs:38` `ParseOptionalLong`, `DateTimeExtensions.cs:18` `ToUnixTimeSeconds` · `Shared/n3q.Tools.Test` untouched; add cases in the existing `StringExtensionsTest.cs`/`DateTimeExtensionsTest.cs`.
- **minor** · `Base/n3q.WebEx/Controllers/ConfigController.cs:16-34` · the new `thoughts` and `notifications` sections have zero assertions in `Base/n3q.WebEx.Test`; the extension hard-depends on `thoughts.itemFrameProperties.IframeUrl` containing `{context}`.
- **minor** · `ItemFrameModel.cs:126` `OnPostGetOwnItems` · no test references the handler; the same test is where the ownership guard would be pinned.
- **minor** · `Thoughts.cshtml.cs:224-312` write handlers with `clientRoomId` · the override is tested on GETs only (`ThoughtsPageTest.cs:335,363`); add one `CreatePost` with `clientRoomId` and read the override board.
- **nit** · `AzureCloudTableExtensions.cs:39` / `SeparateJsonAzureTableStorageProvider.cs:190` `GetRecordsByRowKeyRange` · no direct test of the half-open range; the sole production caller's end key never equals a row key, so add one case beside `GetRecordsByPartitionKeyRange` (`SeparateJsonAzureTableStorageTableGrainTest.cs:111`).

**Positives.**
Type-placement rule followed throughout; every persisted state and wire record is a `sealed record` with `required`/`init`; file-scoped namespaces, no `out` parameters, no `Async` suffix on own methods, `[TestCategory]` on all 259 test methods.
The board → thread → note/reactor call direction is strictly downward, the observer callback is `[OneWay]` with eviction of unresolvable observers, and `ActivationSubscribeDoesNotDeadlock` is a real deadlock regression test.
`NotesHtmlPolicy` is allow-list based and applied on every write path; the vocabulary is single-sourced into the SPA schema, which fails loud on an unknown part.
`NoteBoardGrainTest_FullBoardLifecycle.cs:57` is a model-based test with an independent in-test oracle compared after every mutation and after deactivating every grain; the `*_Repair` tests plant corrupt rows straight into storage; notification tests observe the real `IUser` client stream.
`ThoughtsPageTest.cs` pins authorisation with read-back (403 and unchanged note) and the length cap with read-back.
Scaffolding is surgical (`TemplateRegistry.cs` +1, `ItemCommandline.cs` +2, `Changes.cs` +2), every new config knob is read by production code, and the SPA layering is clean (`ThoughtsApi` stateless I/O, `ThoughtsModel` passive store, `ThoughtsController` orchestration, views with `isStopped`/`stop()`).

### Code-review — needs-work: four confirmed bugs, one of them client-facing and permanent on busy boards

Run at effort `high`, retargeted at the pinned diff: two conventions finders (C# and TypeScript), one diff-only bug finder, one in-context cross-file finder, then one verifier per candidate (PLAUSIBLE by default, REFUTED only when constructible).
Findings that duplicate a line already confirmed by another lens are marked as merged.

- **major** · CONFIRMED · `Items/n3q.Grains/NoteBoardGrain.cs:1068,1079,511-531` with `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:163,176,182,200-201,213-214` and `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsModel.ts:291-292,555,561-565`.
  `GetComments`, `GetNoteWithAncestors` and the comment-preview cache return `inner with { Authors = authors }`, so the page carries the thread grain's own `DataVersion`; the page model folds it into the board version with `Math.Max`, and the two comment handlers return it directly through `NotesPageWire.From` (`ThoughtsWire.cs:85`).
  The SPA keeps one watermark that only ever rises (`updateDataVersionIfNewer` accepts `version > this.dataVersion`) and polls the board-only `OnGetDataVersionAsync` (`:220-221`), so the two counters are compared as if they were one.
  At the comment cap (default 300, `SiloConfigDefinition.cs:105`) `CreateComment` calls `CullOne`, which ends in `CommitComments` and a write (`NoteThreadGrain.cs:659-660`), and then calls `CommitComments` again (`:379`); each call does `DataVersion + 1` (`:1131-1136`), while the board increments once (`NoteBoardGrain.cs:717`), so the thread gains one on the board per culled comment and nothing ever closes the gap.
  Concrete sequence: a post at 300 comments has board and thread at V; k more comments put the thread at V+2k and the board at V+k; a client that loads the frame stores V+2k, its polls return V+k, V+k+1, …, none count as newer until the board passes V+2k, and no other user's post, comment, edit, delete or reaction is reconciled until then.
  Who is hurt: everyone viewing a board where any thread has passed the cap, which are the busy rooms; on a long-running hot thread it is effectively forever, and a reload picks up the inflated value again.
  No test pins it: `NoteBoardGrainTest_FullBoardLifecycle.cs:321` checks `GetPosts` only, `NoteThreadGrainTest.cs:210,276` check +1 per thread operation but not the cull path, `ThoughtsPageTest.cs:23-57` checks the board-only handler with 0 and 1.
  Merge instead: in `GetComments`, `GetNoteWithAncestors` and `MakePageFromCommentPreview` return `inner with { Authors = authors, DataVersion = State.DataVersion }`, so every page carries the board's version; the `Math.Max` at `Thoughts.cshtml.cs:176` then becomes a no-op to remove; add a grain test asserting that the `GetComments` version equals `GetDataVersion()` after a cull.
  Note: the CTO lens's verifier dropped this candidate as impossible "by construction"; the orchestrator re-read `NoteThreadGrain.cs:372-379` and `:659-660` and confirms the double bump, so the finding stands.
- **major** · CONFIRMED · `Items/n3q.Grains/NoteBoardNotificationSender.cs:347` · the Board → User → Room → Board wait ring; merged with the engineering major above.
  The verifier adds: Orleans is 8.2.0 for every package (`Directory.Packages.props:25-31`), `Items/n3q.Silo` contains no reentrancy or response-timeout options, call-chain reentrancy would not help because the three calls are three separate chains (a timer, a client request, a client request), and `UserGrain.HandleClientGone` → `IRoom.HandleClientGone` → `Maintain` is a second entry point for the user → room edge.
- **minor** · CONFIRMED · `Items/n3q.Grains/ThoughtsItemManager.cs:19,37,49` with `Items/n3q.Grains/RoomGrain.cs:42-52,71-74,121,135` and `Items/n3q.Grains/NoteBoardGrain.cs:224,574-601`.
  A subscription left behind by a silo shutdown is never removed, contrary to the Thoughts doc (lines 88 and 121: "the board's next callback activates the room; its next sweep unsubscribes if nobody is there").
  `_hasSubscribedToNoteBoard` is an in-memory flag that starts false on every activation; `Stop` skips the unsubscribe on `ShuttingDown` as designed; `InformNoteCountChangedObservers` evicts only observers whose interface no longer resolves, so it re-activates the room; `OnActivateAsync` only calls `Init`; the callback only calls `SetPostCount` and `UpdateThoughtsItem`; `Maintain` runs only from `ProcessUserRequest` and `HandleClientGone`, and even then its unsubscribe branch requires the flag.
  Effect: nothing visible to users; after every shutdown, crash or deploy, every occupied room leaves one stale entry; boards with posts re-activate that empty room and write a `SetItemProperties` on every create or delete (the "last applied" count is null on a fresh activation, `ThoughtsItemManager.cs:76`) until someone enters; boards that never had posts never self-heal, the stale entry keeps the board row alive forever, and `isUnused = … && State.NoteCountObservers.Count == 0` (`NoteBoardGrain.cs:224`) blocks the never-used-state cleanup; a slow storage leak across the very large room space.
  No test pins the shutdown behaviour: `RoomGrainTest_Notes.cs:135-166` uses a normal `Deactivate()`, `NoteBoardGrainTest.cs:1789` simulates a stale entry only to test that the board does not deadlock, and no test mentions `ShuttingDown`.
  Merge instead: in `RoomGrain.OnNoteBoardPostCountChanged`, when `GetUserCount() == 0`, call `UnsubscribePostCountChangedObserver` without checking the flag (the board's unsubscribe is idempotent, `:567`) and skip the inventory update; add a test that subscribes, deactivates with a shutdown reason (or injects the stale entry), creates a post, and asserts `NoteCountObservers` ends up empty.
  Adjacent, unconfirmed: `UpdateThoughtsItem` always issues an inventory `Modify` on the first callback after an activation even when the item already carries that count, and a failed subscribe is retried on every heartbeat with nothing throttling it (the CTO lens files the latter separately).
- **minor** · CONFIRMED · `Items/n3q.Grains/NoteBoardGrain.cs:629-636` · `SetDestinationIfUnset` never persists; merged with the engineering minor above.
  The verifier adds: `SetDestinationIfUnset` returns `Task<bool>` but its only external caller discards the result, and Thoughts doc line 179 ("rides along in the observer-held row") holds only if the row is actually written.
- **minor** · PLAUSIBLE · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml:195`.
  `itemContext: '@Model.HttpContext.Request.Query["context"].ToString()'` embeds the base64 context token inside a `<script>` block through a Razor `@` expression, which HTML-encodes `+` as `&#x2B;`; the token is standard base64 (`StringExtensions.cs:172`, `Convert.ToBase64String`), so `+` occurs, no `HtmlEncoder` override exists in WebIt, nothing on the client decodes entities, and the SPA sends the string back as the `context` query parameter (`ThoughtsApi.ts:102`).
  The identical pattern pre-exists at `ItemShop.cshtml:754` and in three other frames, and `ThoughtsPageTest.cs:33-42` calls the handlers with the raw token without rendering the script, so no test can tell; either production works for a reason outside this repo or every frame has been broken for tokens containing `+`.
  Merge instead: one manual check with a token containing `+`; if it fails, emit the token with `Html.Raw(JsonSerializer.Serialize(...))` as the neighbouring lines (`:190-191`) already do.
- **minor** · CONFIRMED conventions · `WeblinClientApi.ts:204-247` · about 35 single-line `case 'X': return …;` lines in the new `NoClientModeWeblinClientApi` with neither braces nor `break`. `› TypeScript Code Style › Patterns` ("Switch cases are always braced and terminated `} break;` — including after `return`").
- **minor** · CONFIRMED conventions · type-only symbols imported without `import type` while mixed with values: `ThoughtsController.ts:6-9` (`ThoughtsApiSettings`, `NotesPage`, `NotesConfigEntry`, `SubtreePageEntry`, `WireNote`, `PostsViewHandlers`), `ThoughtsCommentsView.ts:5` and `ThoughtsPostsView.ts:4-5` (`NoteViewHandlers`, `CommentsViewHandlers`), `WeblinClientApi.ts:3` (`SiteCommunicatorSettings`); the repo already uses the inline `import { type Logger, … }` form, so either split or use the inline modifier.
- **nit** · CONFIRMED conventions · `DomUtils.ts:36` · `} catch (_error) { /* Just fall back to Element.append. */ }` is a plain `/* */` comment inside code and a single-line braced body. `› General Code Style`.
- **nit** · CONFIRMED conventions · `NoteBoardGrainTest.cs:96,114,1795,1811` `PokeCounts` and `NoteBoardGrainTest_FullBoardLifecycle.cs:506` `victim` · the rule names both words. `› General Code Style` ("no metaphors, colloquialisms, or imported jargon in identifiers (poke, mint, victim, …)").
- **nit** · CONFIRMED literally, informational · 32 multi-level relative imports (`../../../is` in every Thoughts file, `../../as` in `WeblinClientApi.ts`) against `› Types & modules` ("single-level relative paths"); the pre-existing WebIt tree already does this at BASE (`itemFrame.ts:1`, nine `ItemShop/*.ts` files) and `Items/n3q.WebIt/tsconfig.json` has no `paths` or `baseUrl`, so a file three levels deep cannot comply; the rule's `../lib/X` form comes from the extension's flat layout, for the rule owner.
- Refuted: `DomUtils.ts:31` `appendElemAndTryToPreventIframeReloads(parent, elem)` as a "helper handed a parent to write into"; it moves an existing element with `moveBefore` to avoid iframe reloads, creates no DOM, and carries a justifying doc block.
- Refuted: `Changes.cs` release-number rule; the deployed top release is "49", the entries sit in "50".
- Carried over as merged with other lenses: tight `new()` (126 lines), `NoteGrain.cs:88` untyped exception, `any` and boolean names in `WeblinClientApi.ts`, `Changes.cs:10` wording.

### Scope — Over-scope, Delivered: everything Thoughts-related traces to the request, one self-contained second feature rides along

**Request source.**
The PR description says only "Implements the Thoughts feature."; the inventory was built from the two design docs the PR adds and verified against code and tests, with items that only the docs ask for marked "requested by doc" so the human can decide whether the PR body covers them.
The discussion adds no scope statement.

**Request inventory** (server side).
Users post persistent public thoughts in the current room; anyone reads, comments on posts and comments, reacts; authors edit and delete their own; deleting a note with comments leaves a `[deleted]` placeholder; reactions from a configured set with exclusivity groups, vote-carrying ones contributing at most ±1 per user and driving sort order; paginated posts and comments with polling on `GetDataVersion`; a limited HTML vocabulary with a TipTap editor, server-side sanitisation on every write and `NotesMaxTextLength`; a Thoughts item in the room inventory with a post-count badge reconciled solely by `RoomGrain` while users are present; the `Thoughts` template and the WebEx `ClientConfig.thoughts` section; the item-frame page model with token validation, `clientRoomId` override, deep links and preview comments; the board destination adopted first-writer-wins; the notes grain layer with scoring, caps, culling, cleanup, `Repair` and `GetStats`; notifications as system instant messages; moderation through the WebIt commandline (requested by doc); the `Notes*` config keys; the generic observer contract through `GrainInterfaceRegistry` (requested by doc); the `Changes.cs` entry.
Stated boundaries: none in the PR body; the Thoughts doc defers images and per-note notification opt-out, and nothing in the diff contradicts that.

**Delivery verdict: Delivered.**
Every server-side requested behaviour has a code path; no major gap.

- **minor** (partial) · `Items/n3q.Grains/NoteThreadGrain.cs:364` and `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:232-245`.
  The Notes doc's Authorization table says `CreateComment` is allowed only when the immediate parent is in a non-deleted status and that the page model "enforces that policy verbatim"; the thread checks only `State.Comments.ContainsKey(parentNoteId)`, the page model checks nothing, and only the client hides Reply (`ThoughtsNoteView.ts:386-392`), so a direct POST replies to a `[deleted]` placeholder.
  No test covers it; what must ship: a parent-status check in the thread grain or the page model returning 4xx, plus a test.
- **minor** (UI deviates) · `ThoughtsNoteView.ts:386-389` treats only `Original` and `EditedByAuthor` as live, so a moderator-edited note shows no Reply or Delete although the doc and the grain (`NoteGrainTest.AuthorDeleteFromEditedByModeratorClearsTextAndAuthor`) permit author delete and replies.
- **minor** (untested) · the `>999 → k` badge (`Templates/Thoughts.cs:15-16`), room unsubscribe on last leave or deactivation (`ThoughtsItemManager.cs:35-53`; `RoomGrainTest_Notes` never asserts the observer set), `UpdateUserProfile` on open (`Thoughts.cshtml.cs:69`), the WebEx `thoughts` section (`ConfigController.cs:16-34`), and the HTTP status of author-edit-after-moderator-edit; merged with the engineering coverage findings.
- **minor** (convention) · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs` lacks the doc-link file header the Notes doc's file plan requires; the other nine listed files have it.
- Not gaps: the extension-side items (participant-menu entry, instant-message dispatch by `boardId` prefix) belong to the extension repo; `IRoom.InternalRoomIdFromClientRoomId`, `IUser.SendSystemInstantMessage`, `RoomEnterRequest.Destination` and the room routing already exist at BASE although the docs describe them as new.
- No *explained* items.

**Coverage verdict: Over-scope.**

1. **No-client mode** — commit 5a0b6fb9 (10 files, +1248/−46): `WeblinClientApi.ts` (new, 381 lines), `itemFrame.ts:33-61`, `ItemFrameModel.cs:126-146` `OnPostGetOwnItems` (sole caller `WeblinClientApi.ts:105`), `_Layout.cshtml:5`, `is.ts`, `EventListeners.ts`, `Logger.ts`, `SiteCommunicator.ts`, `itemframe.css` (+803), its own `Changes.cs` entry.
   Why not traceable: the Thoughts SPA uses only `logger`, `setClientStyleAppliedHandler`, `handleException`, `setTitle` and `setVisible` (`ThoughtsController.ts:92,143,181,215,217`), all declared at the second commit and implemented by the unchanged `wwwroot/js/WeblinClientApi.js`; no Thoughts file references `noClientMode`, `GetOwnItems`, `mapObject`, `isEmpty` or any of the 156 new CSS selectors; the destination link is `rel="noreferrer"`, so the referrer meta cannot affect it.
   Disposition: split into its own PR (it is already a separable commit with its own changelog line); if kept, call it out in the PR description as a second feature and resolve the `OnPostGetOwnItems` ownership note there.
2. **`notifications` client-config section** — `ClientConfig.cs:166-169,191`, `ConfigController.cs:31`: in neither doc and read by no server code; the engineering lens found its consumer in the extension repo.
   Disposition: keep, and name the cross-repo companion in the PR description.
3. **Encoding and whitespace churn** — UTF-8 BOM removed on line 1 of nine files, trailing-space trims in `WebItConfigDefinition.cs:43-52`, blank-line rewrites in `ClientConfig.cs`, `RoomItems.cs:95`.
   Disposition: acceptable to keep; merged with the engineering drive-by-churn nit.

Requested by doc but separable from the user-facing feature, for the human to weigh: `NotesCommandlineCommands.cs` plus `ItemCommandline.cs` (about 220 lines) and the four `ModeratorEdit*`/`ModeratorDelete*` grain methods, `GetStats` and `Repair`, which only the commandline calls; and `GrainInterfaceRegistry` (sole production caller `NoteBoardGrain.cs:579`).
Verified as supporting: `DateTimeExtensions.ToUnixTimeSeconds`, `StringExtensions.ParseOptionalLong`, the storage provider's `GetRecordsByRowKeyRange` chain, `IUser.UserProfile.HasSameIdAndNameAndAvatar`, the `AppPageModel` import map (the browser needs it to resolve the TipTap ESM bundles), `DomUtils`/`domSchedulingUtils`, the `site.ts` clip-path fix, about 19 lines of `itemframe.css`, and the `.xcf`/`.jpg` design sources (repo convention: BASE already holds eight `.xcf` and many `.psd` sources under `Docs/`).

### CTO — needs-work: the grain layer is a model of path-B guarantees, but as merged it adds stalls, a deploy-time fan-out, log and presence leaks, and an unverifiable dev harness

**Two-paths judgement.**
Grain layer: A + B (762-line doc; 218 grain tests on a real silo with the real storage provider; cold reactivation, culling with cascade, observer eviction, the activation-subscribe deadlock, delayed-notification cancellation, config drift, `Repair` at all three levels, LRU eviction, never-used-state cleanup and old-row deserialization all covered; not covered: concurrent writers to one board, the 64 KiB reactor-row overflow, the user ↔ board cycle, the first-request rebuild cost).
RoomGrain integration: A + B with gaps (subscribe and reconcile on first enter, duplicate-item repair, removal at zero, reactivation, destination first-usable-wins covered; unsubscribe on last leave or on deactivation, and a callback into a room with no users, not covered).
WebIt page model and HTML policy: A + B (24 HTTP tests with real grains proving forbidden writes did not land; the seven `Notes_*` commands are path A only with a duplicated gate; `OnPostGetOwnItems` is neither; the process-lifetime `_notesConfigTask` is untested).
SPA: A only, no TS runner exists; the posts-prefix sweep and the exclusivity switch are short enough to verify by reading, the comments-reconcile anchor rules are not in the doc.
No-client mode: neither.
Template and config: A only.

**Reliability in long-term operation**

- **major** · `NoteBoardNotificationSender.cs:347` · the wait ring; merged with the engineering and code-review majors above.
- **major** · `Items/n3q.Grains/NoteBoardGrain.cs:166-168,657-661` with `Items/n3q.Grains/NoteThreadGrain.cs:14-21,168-170,1309-1316,1376-1392`.
  After any scoring-coefficient change every board sets `_isScoringRebuildPending` on activation, and the first `GetPosts` or mutation awaits `GetSummary()` on every thread (`Task.WhenAll`, up to 300), each of which awaits `GetState()` on every note (up to 300) and writes; up to about 90,000 grain activations and reads inside one user request, bounded only by the 30 s timeout, with every reader of that room queued behind it.
  `NoteThreadEntry` lacks the meaning counts `GetAdjustedVoteSum` needs, so the note reads are unavoidable as designed.
  Why it matters: "activation is fan-out-free" (Notes doc line 114) is true only because the fan-out moves to the first request after every scoring deploy, for every board in the install base; a tuning change becomes a cluster-wide latency event.
  Merge instead: persist the upvote and downvote meaning counts in `NoteThreadEntry` so the recompute is arithmetic, or run the refresh with a bounded degree from a timer while serving the stale order and flag it in `GetStats`.
  anti-patterns › Unbounded blocking.
- **major** · `Items/n3q.Grains/NoteThreadGrain.cs:525-581`, from `NoteBoardGrain.cs:683-686,734`.
  Post culling on a full board does up to 300 sequential `GetNote` reads (the cold thread's cache is empty, so each is a `NoteGrain` activation and a row read), up to 300 sequential `Delete()`s, a reactor range query and N reactor deletes inside the user's `CreatePost` turn; the reads serve the `cullingDelete` notifications, the thread's state is already cleared first (`:543`) and `Repair` reaps orphans, so nothing requires this on the request path.
  Why it matters: once a room's board is full (300 posts, the steady state of any popular page), every "post" click pays 6–30 s with board and thread both holding their turns, stalling every reader of the room.
  Merge instead: `Task.WhenAll` with bounded degree for reads and deletes at minimum; better, return after the state clear and finish the note and reactor deletes from a one-shot grain timer.
  anti-patterns › Unbounded blocking; desired-patterns › Fail closed, degrade gracefully.
- **minor** · `RoomGrain.cs:70-73` and `ThoughtsItemManager.cs:37-39,68-110` · stale subscriptions after shutdown; merged with the code-review finding above, adding: exceptions from `inventory.Modify` inside the `[OneWay]` callback are uncaught, so the board's "subscription preserved, retried" comment (`NoteBoardGrain.cs:593-599`) is vacuous for that failure; wrap the callback body in catch-and-log.
- **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:982-987` with `ThoughtsWire.cs:38-50`, `Thoughts.cshtml.cs:166`, `ThoughtsModel.ts:306,426`.
  A post whose thread is gone is rendered to every user as a ghost note: `GetPosts` substitutes `NoteState.Empty`, caches and ships it, and every downstream filter checks only `Note == null`, so the room shows an empty post whose reply and react buttons target note 0 until an admin runs `Repair`; reachable after a crash between `RemoveThread`'s state clear (`NoteThreadGrain.cs:543`) and the board's write (`NoteBoardGrain.cs:696,741-748`).
  Merge instead: drop such entries from the page, log, schedule a self-repair of `Posts`, and never ship `Id == 0`.
  desired-patterns › Fail closed, degrade gracefully; anti-patterns › Swallowed errors / silent failure.
- **minor** · `ThoughtsItemManager.cs:42-46,112-122` · a failing board (storage outage, activation failure) is re-attempted and error-logged on every ping of every user in the room, each paying the call's latency; the degrade is correct, the retry policy is missing. Merge instead: a `_nextSubscribeAttemptAt` latch with exponential backoff, read and set in `Maintain`.
- **minor** · `NoteBoardGrain.cs:884-886`, `NoteBoardNotificationSender.cs:212-214,278-280` · bare catches; merged with the engineering minor above.
- **minor** · `Thoughts.cshtml.cs:27-33,108-117` · the faulted-task cache; merged with the engineering major above, with the CTO note that a mutable static in a Razor page model is invisible to the next page author.

**Privacy & data stewardship**

- **major** · `Items/n3q.WebIt/ItemFrameModel.cs:59-73` with `Thoughts.cshtml.cs:224,236,250,268,316` and `Items/n3q.WebIt/Program.cs:46-58`.
  `OnPageHandlerExecutionAsync` logs all handler arguments at Info, truncating only `context`, next to `LogData.Key.User = Context.UserId`; `ThoughtsModel : ItemFrameModel` binds `string text`, `string[]? reactionIds` and `clientRoomId`, and the filter runs before sanitization; the sink is Console plus the optional `AzureTableStorageLoggerProvider`, which is persistent and off-box.
  Why it matters: the mechanism is pre-existing, but this PR is the first to route user-authored prose through it (BASE frames logged base64 blobs and JSON arguments); the feature's own UI keeps reactions anonymous, the log does not; it fails "would I be comfortable if this log left the building".
  Merge instead: allow-list the logged argument keys in the `switch` at `:65-68` (drop or length-only `text`, drop `reactionIds`), or override the virtual filter in `ThoughtsModel`.
  anti-patterns › PII / secrets in logs & over-collection.
- **major** · `Items/n3q.WebIt/Pages/ItemFrame/ThoughtsWire.cs:77,83` via `NoteBoardGrain.cs:331-339,1092-1105` and `IUser.cs:12-24`.
  `NotesPageWire.Authors` forwards the grain `UserProfile` unchanged, which carries `OnlineStatus` (refreshed from live presence by `UserAccount.GetUserProfile`, `UserAccount.cs:51-58`) and `Lang`; `JsonOptions` serializes both, the SPA type does not declare `onlineStatus` but it is on the wire; the board caches the first-resolved profile per activation, so it is a stale presence snapshot of every author; any user reaches any board through the `clientRoomId` override (`Thoughts.cshtml.cs:332-340`).
  Why it matters: at BASE presence was room-scoped and id-only (`RoomUser.ToPresence()` → `RoomPresence { UserId }`, delivered only to the room's clients) and no WebIt wire type exposed `OnlineStatus`; this widens who can see an existing datum to anyone who knows a URL, and contradicts the code's own intent (`UserProfile.HasSameIdAndNameAndAvatar`, Notes doc "name + avatar URL").
  Merge instead: a `UserProfileWire { UserId, Name, ImageUrl }` (optionally `Lang`) in `ThoughtsWire.cs`, and have the board cache only those fields.
  anti-patterns › PII / secrets in logs & over-collection (widening who can see data).
- **minor** · `ItemFrameModel.cs:101-107,126-146` · `OnPostGetOwnItems`; merged with the engineering finding above (the CTO lens rates it major as an inherited base-class endpoint; the maintainer's stance keeps the merged severity at minor).
- **minor** · `NoteBoardNotificationSender.cs:189-215,336-340` with `NoteBoardGrain.cs:937-941` · `OnReactionAdded` fires for any non-empty `ReactionsAdded` with no meaning filter and the payload carries `actorUserId`, `actorName` and `actorImageUrl`; `ThoughtsReactionsView.ts` renders counts only, so users will reasonably believe a downvote is anonymous. Merge instead: omit actor identity on `reaction` (at least when the reaction's `Meanings` include `downvote`), or show reactors in the UI so expectations match.
- **minor** · `NoteBoardNotificationSender.cs:164-170,242,342-347` with `UserInstantMessages.cs:62-65` and `InstantMessageGrain.cs:85,96` · `parentAuthorDelete`, `parentModeratorDelete` and `cullingDelete` carry `before.Text` as `originalText`, which is stored in a new persisted `IInstantMessage` grain per child author until that recipient is online and acknowledges; an offline or churned recipient keeps it indefinitely, while `SoftDelete` correctly wipes the note itself (`NoteGrain.cs:138-146`). Merge instead: drop `originalText` on the three delete events or send a short excerpt; a doc-stated field, so a design choice to confirm.
- **minor** · `NoteThreadGrain.cs:1514-1540` with `NoteThreadUserReactionsGrain.cs:14-36` · `{boardId}:{postId}:{userId}` rows record who reacted how on which page; after a soft delete the entries are dropped only when that same user next reads the thread, so a user who never returns keeps the record forever, and `IUser`, `UserGrain` and both commandlines have no erase or purge path, so a GDPR-style erasure would need a storage scan. Merge instead: record it in the Notes doc as a known gap now; a `RemoveNotes` fan-out on note removal or a `Notes_EraseUser` command later.
- **minor** · `ThoughtsDraftStore.ts:15-18,29,48-54,84-89` · the single `localStorage` slot `thoughts.lastEditorDraft` holds room, target ids and `text`; `isRecordFor` matches on room and target only, so a second weblin account on the same browser profile sees the previous user's unsent text. Merge instead: include `userId` in the record and the match, add a timestamp and drop stale drafts.

**Maintainability & extensibility at scale**

- **major** · documentation gate; merged with the engineering finding above.
- **major** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/itemFrame.ts:42` with `WeblinClientApi.ts:21-34,165-381,207,213`, `ItemFrameModel.cs:126-146`, `Changes.cs:10`.
  `ItemFrameUtils` switches to `NoClientModeWeblinClientApi` when `?noClientMode` is present or `window === window.top`, so any item-frame URL opened directly runs a 217-line emulation whose answers differ from the real client (`Room.GetItems` → empty, `Client.CreateItem` → reject) and which polls `OnPostGetOwnItems`, a handler with no other caller; the file carries "Todo … for non-dev client-less mode use"; no test sets `noClientMode`, neither design doc mentions it, and the changelog presents it as a user feature.
  Why it matters: every future item frame built on `ItemFrameUtils` silently inherits a second, half-implemented client API that must mirror the real one forever, plus the hand-written `declare global class WeblinClientApi` as a second mirror of `wwwroot/js/WeblinClientApi.js`; it satisfies neither path.
  Merge instead: split into its own PR with a short item-frame doc section (activation rule, emulated versus rejected messages) and a `WebIt.Test` for `OnPostGetOwnItems` with the ownership guard; or at minimum gate on the explicit query parameter only and reword or drop the changelog line.
  engineering §1 Proportionality, §3 Blast radius; anti-patterns › Premature abstraction; test.md › Test the real production code.
- **minor** · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:23-27,67-103` with `NoteBoardGrain.cs:578-590` and Notes doc line 438.
  The doc says the registry "rebuilds once on a miss before treating the entry as permanently dead"; `GetGrainInterface` only calls `InitIfNeeded` (guarded by `_initComplete`, never re-run) and a lookup, so a miss evicts and persists the eviction immediately; the risk is low because every referenced assembly is force-loaded at first use, but a new observer author will trust the doc.
  A simpler design exists with a stated trade-off: persist the Orleans 8.2 `GrainId` and resolve with `GrainFactory.GetGrain(grainId).AsReference<…>()`, removing the registry, its test and the "name your own primary interface's FullName" convention that nothing enforces, at the cost of the class-implements pre-check, currently the only eviction mechanism for an observer class that dropped the interface.
  Merge instead: either fix the doc (and the `INoteBoard.cs:9-18` hover block) to the as-built behaviour, or adopt `GrainId` with N-consecutive-failure eviction; do not leave both the doc and the code claiming different things.
  Downgraded from major: mismatch real, operational risk low.
- **minor** · `INoteBoard.cs:10,15` with `NoteBoardGrain.cs:598` · stale hover documentation on the one interface every future consumer implements; merged with the engineering nit above and raised to minor by the CTO lens.
- **minor** · `Templates/Thoughts.cs:81-90` ↔ `ConfigController.cs:16-25` · duplicated geometry and URL suffix; merged with the engineering finding above.
- **minor** · `NoteBoardGrain.cs:358-399` ↔ `NoteThreadGrain.cs:1571-1610` · the LRU twice; merged with the engineering finding above.
- **minor** · `NotesCommandlineCommands.cs:109-204,137-144` ↔ `Thoughts.cshtml.cs:367-369` · untested moderator commands with a duplicated gate; merged with the engineering finding above.
- **minor** · `AppPageModel.cs:60-152,184-193` with `lib-shims/*.d.ts` · the import map in the base model; merged with the engineering finding above.
- **minor** · both design docs, "File-Level Implementation Plan" sections.
  The file tables describe a plan, not this diff: Notes doc lines 736-738 list `IUser.cs`, `UserGrain.cs` and `UserInstantMessages.cs` as modified for `SendSystemInstantMessage`, but the diff touches neither grain file and BASE `IUser.cs:32,101` already has both members; Thoughts doc line 289 lists `IRoom.InternalRoomIdFromClientRoomId` as added (BASE `IRoom.cs:13`); Thoughts doc line 116 says `RoomGrain` fetches the language via `IUser.GetUserLang` (no such call anywhere in `Items/`); Notes doc line 758 names a `ProjectReference` that does not exist; `GrainInterfaceRegistry.cs` lacks the doc-link header the Notes doc mandates.
  Merge instead: convert both sections to an as-built inventory of this branch and mark pre-existing and client-repo rows.
  `criteria/project.md › The documentation gate` (minor for a stale sentence).
- **minor** · `ThoughtsModel.ts:412-420,447-451,516-519` with `ThoughtsController.ts:290-297,380-387`.
  The comments-reconcile anchor and preview-clobber rules are the most intricate client logic in the feature and are pinned by nothing; no TS runner exists, and the Thoughts doc's SPA section describes only the posts-prefix sweep; the code carries substantive comments at exactly these points, so a careful reader can verify it, which is why this is minor rather than "neither".
  Merge instead: lift the anchor and preview rules into the Thoughts doc's SPA section now; a minimal TS harness for the DOM-free `ThoughtsModel` is the lasting fix.
- **minor** · `RoomGrainTest_Notes.cs:51-248` · no test sends a leave or heartbeat timeout and asserts the board unsubscribed, none pins "unsubscribe on deactivation unless shutdown" (`ThoughtsItemManager.cs:35-39`), none drives a callback into a room with no users, which are exactly the cases where the code and the doc disagree; the board side is pinned (`SubscribeUnsubscribeLeavesNoState`), so a room that forgets to unsubscribe turns nothing red.
  Merge instead: `LastLeaveUnsubscribesFromBoard` and `DeactivateUnsubscribes`, reading `NoteCountObservers` through the table grain as `NoteBoardGrainTest.cs:676-683` does.
- **minor** / **nit** · the useless-test shapes at `NoteBoardGrainTest.cs:1250`, `:1090` and `NoteGrainTest.cs:159` were found independently by the CTO lens; merged with the engineering list above.

**Nits (CTO)**

- `SiloConfigDefinition.cs:104` · `NotesMaxPostsPerRoom` is brand-new yet its own doc (Notes doc line 625) apologises for the name; rename to `NotesMaxPostsPerBoard` while nothing persists it.
- `NoteBoardGrain.cs:124-125` · the comment "bounded by the posts on the board … like `_noteStateCache`" is wrong; `_noteStateCache` holds comments too and is bounded by posts × comments (90,000 entries at defaults) per activation.
- `NoteBoardGrain.cs:689-696,707-718`, `NoteThreadGrain.cs:377-380` · state is mutated before `WriteStateAsync` with no failure handling; matches the repo's posture (`InventoryGrain.CommitTransaction`), so a convention note, but `CreateComment` is therefore not idempotent on a failed board write.
- `SeparateJsonAzureTableStorageTableGrain.cs:8` with `AzureCloudTableExtensions.cs:9-24` · range queries fetch full bodies with no `select` and funnel through one non-reentrant activation per table; pre-existing pattern, hit only at thread end and `Repair`.
- `ThoughtsItemManager.cs:23-52` with `RoomGrain.cs:32-40` · `Init(internalRoomId)` must run before `Maintain`/`Stop`; an exact mirror of the `RoomItems` precedent, so a consistency choice.
- `NoteBoardNotificationSender.cs:277` · `DispatchToDirectChildren` reads with `userId = "system"`, activating a phantom reactor grain per thread per dispatch and taking an LRU slot; add a reactions-free read path.
- `NoteBoardNotificationSender.cs:74-79` duplicates `NoteBoardGrain.RegisterOneShotTimer` (`:280-288`).
- `NoteBoardGrain.cs:39-57` · twelve `required` members mean every future field must be non-`required` or old rows throw; the doc says so at `Destination`, a comment on the record would make the rule enforceable.
- `NoteBoardGrain.cs:53` with `IRoom.cs:54-59` · the page URL is persisted per board with query and fragment kept; strip them.
- `RoomGrain.cs:51,65` · `_logData["Destination"]` puts the page URL on every room log line (pre-existing); log the internal room id only.
- `ItemFrameModel.cs:92` · a context parse failure logs the full token (pre-existing); truncate as line 66 does.
- `Templates/Thoughts.cs:16-24` · the `>999 → k` badge branch has no test.

**What is good.**
The strict call direction board → thread → note/reactor is held in code and is the single most valuable structural property for future consumers.
The grain tests are the model case for path B in this codebase: real silo, real storage provider, persisted rows read back through the same provider, an independent in-test scoring oracle, cold deactivation mid-test, and a genuine regression test for the activation-subscribe deadlock with a 5 s bound.
Never-used-state cleanup, bounded observer sets, fan-out-free activation, `[PreferLocalPlacement]`, per-(thread, user) reactor rows and three-level `Repair` are the right shape for billions of potential rooms.
The design docs are substantial and mostly true to the code.

## Coverage

- Lenses run: Engineering, Code-review skill (effort `high`), Scope, CTO; each with its own dimension or accountability fan-out and refute-biased verification.
- Lenses skipped (planned, not active): Architecture, Security, Adversarial security, Feature regression.
- PR description and discussion read via `gh pr view 92 --comments`; the description is one sentence, so the scope inventory was built from the PR's own design docs and verified against code and tests, as stated in the Scope section.
- Diff sanity check: 110 files, matching GitHub's Files changed count; base is the merge-base `2409cf12`, head `5a0b6fb9`.
- Sampling: the ~30 minified vendor bundles under `wwwroot/lib` were skipped by every reviewer; `NoteBoardGrainTest.cs` (3203 lines) was read in the ranges 1-400, 700-790, 950-1000, 1085-1135, 1250-1280, 1400-1800 and 2800-3203 plus whole-file greps, and `NoteThreadGrainTest.cs` in the ranges 1-400, 550-1000 and 1175-1574 plus greps; all other touched test files were read fully; the C# conventions finder grep-swept the added `.cs` files (about 14,000 lines) and read the hit contexts rather than every line; the TypeScript conventions finder read `WeblinClientApi.ts`, `thoughtsConfirmDialog.ts`, `domSchedulingUtils.ts` and the shared-script diffs fully and pattern-scanned the other thirteen Thoughts files.
- The code-review line scan was therefore sampled, not full; the correctness finders read all C# and TypeScript hunks of the diff and the full grain, page-model, wire, policy and SPA API files.
- One finding (`Thoughts.cshtml:195`) is PLAUSIBLE, not confirmed, and needs a manual check; one CTO verifier drop (the DataVersion mix) was overridden by the orchestrator on a direct code read, as noted in the Code-review section.
- Test methods counted with `grep -c "\[TestMethod\]"`: 259 across ten files.
