# Review — server · PR 92 · all · fable

- Change: [Feature/aw thoughts](https://github.com/lupuslabs/nine3q/pull/92)
- Intent: Implements the Thoughts feature: users leave persistent public thoughts and comments in the room they are in, with reactions that drive the sort order (PR description; detail taken from the commit messages and the two design documents the PR adds under `Docs/Features/`)
- Branch: `feature/aw_thoughts`
- Runtime: 2 h 54 min
- Verdict: needs-work

## Summary

- Engineering: needs-work — one transient grain-call failure disables every Thoughts page until WebIt restarts (E1); a reply to the comment the cap removes becomes a hidden orphan (E2); no document in `docs/` covers any edited area (E3); the new item-frame handler and the config keys served to the extension have no test (E4, E5).
- Code-review: needs-work — "Show more replies" reloads the same first page, so comments past the first twenty can never be shown (CR1).
- Scope: over-scope, delivered — the no-client mode for item frames (commit 5a0b6fb9, about 1250 lines) is a second feature with no code dependency from Thoughts (SCP1); a page-wide referrer policy ships without a stated reason (SCP2).
- CTO: needs-work — a board → user → room → board wait cycle stalls room requests for the Orleans timeout (CTO1); every post and edit writes the full note text with the user id into the persistent log (CTO2); the no-client mode has neither a test nor a document and switches on in every unframed item frame (CTO3).

| Stat | Count |
|-|-|
| Files changed | 110 |
| Code files changed | 70 |
| Code files changed (excluding tests) | 59 |
| Code lines delta (excluding tests and comments) | +7642 |
| Scaffolding files changed (non .ts/.cs) | 40 |
| Code files new (excluding tests) | 34 |
| Code files deleted (excluding tests) | 0 |
| Unit test files | 3 |
| Unit test functions/methods | 32 |
| Grain/integration test files | 8 |
| Grain/integration test functions/methods | 227 |

## Short

**What it is.**
Three commits.
The first adds a notes layer to the silo: a board grain per room, a thread grain per post, a note grain per text, a reactions grain per user and thread, a notification sender, a reflection-based observer registry, a row-key-range query on the storage provider, and 227 grain tests on the isolated silo.
The second adds the Thoughts item: the room subscribes to its board and shows a badge item while posts exist, WebIt serves an item-frame page with Razor handlers for reading, writing, editing, deleting and reacting, and a TypeScript single-page app on vendored tiptap/prosemirror renders and edits the notes.
The third adds a no-client mode: when an item-frame page is opened outside the weblin client, a second client-API implementation emulates or fails the client calls.

**What is sound.**
The grain layer is well designed and well tested.
State is bounded by configured caps, the grain call graph is a tree except for the cycle in CTO1, the HTML policy runs on every write path, authorship is checked in WebIt, and the tests run the real grains and the real WebIt pipeline including deactivation and cold reads.
The lifecycle test compares the whole read API against an in-test model after every mutation.
The design documents describe the system in depth and most spot-checked claims match the code.

**What turns the call.**

- **E1 · Major — one failed config call breaks every Thoughts page until WebIt restarts.** `Thoughts.cshtml.cs:108` caches the first `GetNotesConfig` task in a static field, faulted or not. A timeout or a silo restart at that moment stores the failure for the process lifetime.
- **E2 · Major — a reply to the comment the cap removes becomes an orphan.** `NoteThreadGrain.cs:373` culls after the parent check; when the parent is the lowest-scored leaf, the reply is created under a parent that no longer exists, is never listed, and is deleted by the next repair.
- **CR1 · Major — comments past the first page can never be shown.** `ThoughtsController.ts:398` always requests page 0; the "Show more replies" button fetches the same twenty comments again. The design document promises load-more at truncation points.
- **CTO1 · Major — a three-grain wait cycle.** `NoteBoardNotificationSender.cs:347` awaits the recipient's user grain, which may be awaiting the room, which awaits the board; none is reentrant, so the three wait for the 30 s Orleans timeout and the user's room request fails.
- **CTO2 · Major — note text in the persistent log.** `ItemFrameModel.cs:63` logs all handler arguments; the new create and edit handlers put the full note HTML, up to 10000 characters, with the user id into log storage on every write. The grains clear the text on delete; the log copy stays.
- **E3 · Major — no document in `docs/` covers any edited area.** The two design documents live in the nine3q repo under `Docs/Features/`, which the weblin root rule does not name, and nothing describes the no-client mode anywhere (CTO3).
- **E4, E5 · Major — two delivered pieces have no test:** the new `OnPostGetOwnItems` handler, and the `thoughts` and `notifications` config sections WebEx serves to the extension.

**Scope.**
Every behaviour the design documents specify has a code path and a test.
The no-client mode is a separate feature in its own commit with its own `Changes.cs` line (SCP1); it also switches on for any item frame opened as a top-level page in production, with a public API call that can hang (E7).

**Coverage caveats.**
The TypeScript front end (about 5000 lines) has no test tier in WebIt, so the behaviour-coverage dimension does not apply to it and path A (readable, documented code) has to carry it.
The code-review line scan covered the full diff minus the vendored libraries.
Four test methods are flagged as weak (E28); that is below the threshold for an aggregate problem.

## Findings

- `E1` · `major` · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:108` · a faulted `GetNotesConfig` task is cached for the WebIt process lifetime
- `E2` · `major` · `Items/n3q.Grains/NoteThreadGrain.cs:373` · `CreateComment` culls the parent leaf after the parent check, creating an orphan comment
- `E3` · `major` · `Items/n3q.Grains/NoteBoardGrain.cs:96` · no document in `C:/Heiner/weblin/docs/` covers the notes grains, the Thoughts item, the item-frame page, the storage provider, the registry, the room change, the WebEx config sections or the no-client mode
- `E4` · `major` · `Items/n3q.WebIt/ItemFrameModel.cs:126` · the new `OnPostGetOwnItems` handler has no test
- `E5` · `major` · `Base/n3q.WebEx/Controllers/ConfigController.cs:31` · the `thoughts` and `notifications` config sections served to the extension have no test on either side
- `E6` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:629` · `SetDestinationIfUnset` changes state in memory only; the destination the room hands over is lost on deactivation
- `E7` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:321` · no-client `Item.GetProperties` can never settle, ignores `timeoutMs`, and throws for the Producer frame
- `E8` · `minor` · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:176` · board and thread data versions are merged by `Math.Max`; polling can stop detecting board changes
- `E9` · `minor` · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:88` · the assembly walk is one level deep, and the design document promises a rebuild on a miss that the code does not do
- `E10` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:884` · three bare `catch` blocks drop notifications with no log line
- `E11` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:594` · a `try`/`catch` around a `[OneWay]` observer call cannot see the observer's exception
- `E12` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsNoteView.ts:258` · avatars from hosts without CORS headers never load; the default avatar replaces them
- `E13` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsController.ts:228` · `await runDataRefresh()` returns before the refresh when one is in flight; the new post is not scrolled to
- `E14` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsApi.ts:173` · JSON responses are cast to the expected type without narrowing
- `E15` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsDraftStore.ts:85` · the unsent draft is not tied to the user; the next identity on the same browser gets it
- `E16` · `minor` · `Items/n3q.WebIt/ItemFrameModel.cs:129` · `OnPostGetOwnItems` returns the context inventory without checking it is the user's; for a room frame that is the room's items
- `E17` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:294` · the sort-score formula and the `"upvote"`/`"downvote"` meaning ids are written in two places each
- `E18` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:360` · the per-user reactions LRU cache is implemented twice with different eviction rules
- `E19` · `minor` · `Items/n3q.Grains/NoteThreadGrain.cs:446` · the cascade-removal walk is written twice, the post-order traversal three times, the note-delete `try`/`catch` seven times, one with the wrong note id in the log
- `E20` · `minor` · `Base/n3q.WebEx/Controllers/ConfigController.cs:19` · the Thoughts frame URL and window options are defined in WebEx and in the item template, kept in sync by comment
- `E21` · `minor` · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:365` · sanitize-then-length-check is written twice in WebIt
- `E22` · `minor` · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:204` · code with no caller: the `NoteWithAncestors` handler and API method, three settings served to the SPA, `waitForRenderComplete`, `INoteBoard.HasPosts`, `INoteThread.GetDataVersion`
- `E23` · `minor` · `Items/n3q.GrainInterfaces/INoteBoard.cs:10` · comments and the design documents name methods, classes and behaviour that do not exist in the code
- `E24` · `minor` · `Items/n3q.Common/Changes.cs:10` · the no-client entry is not written from the user's view, and the `site.ts` clipping fix that changes the Person and ItemShop frames has no entry
- `E25` · `minor` · PR title and description · the title is the branch name; the description names one of four things that ship
- `E26` · `minor` · `Items/n3q.WebIt/Scripts/is.ts:76` · `is.stringsObject` is deprecated while its three callers stay on it
- `E27` · `minor` · `Items/n3q.WebIt/Pages/NotesCommandlineCommands.cs:8` · partial coverage: the admin commands, the Thoughts item badge properties, the import map and the room's unsubscribe have no test
- `E28` · `minor` · `Items/n3q.GrainTests/NoteBoardGrainTest.cs:1261` · `StateSurvivesDeactivation` never deactivates; the lifecycle test asserts lower bounds where exact counts are known; two `NoteGrainTest` methods check nothing
- `E29` · `minor` · `Items/n3q.Grains/NoteGrain.cs:88` · untyped `throw new Exception`
- `E30` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:129` · null-forgiving `!` on own-code null flow in four places
- `E31` · `minor` · `Items/n3q.Grains/ThoughtsItemManager.cs:11` · `static readonly` fields named `_camelCase` instead of PascalCase
- `E32` · `minor` · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:375` · four helpers against the naming rules: `CheckAuthor`, `EnrichAuthors`, `ParseOptionalLong`, `EnqueueImmediateIm`
- `E33` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:68` · new `any` on three lines
- `E34` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:318` · a block-form `eslint-disable` turns the rule off for the rest of the file
- `E35` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsEditor.ts:247` · toolbar buttons built by a write-into-parent helper
- `E36` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsNoteView.ts:291` · hard-coded English `(edited)` marker
- `E37` · `nit` · `Items/n3q.Grains/NoteBoardGrain.cs:990` · `GetPosts` rebuilds the author map and the empty-reaction strip its sibling helpers already provide
- `E38` · `nit` · `Items/n3q.Grains/NoteBoardNotificationSender.cs:74` · one-shot timer options written inline although `RegisterOneShotTimer` exists
- `E39` · `nit` · `Items/n3q.Grains/NoteThreadGrain.cs:893` · `ParseOptionalLong` wrapped in the same `catch (FormatException)` twice; an `OverflowException` would abort the repair sweep
- `E40` · `nit` · `Items/n3q.Grains/NoteGrain.cs:116` · the soft-deleted predicate is spelled out inline six times; the thread key split is written by hand
- `E41` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsController.ts:159` · the controller has no `stop()` although it owns a timer, a document listener and a view with `stop()`
- `E42` · `nit` · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:119` · `Async` suffix on eleven named Razor handlers
- `E43` · `nit` · `Items/n3q.Grains/NoteBoardNotificationSender.cs:49` · tight `new()` on 167 added C# lines
- `E44` · `nit` · `Items/n3q.Grains/NoteThreadUserReactionsGrain.cs:63` · about 220 added lines over 120 columns, 69 outside tests
- `E45` · `nit` · `Items/n3q.GrainInterfaces/INote.cs:87` · member order: constants last, factories after properties, a field after a property, private fields before public ones
- `E46` · `nit` · `Items/n3q.Grains/ThoughtsItemManager.cs:23` · K&R brace on a constructor declaration
- `E47` · `nit` · `Docs/Features/Notes/Notes Infrastructure - Boards, Posts, Comments, Reactions.md:443` · "iff" in prose
- `E48` · `nit` · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:14` · boolean names that do not read as booleans: `_initComplete`, `handleOwnItems`, `inOwnItemsUpdate`, `ownItemsUpdateRequested`
- `E49` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:204` · 33 unbraced one-line switch cases
- `E50` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:101` · missing semicolons on five new lines
- `E51` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:2` · unused `ConsoleLogger` import, import order, and type-only imports without `import type` in seven places
- `E52` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsEditor.ts:444` · `getSubmitHtmlOrNull` changes editor state under a `Get` name
- `E53` · `nit` · `Items/n3q.WebIt/Scripts/DomUtils.ts:35` · single-line braced `catch` with a `/* */` comment
- `E54` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsApi.ts:45` · small cleanups: `String()` on strings, dead nil branches, a parameter named `noteId` that takes a post id, serializer options rebuilt per call
- `E55` · `nit` · `Items/n3q.WebIt/AppPageModel.cs:86` · 29 vendored bundles carry no version in the file name and no license text
- `E56` · `nit` · `Items/n3q.Grains/RoomItems.cs:94` · whitespace-only edits on lines the feature gives no reason to touch
- `E57` · `nit` · `Items/n3q.Grains/ThoughtsItemManager.cs:17` · the manager's log lines carry no room id
- `E58` · `nit` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsDraftStore.ts:58` · a failed draft save is logged only at verbose level
- `CR1` · `major` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsController.ts:398` · comments are always requested as page 0; "Show more replies" reloads the same page
- `SCP1` · `minor` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:165` · no-client mode for item frames is a separate feature the Thoughts request does not depend on
- `SCP2` · `minor` · `Items/n3q.WebIt/Pages/ItemFrame/Shared/_Layout.cshtml:5` · a referrer policy for every item frame page, with no stated reason
- `CTO1` · `major` · `Items/n3q.Grains/NoteBoardNotificationSender.cs:347` · board → user → room → board wait cycle resolved only by the Orleans timeout
- `CTO2` · `major` · `Items/n3q.WebIt/ItemFrameModel.cs:63` · full note text with the user id is written to the persistent log on every create and edit
- `CTO3` · `major` · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/itemFrame.ts:42` · no-client mode is guaranteed by neither path: no test, no document, automatic activation in every unframed item frame
- `CTO4` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:230` · after a failed state write the activation keeps state storage does not have; `LastNoteId` can be re-issued
- `CTO5` · `minor` · `Items/n3q.Grains/ThoughtsItemManager.cs:117` · a failed Thoughts-item update is not retried until the next post-count change
- `CTO6` · `minor` · `Items/n3q.Grains/NoteBoardGrain.cs:985` · a post whose note row is missing is served to everyone as an empty note with id 0, cached, unlogged
- `CTO7` · `minor` · `Items/n3q.Grains/NoteBoardNotificationSender.cs:277` · notification dispatch reads comments as user `"system"`, activating a reactor grain and taking a cache slot per dispatch
- `CTO8` · `minor` · `Items/n3q.WebIt/Pages/ItemFrame/ThoughtsWire.cs:77` · every reader receives each author's live online status, `isUser` flag and language

## Pre-existing Debt

Debt this review revealed; not caused by this change and not a condition of merging it.

- `E59` · `minor` · `Items/n3q.WebIt/Controllers/ItemApiController.cs:109` · `GetItemIds` and `GetItemProperties` grant Owner access to any inventory id once the user's own token authenticates; a check that `inventoryId == userId` would close it
- `E60` · `nit` · `Items/n3q.Common/Changes.cs:8` · release 50 has an empty name; the Change Log rule asks for a short PascalCase name

## Detail

### Engineering — needs-work: well built and well tested, but a cached failure, an orphaning cull, the unmet documentation gate and two untested deliveries should be fixed before merge

Read in full by the dimension reviewers: every new C# and TypeScript file, the hunks of every modified file with base context, both design documents, both `CLAUDE.md` files, `docs/README.md` and the four `docs/` documents; tests sampled as stated under Coverage.
Cleared checks worth recording: the grain-record type-placement rule is followed; no test helper ships in production code; the HTML policy runs on every write path and the client renders through its schema; the wire records match the TypeScript types; every `[TestMethod]` carries `[TestCategory]`; the `Changes.cs` entries sit in release 50, above the deployed 49; the storage-provider change is additive and leaves existing grains as they were; the shared helpers added to `n3q.Tools`, `DomUtils`, `EventListeners` and `is.ts` all have callers.
The `site.ts` change from `clipPath` to `clip-path` is Supporting work: `ThoughtsNoteView.ts:320` reaches `calcClipImageElemByBoxDistancesStyle` and applies the result with `style.setProperty`, which ignores the camelCase key, so the avatar clipping the view implements needs the rename.
Behaviour coverage: the grains, the room integration and the WebIt handlers are covered by tests that run the real code through its real entry points; the TypeScript front end has no test tier in WebIt (`package.json` carries only lint scripts), so the dimension is not applicable to it.

**E1** · **major** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:108` · *introduced* · a faulted `GetNotesConfig` task is cached for the WebIt process lifetime.
`GetCachedNotesConfig` stores the first `board.GetNotesConfig()` task in a static field with `Interlocked.CompareExchange` and returns that task to every later request; nothing resets it.
The comment at lines 27-32 argues that a misconfiguration should be visible on every request, but `NoteBoardGrain.GetNotesConfig` (`NoteBoardGrain.cs:1117`) only wraps `_app.Config.NotesReactions` in `Task.FromResult` and cannot fail on configuration.
The only way the task faults is the grain call itself: a response timeout, a silo restart or a gateway reconnect at that moment.
From then on every Thoughts frame for every user fails until someone restarts WebIt, while the silo has long recovered, and nothing in the logs names the cached task as the cause.
The cache saves one of five grain calls per page load.
Merge instead: drop the static cache and call `await board.GetNotesConfig()` in `OnGetAsync` like the other grain calls; or cache the `NotesConfig` value only after a successful await and leave the field null on failure so the next request retries.
(criteria/engineering.md §2 Simplicity; desired-patterns.md Fail closed, degrade gracefully; also raised by the CTO lens as reliability, merged here)

**E2** · **major** · `Items/n3q.Grains/NoteThreadGrain.cs:373` · *introduced* · `CreateComment` culls the parent leaf after the parent check, creating an orphan comment.
The parent check at line 364 runs before `CullOne` at line 373.
At the comment cap, when `parentNoteId` is the lowest-scored leaf, `CullOne` (lines 602-660) removes it: the new comment is not registered yet, so `HasChildren(parent)` is false and the parent is a candidate.
`Initialize` at line 376 then creates a note whose `ParentId` is gone; `RegisterEntry` adds it under a parent that is no longer in `Comments`, `GetComments` from the post never lists it, `GetCommentIdsLeavesFirst` classifies it as an orphan, and `Repair` deletes it.
The caller gets a success response with the note and the comment vanishes on the next reload.
`NoteThreadGrainTest_CommentCap` replies only to the post, never to a leaf.
Merge instead: exclude the new comment's ancestor chain from the cull candidates, or re-check after `CullOne` that `parentNoteId` is still present and throw `KeyNotFoundException` before `Initialize`; add a test that replies to the lowest-scored leaf at the cap.
(criteria/engineering.md correctness; anti-patterns.md Hidden temporal coupling)

**E3** · **major** · `Items/n3q.Grains/NoteBoardGrain.cs:96` · *introduced* · no document in `C:/Heiner/weblin/docs/` covers any area the change edits.
The weblin root `CLAUDE.md` Hard Rule says a document in `docs/` must cover every edited area before code changes.
`docs/` holds the README and four documents (upload avatar, vidconf, backpack item rendering, GUI styling); none covers the notes grains and interfaces, the Thoughts item and `ThoughtsItemManager`, the WebIt item-frame page with its wire layer, sanitizer, vocabulary and admin commands, the storage provider's row-key-range query, `GrainInterfaceRegistry`, the `RoomGrain` observer role, the WebEx `thoughts` and `notifications` config sections, the vendored editor import map, or the no-client mode.
The vidconf document references `ClientConfig.cs:69` only; the GUI-styling claim that theme CSS is the only path into iframes still holds because the no-client mode answers `Client.GetBaseCss` with an error.
The PR's two design documents live in the nine3q repo under `Docs/Features/`, next to the pre-existing `CryptoClaim.md` and `iframeApi.md`; the root rule and `docs/README.md` name only the weblin `docs/`, and the root `CLAUDE.md` index does not list them, so a developer following the rule concludes the area is undocumented.
Nothing describes the no-client mode anywhere beyond the commit message and a `Changes.cs` line (CTO3).
Merge instead: add documents under `C:/Heiner/weblin/docs/` (notes infrastructure; Thoughts item and frame; no-client item-frame mode), or have the owner amend the root rule to recognise nine3q `Docs/Features/` and list the two documents in the root `CLAUDE.md` index.
Observation for the human, not a finding against the PR: the root rule and the in-repo `Docs/Features/` precedent contradict each other, and whether `docs/README.md`'s conventions (one sentence per line, no meta-narration) reach the nine3q folder is undecided.
(weblin root CLAUDE.md Hard Rule; criteria/project.md The documentation gate; the CTO maintainability findings on the missing index entry and the undocumented mode are merged here)

**E4** · **major** · `Items/n3q.WebIt/ItemFrameModel.cs:126` · *introduced* · the new `OnPostGetOwnItems` handler has no test.
The handler returns every item of the context's inventory with client-prepared properties and is called by `WeblinClientApi.ts:105`.
No test under `Items/n3q.WebIt.Test` runs it: `ItemFrameModelTest.cs` only GETs `/ItemFrame/Test` for the context filter, and `ThoughtsPageTest.cs` POSTs only Thoughts handlers.
Reverting the handler fails no test, and the tier can express one: `WebItWebServiceTestBase` runs the real WebIt app on the isolated silo with a minted context token.
Merge instead: a test that creates two items in the user's inventory, POSTs `?handler=GetOwnItems`, and asserts both ids with their properties; a second case with a context whose inventory is not the user's fixes the behaviour E16 asks about.
(lenses/engineering.md Behaviour coverage)

**E5** · **major** · `Base/n3q.WebEx/Controllers/ConfigController.cs:31` · *introduced* · the `thoughts` and `notifications` config sections served to the extension have no test on either side.
These keys are a wire contract: the extension reads `thoughts.enabled`, `thoughts.itemFrameProperties` (`ChromeExt/src/lib/Utils.ts:94`, `contentscript/ContentItemFrames.ts:66`) and `notifications.enabled` (`ContentInstantMessageManager.ts:48`).
`Base/n3q.WebEx.Test/ConfigControllerTest.cs:11` runs the real `/Config` endpoint but asserts only `clusterName`, the `xmpp` fields and `vp.vpiRoot`; it passes with this diff reverted.
No file under `ChromeExt/src/test` mentions either key; this review cannot run the extension's tests and only states that no covering test exists there.
Merge instead: extend `ConfigControllerTest_Get` to assert `thoughts.enabled`, that `thoughts.itemFrameProperties.IframeUrl` ends with `Thoughts?context={context}`, and `notifications.enabled`; note for the extension PR that a Chai test should cover reading `thoughts.itemFrameProperties`.
(lenses/engineering.md Behaviour coverage, wire contracts need a test on each side)

**E6** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:629` · *introduced* · `SetDestinationIfUnset` changes state in memory only; the destination the room hands over is lost on deactivation.
The method sets `State = State with { Destination = destination }` and returns; its sibling `SetDestination` (lines 612-627) ends with `await WriteOrDeleteIfUnused()`.
The internal caller `SubscribePostCountChangedObserver` (lines 555-560) writes afterwards; the external caller `ThoughtsItemManager.SetDestinationOnNoteBoard` (`ThoughtsItemManager.cs:62`, reached from `RoomGrain.cs:68`) ignores the result and nothing on the board writes after it.
The path runs when a room is already subscribed with a null destination and a usable one arrives later: an earlier enter without a destination, or a room activation started by a ping or chat request.
If the board is idle-collected or the silo shuts down before another write, the destination is gone, and the room does not offer it again in that activation because `_destination` is latched.
The design documents say the value is persisted with the board row; `RoomGrainTest_Notes.cs:171` covers exactly this path but asserts only in memory.
Merge instead: a private non-writing `AdoptDestinationIfUnset` used by the subscribe path, and a public `SetDestinationIfUnset` that calls it and then `WriteOrDeleteIfUnused()`; extend the room test to read the stored state.
(criteria/engineering.md §4 functions doing less than their name claims; the design document's recorded contract; also confirmed by the code-review lens)

**E7** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:321` · *introduced* · no-client `Item.GetProperties` can never settle, ignores `timeoutMs`, and throws for the Producer frame.
`handleItemGetPropertiesRequest` resolves only when an `Item.Properties` notification for the requested id arrives.
In all-own-items mode (after `Item.SubscribeToUpdates` or a request for a non-frame item) `onOwnItems` (line 142) notifies only items whose properties changed since the last fetch, so a repeat request for an unchanged item never resolves.
In both modes an item missing from the response, or a failed `GetOwnItems` page call, leaves the promise pending and the listener registered; `timeoutMs` (line 199) is never read, while the real client rejects after the timeout.
`Producer.cshtml:149` calls `request('Item.GetProperties')` with no `messageData`, so line 316 throws a `TypeError` on `messageData.itemId` and the Producer frame's activate button is never set up in no-client mode.
`ItemHandling.update` (lines 103-115) runs its `.then` after the `.catch`, so a failed page call also logs a misleading "Malformed page call result!".
No shipped frame hits the hang today; the mode is reachable in production because `itemFrame.ts:42` switches it on for any frame opened as a top-level page.
Merge instead: have `update()` return the fetched map and resolve `GetProperties` from it, reject when the item is absent or the page call fails, honour `timeoutMs`, read `messageData?.itemId`, and move the `.catch` after the `.then`.
(anti-patterns.md Unbounded blocking; confirmed by the code-review lens)

**E8** · **minor** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:176` · *introduced* · board and thread data versions are merged by `Math.Max`; polling can stop detecting board changes.
`NotesPage.DataVersion` is the board's counter for `GetPosts` and the thread's own counter for `GetComments` (design document line 140).
`GetPostsPageWithPreviewComments` takes the maximum over both, and the SPA compares that number, and every comment page's thread version (`ThoughtsModel.ts:555`), against the polled board counter with `version > this.dataVersion`.
A thread counter runs ahead of the board's in normal operation: a comment create that culls commits the thread twice and the board once.
Once a thread value exceeds the board value, board advances below it are ignored and the page stops refreshing until the board catches up.
Merge instead: keep the board's `DataVersion` as the merged page's version and leave thread versions out of the comparison, or give the wire record two named fields.
(criteria/engineering.md correctness; code contradicts its own design document)

**E9** · **minor** · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:88` · *introduced* · the assembly walk is one level deep, and the design document promises a rebuild on a miss that the code does not do.
`LoadAllReferencedAssemblies` adds a reference to `visited` at line 92 before loading it; when that assembly is dequeued, `visited.Add(assembly.FullName)` at line 88 returns false and its own references are never enqueued.
Which grain interfaces the registry knows therefore depends on what was loaded before the first call; today the silo loads every n3q assembly, so no observer entry is lost.
The notes design document (line 438) says the registry "rebuilds once on a miss"; `InitIfNeeded` builds once behind `_initComplete` and a miss returns null, after which `NoteBoardGrain.cs:582` evicts the persisted observer entry.
Merge instead: track visited on dequeue (separate enqueued set) so loaded assemblies are walked; correct the document sentence or implement the rebuild.
(criteria/engineering.md correctness; anti-patterns.md Hidden temporal coupling; code contradicts its own design document)

**E10** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:884` · *introduced* · three bare `catch` blocks drop notifications with no log line.
`NoteBoardGrain.GetNoteOrNull` (lines 878-886) returns null on any exception, which every timer callback reads as "note gone"; `NoteBoardNotificationSender.cs:212` returns without sending on a failed `GetUserReactions`; `NoteBoardNotificationSender.cs:278` breaks out of the paging loop on a failed `GetComments`, so some children are notified and others not.
A thread timeout or activation failure then means users get no reply, reaction, edit or delete notifications and nothing in the silo log says so; the sibling catch at lines 348-351 logs.
Merge instead: log a warning with the board log data in each of the three catches.
(anti-patterns.md Swallowed errors / silent failure; nine3q CLAUDE.md Failure signaling; the CTO reliability finding on the same lines is merged here)

**E11** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:594` · *introduced* · a `try`/`catch` around a `[OneWay]` observer call cannot see the observer's exception.
`OnNoteBoardPostCountChanged` is declared `[OneWay]` (`INoteBoard.cs:18`); its task completes when the message is dispatched and an exception in the observer never returns.
The catch and its message "notification threw for an observer; preserving subscription" describe handling that cannot happen; only a send-side failure could reach it, and nothing would evict the subscription anyway.
Merge instead: call the observer without the `try`/`catch`, or word the message for a send-side failure.
(criteria/engineering.md §2 Simplicity)

**E12** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsNoteView.ts:258` · *introduced* · avatars from hosts without CORS headers never load; the default avatar replaces them.
`crossOrigin = 'anonymous'` makes the browser issue a CORS request; an image host without `Access-Control-Allow-Origin` fails the load outright, and the error handler (lines 261-267) swaps in the default avatar.
Author image URLs come from `UserProfile.ImageUrl`, which the avatar grain derives from any absolute host the user's animation XML names; only images WebIt hosts itself survive.
The comment at lines 296-302 describes a tainted-canvas fallback with the real image, which cannot occur with `crossOrigin` set.
Merge instead: load the displayed image without `crossOrigin`, and run the opacity fit on a second `Image` with `crossOrigin` set, keeping the CSS crop on failure.
(desired-patterns.md Fail closed, degrade gracefully)

**E13** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsController.ts:228` · *introduced* · `await runDataRefresh()` returns before the refresh when one is in flight; the new post is not scrolled to.
`runDataRefresh` (lines 297-312) returns at once with `isRefreshRequested = true` when a refresh is running; `submitNewPost` then looks up the post's view before the trailing pass applies the page and skips the scroll and highlight; `onDeleteClicked` (line 476) has the same shape.
The doc comment at lines 294-295 promises post-reconcile state to awaiting callers.
Merge instead: keep the in-flight promise in a field and return it to coalesced callers so an `await` ends after the pass that includes the caller's change.
(anti-patterns.md Hidden temporal coupling)

**E14** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsApi.ts:173` · *introduced* · JSON responses are cast to the expected type without narrowing.
After `is.object`/`is.nil` the body is returned as `parsed as T` for `NotesPage`, `WireNote` and the data-version object.
The project rule types boundary data `unknown` and narrows it with the `is.*` guards; `WebsocketMessage.ts` and the same PR's `ThoughtsDraftStore.getRecordOrNull` show the form.
A changed server shape shows up as undefined ids deep in the model instead of a `ThoughtsApiError` at the boundary.
Merge instead: guards for the three shapes, and `ThoughtsApiError('Unexpected response shape')` on mismatch.
(nine3q CLAUDE.md TypeScript Code Style, Types & modules)

**E15** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsDraftStore.ts:85` · *introduced* · the unsent draft is not tied to the user; the next identity on the same browser gets it.
The single `localStorage` record is matched on room, `isForNewNote`, post and note only; it carries no user id although the controller knows `settings.userId`.
After an identity reset or account switch in the extension, the next user opening the same room's frame gets the earlier user's unsent text loaded into the editor.
Merge instead: store `userId` in the record and compare it in `isRecordFor`.
(criteria/engineering.md correctness, data isolation)

**E16** · **minor** · `Items/n3q.WebIt/ItemFrameModel.cs:129` · *introduced* · `OnPostGetOwnItems` returns the context inventory without checking it is the user's; for a room frame that is the room's items.
`InventoryGrain` is the inventory named by the signed context (`Context.InventoryId`), verified only against the user's own token.
For a room item frame (the Thoughts case) the "own items" the emulation receives are the room's items; a context with any inventory id and the user's token is also served.
The repository owner's PR note counts the data exposure as pre-existing because the base `ItemApiController` already serves any inventory with the user's token (debt E59); the handler and its wrong-list behaviour are new.
Merge instead: read the user's inventory explicitly for "own items", or guard with `Context.InventoryId == Context.UserId` as the owner suggested.
(criteria/engineering.md authorization; anti-patterns.md over-collection; the CTO privacy finding on the same line is merged here)

**E17** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:294` · *introduced* · the sort-score formula and the `"upvote"`/`"downvote"` meaning ids are written in two places each.
`NoteBoardGrain.GetSortScore` (lines 294-299) and `NoteThreadGrain.GetSortScoreFromEntry` (lines 1334-1339) compute the same `CreatedAtUnixSeconds + AdjustedVoteSum + ChildBoostFactor * max(0, BestDescendant)` over two record types, and the two sort lambdas repeat "score desc, id desc"; the design document states the formula once.
`"upvote"` and `"downvote"` are literals in `NoteThreadGrain.cs:1314-1316`, in `SiloConfigDefinition.cs:107-108` and in `ThoughtsModel.ts:98-99`; a rename in one place leaves scoring reading zero votes with no compile error.
Merge instead: one static scoring helper next to `NotesScoringConfig` called by both grains, and two meaning-id constants in `n3q.AppInterfaces` served to the SPA in `NotesConfig`.
(anti-patterns.md Duplicated source of truth; raised by four reviewers, merged)

**E18** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:360` · *introduced* · the per-user reactions LRU cache is implemented twice with different eviction rules.
`NoteBoardGrain` (lines 112-115, 360-393) and `NoteThreadGrain` (lines 106-110, 1571-1608) each carry a dictionary, a linked list and a node index with their own move-to-front and evict loops.
They already differ: the board clears everything when the cap is zero or less and seeds unknown users; the thread never seeds and simply does not cache.
The design document records two caches with one cap, not two implementations.
Merge instead: a small `LruCache<TKey, TValue>` in `n3q.Grains` or `n3q.Tools` used by both grains.
(criteria/engineering.md §5; anti-patterns.md Duplicated source of truth)

**E19** · **minor** · `Items/n3q.Grains/NoteThreadGrain.cs:446` · *introduced* · the cascade-removal walk is written twice, the post-order traversal three times, the note-delete `try`/`catch` seven times, one with the wrong note id in the log.
`Delete` (lines 440-461) and `CullOne` (lines 641-658) contain the same walk up through childless soft-deleted ancestors; the explicit-stack post-order loop appears in `GetCommentIdsLeavesFirst` (671-691), `RebuildDerivedMaps` (1154-1181) and `RebuildBestDescendantBottomUp` (1188-1220); the `try`/`catch` around `GetNoteGrain(..).Delete()` with the log line "NoteGrain.Delete failed" is copied at lines 431, 452, 553, 563, 633, 650 and 748.
Line 454 logs `noteId`, the originally deleted note, while the note being deleted is `cursor`, so a failed cascade delete names the wrong note.
Merge instead: one `DeleteNoteGrainLogged(long noteId)`, one `RemoveChildlessSoftDeletedAncestors(...)` used by `Delete` and `CullOne`, and one post-order traversal the three callers pass an action to.
(criteria/engineering.md §4, §5; raised by three reviewers, merged)

**E20** · **minor** · `Base/n3q.WebEx/Controllers/ConfigController.cs:19` · *introduced* · the Thoughts frame URL and window options are defined in WebEx and in the item template, kept in sync by comment.
`ConfigController.cs:16-25` and `Items/n3q.Items/Templates/Thoughts.cs:81-90` each carry the URL pattern and the six option values (600, 700, 150, Window, hidden, resizable) with a reciprocal "update the other file" comment.
Two separately deployed services will show the frame at different sizes once one copy changes.
Both projects reference `App/n3q.AppInterfaces`, so a shared home exists; the design document states the sync requirement, not a decision to keep two copies.
Merge instead: a small static class in `n3q.AppInterfaces` holding the route segment and the options, read by both.
(anti-patterns.md Duplicated source of truth)

**E21** · **minor** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:365` · *introduced* · sanitize-then-length-check is written twice in WebIt.
`ThoughtsModel.GetSanitizedText` and `NotesCommandlineCommands.SanitizeNoteTextAndCheckLength` (lines 137-145) both call `NotesHtmlPolicy.Sanitize`, compare to `NotesMaxTextLength` and throw with near-identical text.
`NotesHtmlPolicy`'s own doc comment describes one rule for every write endpoint.
Merge instead: one `GetSanitizedTextOrThrow(text, maxLength)` on `NotesHtmlPolicy`.
(criteria/engineering.md §5)

**E22** · **minor** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:204` · *introduced* · code with no caller.
`OnGetNoteWithAncestorsAsync` and `ThoughtsApi.getNoteWithAncestors` (`ThoughtsApi.ts:49`) are called by nothing but one test; the deep-link case is served by `OnGetAsync` embedding the result at page load, as the design document describes.
`notesPostPageSize`, `notesCommentPageSize` and `notesCommentMaxDepth` travel from config through `Thoughts.cshtml.cs:42-44` and `Thoughts.cshtml:206-208` into `ThoughtsControllerSettings` and no TypeScript reads them.
`waitForRenderComplete` (`domSchedulingUtils.ts:9`) has no caller.
`INoteBoard.HasPosts` and `INoteThread.GetDataVersion` are called only from tests.
Merge instead: remove each until a caller exists, or wire the SPA path that needs the handler and say so in the design document.
(anti-patterns.md Premature abstraction / speculative generality; the scope finding on the handler is merged here)

**E23** · **minor** · `Items/n3q.GrainInterfaces/INoteBoard.cs:10` · *introduced* · comments and the design documents name methods, classes and behaviour that do not exist in the code.
`INoteBoard.cs:10` names `SubscribeHasPostsObserver` (the method is `SubscribePostCountChangedObserver`) and line 15 says the callback fires when the count reaches 0 or leaves 0, while the board calls it after every create and delete.
`NoteBoardGrain.cs:598` logs "HasPosts notification threw"; the region at line 278 says `NoteBoardNotifier`; `NoteBoardGrain.cs:98` and `NoteThreadGrain.cs:85` point at a `NoteGrain.TableName` comment that does not exist.
`NoteBoardNotificationSender.cs:208` says "pae revert check".
`ThoughtsModel.ts:3` points at `INoteBoardGrain`; the types mirror `ThoughtsWire.cs`.
The Thoughts document (line 195) and `NotesVocabulary.cs:15` name `buildThoughtsExtensions`; the export is `makeThoughtsExtensions`.
The notes document line 470 says `SetReactions` rejects unknown reaction ids; `NoteThreadUserReactionsGrain.cs:102` drops them.
Both documents' file-level plan tables list `IUser.SendSystemInstantMessage`, `UserGrain`, `UserInstantMessages`, `IRoom.InternalRoomIdFromClientRoomId` and three extension files as modified by this feature; the server members exist at the merge-base and none of those files is in the diff.
Merge instead: correct each mention to the current name and behaviour, and mark the plan tables' pre-existing and extension-side entries as such.
(criteria/engineering.md §4 accurate names; code that contradicts its own documentation; raised by four reviewers, merged; the registry sentence is in E9)

**E24** · **minor** · `Items/n3q.Common/Changes.cs:10` · *introduced* · the no-client entry is not written from the user's view, and the `site.ts` clipping fix that changes the Person and ItemShop frames has no entry.
"No-client mode for item frames" names the implementation; the Change Log rule asks for one sentence from the user's point of view, e.g. "Item frame pages work when opened in a plain browser tab without the weblin client", or no line if the mode is for development only.
The `site.ts:175` change from `clipPath` to `clip-path` and from string to numeric negation makes `clipImageElemByOpacity` apply a clip for the first time; its callers are `Person.cshtml:152` and `ItemShopUiHelper.ts:7` via `domImageTools.ts:40`, so those frames render differently after this PR, and no "Fix" line, commit message or description says so.
Merge instead: reword the no-client line, add a "Fix" line for the clipping, and check the Person frame once.
(nine3q CLAUDE.md Change Log)

**E25** · **minor** · PR title and description · *introduced* · the title is the branch name; the description names one of four things that ship.
"Feature/aw thoughts" becomes the merge commit subject; "Implements the Thoughts feature." omits the notes grain infrastructure, the no-client mode, the 29 vendored editor bundles with their import map, and the clipping fix.
Merge instead: a title such as "Thoughts: persisted notes and comments in rooms (notes grains, tiptap editor, no-client mode for item frames)" and one description line per shipped part.
(lenses/engineering.md PR metadata; always minor)

**E26** · **minor** · `Items/n3q.WebIt/Scripts/is.ts:76` · *introduced* · `is.stringsObject` is deprecated while its three callers stay on it.
`WebsocketMessage.ts:43`, `:61` and `:85` still call it; only `WeblinClientApi.ts:110` uses the new `stringMapObject`.
Half a rename leaves two names for one guard and a warning on code nobody asked to change.
Merge instead: add the new guards without deprecating the old one, or finish the rename at the three call sites.
(desired-patterns.md Intentional, called-out refactors)

**E27** · **minor** · `Items/n3q.WebIt/Pages/NotesCommandlineCommands.cs:8` · *introduced* · partial coverage.
The seven admin commands parse arguments, validate the board id, sanitise and call the board; the board methods are covered at the grain tier, the command layer runs in no test, and no test in the repository runs any commandline command.
`Thoughts.MakeThoughtsItemCreateOrUpdateProperties` (`Templates/Thoughts.cs:9`) computes the badge text, radius, SVG, size and position; the room tests assert only that an item with the Thoughts template exists, and `n3q.Items.Test/TemplateTest.cs` could assert the values.
The import map for the vendored editor (`AppPageModel.cs:86`) is rendered into every item-frame page; the page tests that fetch rendered HTML assert nothing about it, and the `File.Exists` skip at line 189 silently drops an entry whose file is missing.
The room's unsubscribe from the board when its last user leaves (`ThoughtsItemManager.cs:42`) runs in no test; `NoteBoardGrainTest.cs:836` shows the effect is observable through `GetDestinationOrNull`.
Merge instead: a WebIt test running two admin commands; a `TemplateTest` case for counts 1, 12 and 1000; an importmap assertion in a page test; a `RoomGrainTest_Notes` case with a leave.
(lenses/engineering.md Behaviour coverage, minor)

**E28** · **minor** · `Items/n3q.GrainTests/NoteBoardGrainTest.cs:1261` · *introduced* · weak tests.
`StateSurvivesDeactivation` never deactivates the grain: both blocks address the same activation, so storage is never read back and its one assertion duplicates `StatsMirrorState` (line 1125); add `await g.Deactivate()` between the blocks or delete it.
`FullLifecycleStaysConsistent` asserts `>= 6`, `>= 2`, `>= 3`, `>= 3` (lines 3144-3150) and a lower bound on the comments page (line 3092) where the exact counts are known; a double-counting regression passes.
`NoteGrainTest.cs:159` `GetReturnsPersistedState` is a store-and-retrieve within one activation (`GetState` is a pass-through getter), and `NoteGrainTest.cs:150` `InitializeSetsParentIdForComment` mirrors the one line `ParentId = parentId`; delete both or fold the assertion into `InitializePersistsSuppliedValuesWithOriginalStatus`.
(criteria/test.md Useless tests; Meaningful unit tests)

**E29** · **minor** · `Items/n3q.Grains/NoteGrain.cs:88` · *introduced* · untyped `throw new Exception("Already initialized!")`.
The rule marks untyped exceptions as legacy; the same file throws `InvalidOperationException` for the same kind of state violation at lines 120, 136 and 162.
Merge instead: `throw new InvalidOperationException($"Note {_noteId} is already initialized")`.
(nine3q CLAUDE.md C# Code Style, Failure signaling)

**E30** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:129` · *introduced* · null-forgiving `!` on own-code null flow in four places.
`_notificationSender = null!` (line 129) forgives a field only set in `OnActivateAsync`; `result.Summary!` (line 861) forgives a value line 852 has just tested; `n.ParentId!.Value` (`NoteThreadGrain.cs:764`) forgives a value the filter at line 738 checked on a different variable; `State.Post!` (`NoteThreadGrain.cs:798`) forgives a value line 787 just assigned.
Merge instead: construct the sender in the constructor; branch on `result.Summary is null`; carry `(noteId, parentId)` pairs from the filter; keep the post in a local.
(nine3q CLAUDE.md C# Code Style, Null-forgiving)

**E31** · **minor** · `Items/n3q.Grains/ThoughtsItemManager.cs:11` · *introduced* · `static readonly` fields named `_camelCase`.
`_thoughtsItemSearchProperties` and `GrainInterfaceRegistry.cs:13` `_lock`; the rule says non-reassignable statics stay PascalCase, even lock objects.
(nine3q CLAUDE.md C# Code Style, Private fields)

**E32** · **minor** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:375` · *introduced* · four helpers against the naming rules.
`CheckAuthor` returns an error result or null and is side-effect-free; `GetAuthorMismatchResultOrNull` says what it does.
`EnrichAuthors` (`NoteBoardGrain.cs:1092`) mutates nothing and returns the authors map; `GetAuthorsOf`.
`ParseOptionalLong` (`StringExtensions.cs:38`) returns null for absence; the house form is `ToLongOrNull`.
`EnqueueImmediateIm` (`NoteBoardNotificationSender.cs:347`) queues nothing and awaits `SendSystemInstantMessage`; the boundary rule names a sending method `SendXxx`.
(nine3q CLAUDE.md General Code Style, Get prefix, …OrX, message naming)

**E33** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:68` · *introduced* · new `any` on three lines.
Lines 68 and 72 type the fallback handlers `{[p: string]: any}` although the listeners only ever receive `ItemPropertiesChangedNotification` or `ItemGoneNotification`; line 199 declares the new `request` as `Promise<any>` where every branch returns `Promise<{}>`.
Lines 8 and 27 are the moved legacy alias and declaration and are not counted.
Merge instead: the notification union for the handlers, `Promise<unknown>` for the method.
(nine3q CLAUDE.md TypeScript Code Style, Types & modules)

**E34** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:318` · *introduced* · a block-form `eslint-disable` turns the rule off for the rest of the file.
`/* eslint-disable prefer-promise-reject-errors */` has no matching enable; line 355 in the same file uses `// eslint-disable-next-line`, which is also the comment form the rules allow.
(nine3q CLAUDE.md General Code Style, comments)

**E35** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsEditor.ts:247` · *introduced* · toolbar buttons built by a write-into-parent helper.
`makeButton(groupElem, key, action)` assembles a fixed `button > span.text` with `createElement` and appends into the parent it is handed; the rule bans write-into-parent helpers and asks for a template string with `as.Html()`; `ThoughtsNoteView.makeNoteActionButton` in the same PR shows the expected shape.
(nine3q CLAUDE.md TypeScript Code Style, DOM building)

**E36** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsNoteView.ts:291` · *introduced* · hard-coded English `(edited)` marker.
Every other user-visible string comes from `controller.texts`; the German locale shows the English marker.
Merge instead: a `thoughts.post.editedMarker` key in both languages' `defaultTexts`.
(Thoughts feature convention, `ThoughtsController.ts:22-25`)

**E37** · **nit** · `Items/n3q.Grains/NoteBoardGrain.cs:990` · *introduced* · `GetPosts` (85 lines) builds the author map by hand where `EnrichAuthors` (line 1092) exists and strips empty reaction sets by hand as `MakePageFromCommentPreview` does at lines 524-528.

**E38** · **nit** · `Items/n3q.Grains/NoteBoardNotificationSender.cs:74` · *introduced* · `OnNoteCreated` writes the four one-shot timer options inline; `OnEdited` and `OnReactionAdded` use `_board.RegisterOneShotTimer`, which exists for this.

**E39** · **nit** · `Items/n3q.Grains/NoteThreadGrain.cs:893` · *introduced* · `ParseOptionalLong` is wrapped in `catch (FormatException)` here and as `NoteBoardGrain.GetIdOrNull` (lines 1245-1252); `long.Parse` also throws `OverflowException`, which would abort the repair sweep the comment says must survive a corrupt row. A `TryParse`-based `GetLongOrNull` fixes both.

**E40** · **nit** · `Items/n3q.Grains/NoteGrain.cs:116` · *introduced* · the two-status soft-deleted test is inline at `NoteGrain.cs:116`, `:135`, `:161`, `:189`, `NoteBoardNotificationSender.cs:322` and as a private helper in `NoteThreadGrain.cs:1018`; `NoteThreadState` (`NoteThreadGrain.cs:50`) splits the thread key by hand while `INoteThread.ParsePrimaryKey` defines the split. A `NoteStatus` extension and the key helper remove the copies.

**E41** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsController.ts:159` · *introduced* · the controller registers a document listener, a `setInterval` and a model listener and owns `ThoughtsPostsView`, whose `stop()` nothing calls; every other class in the feature has `stop()` with `isStopped`. No observable effect today since frames are destroyed on close. (nine3q CLAUDE.md Lifecycle)

**E42** · **nit** · `Items/n3q.WebIt/Pages/ItemFrame/Thoughts.cshtml.cs:119` · *introduced* · `OnGetPostsAsync` and ten more named handlers carry the `Async` suffix. Razor Pages matches named handlers with or without the suffix, and every existing named handler in WebIt, including this PR's `OnPostGetOwnItems`, omits it. The rule's example names the unnamed `OnGetAsync`, which makes the named case arguable; the repository's own practice is suffix-free. (nine3q CLAUDE.md C# Code Style, no `Async` suffix)

**E43** · **nit** · `Items/n3q.Grains/NoteBoardNotificationSender.cs:49` · *introduced* · tight `new()` on 167 added C# lines (`ThoughtsPageTest.cs` 82, `SiloConfigDefinition.cs` 26, `NotesVocabulary.cs` 14, `NoteBoardGrainTest.cs` 14, `NotesCommandlineCommands.cs` 9, `NoteBoardNotificationSender.cs` 6, others 1-3) against 150 spaced `new ()`. (nine3q CLAUDE.md C# Code Style, target-typed new)

**E44** · **nit** · `Items/n3q.Grains/NoteThreadUserReactionsGrain.cs:63` · *introduced* · about 220 added lines exceed 120 columns, 69 outside tests; shortenable examples: this field initializer (143), `GrainInterfaceRegistry.cs:16` (146), `NoteGrain.cs:186` (147), `NotesCommandlineCommands.cs:76` (185), `WeblinClientApi.ts:66` (159), `:138` (156), `:214` (161). The `[PersistentState]` parameter lines are the awkward case the rule allows. (nine3q CLAUDE.md General Code Style, soft 120-column limit)

**E45** · **nit** · `Items/n3q.GrainInterfaces/INote.cs:87` · *introduced* · `TableName` constants sit last in `INote` and `INoteThreadUserReactions.cs:69`; the static `From` factories in `ThoughtsWire.cs` follow the properties; `NoteBoardGrain.cs:137` declares a field after the `State` property; `ThoughtsController.ts:61-80` and `WeblinClientApi.ts:57-72` put private fields before public ones. (nine3q CLAUDE.md General Code Style, member order)

**E46** · **nit** · `Items/n3q.Grains/ThoughtsItemManager.cs:23` · *introduced* · the constructor's opening brace is on the declaration line; every other new constructor uses Allman. (nine3q CLAUDE.md C# Code Style, Allman braces on declarations)

**E47** · **nit** · `Docs/Features/Notes/Notes Infrastructure - Boards, Posts, Comments, Reactions.md:443` · *introduced* · "true iff" in prose. (nine3q CLAUDE.md General Code Style, write plain "if")

**E48** · **nit** · `Shared/n3q.OrleansGrainTools/GrainInterfaceRegistry.cs:14` · *introduced* · `_initComplete` has no predicate prefix and no verb; `WeblinClientApi.ts:62-65` `handleOwnItems`, `inOwnItemsUpdate`, `ownItemsUpdateRequested` are state flags, not option members. (nine3q CLAUDE.md General Code Style, boolean names)

**E49** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:204` · *introduced* · the `request()` switch has 32 `case` lines and a `default` of the form `case 'X': return …;`; `on()` in the same file follows the braced `} break;` rule. (nine3q CLAUDE.md TypeScript Code Style, switch cases)

**E50** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:101` · *introduced* · missing semicolons at `WeblinClientApi.ts:101`, `:147` and on the new or rewritten `itemFrame.ts` lines 42, 43 and 61. (nine3q CLAUDE.md TypeScript Code Style, semicolons always; legacy is converted when touched)

**E51** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:2` · *introduced* · `ConsoleLogger` is imported and never used; `is` and `iter` are imported after `SiteCommunicator` and `EventListeners`; type-only names are imported in value form at `ThoughtsController.ts:6`, `:8`, `:12`, `ThoughtsPostsView.ts:4-5`, `ThoughtsCommentsView.ts:5`, `ThoughtsEditor.ts:1`. (nine3q CLAUDE.md TypeScript Code Style, imports)

**E52** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsEditor.ts:444` · *introduced* · `getSubmitHtmlOrNull` calls `setContent` and assigns `htmlSource` in source mode before returning; move the normalisation into `onSubmitClicked`. (nine3q CLAUDE.md General Code Style, `Get` promises side-effect-freedom)

**E53** · **nit** · `Items/n3q.WebIt/Scripts/DomUtils.ts:35` · *introduced* · `} catch (_error) { /* Just fall back to Element.append. */ }` is a single-line braced body with a block comment. (nine3q CLAUDE.md General Code Style)

**E54** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsApi.ts:45` · *introduced* · `String()` wraps ids that are already strings throughout `ThoughtsApi.ts`, `ThoughtsNoteView.ts:98`, `ThoughtsCommentsView.ts:263`, `ThoughtsPostsView.ts:208`; `ThoughtsReactionsView.ts:106-107` tests `is.nil(config)` after a `continue` on nil; `ThoughtsModel.hasCommentsLoaded(noteId)` takes a post id; `NotesCommandlineCommands.cs:195` and `:215` rebuild indented serializer options per call.

**E55** · **nit** · `Items/n3q.WebIt/AppPageModel.cs:86` · *introduced* · the 29 vendored tiptap, prosemirror and linkify bundles are identified only by the jsDelivr banner inside each minified file; the bootstrap and jquery folders beside them carry LICENSE files and `web3-1.6.1.min.js` carries its version in the name. A short manifest and the MIT license texts are missing.

**E56** · **nit** · `Items/n3q.Grains/RoomItems.cs:94` · *introduced* · the only change to `RoomItems.cs` is a removed blank line; `WebItConfigDefinition.cs:46`, `:49` lose trailing spaces and `ClientConfig.cs:86`, `:165` change line endings on lines no other hunk touches. (anti-patterns.md Drive-by churn)

**E57** · **nit** · `Items/n3q.Grains/ThoughtsItemManager.cs:17` · *introduced* · `_logData` is never given the room id, so "Subscribe to note board failed!" and its two siblings (lines 64, 120, 131) cannot be attributed to a room. (nine3q CLAUDE.md Grain logging)

**E58** · **nit** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsDraftStore.ts:58` · *introduced* · a `QuotaExceededError` or blocked storage on save goes to `handleErrorVerbose`, which is off in production; the draft feature then stops with no record. Log once at warning level with a latch. (anti-patterns.md Swallowed errors)

Rules that look stale, for the human: the C# Grain logging rule says to stamp identity with `App.Log.SetValue(LogData.Key.ObjectId, …)` in `OnActivateAsync`; `ICallbackLogger` has no `SetValue`, and every grain in the repository, this PR included, passes a `LogData` per call instead. The rule as written cannot be followed.

### Code-review — needs-work: one confirmed defect of its own, two confirmed defects shared with Engineering

Protocol: two CLAUDE.md compliance finders (C#, TypeScript), one diff-only bug finder over the full 22k-line diff minus vendored libraries, one context-aware bug finder; every candidate validated by an independent agent with high confidence required.
A CRLF line-ending candidate was refuted: every HEAD blob checked has zero CR bytes and the repository's `.editorconfig` requires LF.
Cleared by the context finder: authorship checks compare the exact `(postId, noteId)` the grain call uses; soft-deleted notes have an empty author so nobody passes the check on them; the sanitizer runs on all four write paths and the two moderator edits; `innerHTML` receives only the schema round-trip; the grain call graph has no re-entrant cycle below the board (the board → user leg is CTO1); the counters match across delete, cascade, cull and `DeleteAll`; the row-key range `"{postId}:"`…`"{postId};"` does not pick up other post ids; removed or changed lines in the modified files keep their behaviour for existing callers.

**CR1** · **major** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/Thoughts/ThoughtsController.ts:398` · *introduced* · CONFIRMED · comments are always requested as page 0; "Show more replies" reloads the same page.
`anchorNoteIds.map(anchorNoteId => this.api.getComments(postId, anchorNoteId, 0))` is the only `getComments` call; neither controller nor model keeps a page index per parent (the only page state is `nextPostsLoadPage` for posts).
The server pages a parent's direct children with `skip = page * pageSize` (`NoteThreadGrain.cs:189-209`, page size `NotesCommentPageSize` default 20) and appends a marker `TruncatedParentId = parentNoteId, TruncatedChildCount = remaining`; a second marker of the same shape marks a depth cut at `NoteThreadGrain.cs:1448`, and the client cannot tell the two apart.
Clicking the button calls `onLoadTruncated` → `loadSubtrees` → the same request; `ThoughtsModel.applyTreeStructure` replaces the parent's `loadedChildIds` with the page's children, so the same twenty come back with the same "N more" button.
A thought with more than twenty direct comments, or a comment with more than twenty direct replies, can never show the rest; the data-version refresh path has the same limit.
`NoteThreadGrainTest.cs:771` tests server paging of page 1, which works; no WebIt or TypeScript test fetches a second comments page.
The Thoughts document (lines 149, 169, 262) specifies paging per parent with load-more at truncation points.
Merge instead: keep a loaded page count per anchor in the model, increment it on a paging-marker click, fetch pages 0..n-1 in `loadSubtrees` and apply them as one subtree (dropping the paging marker from all but the last page); or add a server parameter that returns the first n pages in one call. Add a test: 25 direct comments, click load-more twice, expect 25 loaded and no marker.

Confirmed candidates shared with Engineering, one write-up each there: `SetDestinationIfUnset` never persists (E6); no-client `Item.GetProperties` hangs, ignores `timeoutMs`, and throws for the Producer frame (E7).

Convention candidates validated and reported under Engineering: untyped exception (E29); null-forgiving `!` (E30); `any` (E33); block `eslint-disable` (E34); `Async` suffix (E42); tight `new()` with the corrected count of 167 (E43); long lines (E44); member order (E45); boolean names (E48); unbraced switch cases, 33 not 38 (E49).
Validated as not a finding: the member order of `ThoughtsEditor.ts`'s two private static helpers, which sit beside their callers under the rule's readability clause.

### Scope — over-scope, delivered: every specified behaviour has a code path and a test; one separable feature and one global policy ship unrequested

Request source: PR description, supplemented by the commit messages and the two design documents the PR adds, treated as the author's specification and verified against the code.

Coverage verdict: Over-scope.
Delivery verdict: Delivered.
Every server-side behaviour the Thoughts document specifies has a code path and a test: the badge item lifecycle (`ThoughtsItemManager.cs`, `RoomGrainTest_Notes`), subscribe/unsubscribe, destination hand-over, paginated reads with preview comments and data-version polling, create/comment/edit/delete with author check and sanitized length-capped HTML, reactions with exclusivity groups, deep links and the `clientRoomId` override, profile refresh on open, the WebEx `thoughts` section, moderation through the admin commandline only, the config keys, and the `Changes.cs` entry.
Explained, not gaps: the extension-side items in the Thoughts document live in the extension repo, whose local checkout reads `thoughts.enabled` and `thoughts.itemFrameProperties`; the server members the plan tables list as modified already exist at the merge-base.
Supporting, shown from code dependency: `GrainInterfaceRegistry` (the board's observer dispatch), the row-key-range query (`NoteThreadGrain` delete-all and repair), the two `n3q.Tools` helpers, `UserProfile.HasSameIdAndNameAndAvatar`, `DomUtils.appendElemAndTryToPreventIframeReloads`, the import map, the `WebAppPostArgList` test helper, the `HtmlSanitizer` package, the vendored bundles, and the `site.ts` clipping fix (needed by `ThoughtsNoteView.ts:320`; its missing change-log line is E24).
Below the bar: the `.xcf` and `.jpg` design sources follow the repository's practice of checking in `.psd` sources; the `notifications.enabled` flag serves the extension's existing default.

**SCP1** · **minor** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/WeblinClientApi.ts:165` · *introduced* · Beyond request · no-client mode for item frames is a separate feature the Thoughts request does not depend on.
Neither design document mentions a client-less item-frame mode; when the page is framed by the client, `ItemFrameUtils` constructs the real `WeblinClientApi` (`itemFrame.ts:59`) and the no-client class is never instantiated; at the Thoughts commit the global declaration already carried `handleException`, so the Thoughts front end compiled before this commit existed.
The commit changes behaviour for every `ItemFrameUtils` page opened unframed (`itemFrame.ts:42`), adds the `OnPostGetOwnItems` handler whose only caller is the emulation, adds a page-wide referrer policy (SCP2), and resyncs about 750 lines of copied client CSS with a manual procedure.
It is the last commit on the branch and touches no Thoughts file, so it is separable.
Disposition: split commit 5a0b6fb9 into its own PR; if it stays, say so in the description and decide E16's guard there.
(lenses/scope.md Step 3 Beyond request; anti-patterns.md Gratuitous rewrite / scope creep; the Engineering scope dimension's finding on the same line is merged here)

**SCP2** · **minor** · `Items/n3q.WebIt/Pages/ItemFrame/Shared/_Layout.cshtml:5` · *introduced* · Beyond request · a referrer policy for every item frame page, with no stated reason.
`<meta name="referrer" content="origin">` changes what every item frame page sends as Referer, not only Thoughts; it arrives in the no-client commit with no mention in its message or either design document.
Notes cannot load external resources (no `img` in the vocabulary) and the only outgoing navigation from note content already carries `rel="noopener noreferrer"`, so no Thoughts code depends on it.
If the purpose is to keep the signed `?context=` token out of Referer headers, that is worth keeping: give it one comment line and a mention in the description, and move it into the Thoughts commit; otherwise drop it.
(lenses/scope.md Step 3; desired-patterns.md Intentional, called-out refactors)

### CTO — needs-work: the grain layer is sound for always-on operation, but the merge as it stands adds 30-second stalls, a persistent copy of every note in the logs, and an untested, undocumented mode switched on in every unframed item frame

Two-paths judgement.
The notes grains rely on path B with path A behind it: the tests run the real grains on the isolated silo against storage, deactivate and read back cold, and the lifecycle test compares the full read API against an in-test model after every mutation; the 762-line design document describes the layer.
The Thoughts item and room integration: path B through `RoomGrainTest_Notes` running the real `RoomGrain.HandleUserRequest`, path A through the Thoughts document.
The WebIt handlers and HTML policy: path B, since `ThoughtsPageTest` runs the real program through `WebApplicationFactory` with only the cluster client and config replaced.
`GrainInterfaceRegistry` and the storage change: path B.
The TypeScript front end: no test tier exists in WebIt, so path A has to carry it; the code is readable (controller, model, API and views with doc blocks, the server as the only authority on ordering, explicit lifecycle) and the Thoughts document describes polling, reconcile, the editor and the two render gates in enough detail, with the caveat that the document is not where the root rule looks for it (E3).
The no-client mode: neither path (CTO3).
Cleared: grain state is bounded by the configured caps; no grain is reentrant and the call graph is a tree apart from CTO1; no completeness-by-timeout anywhere; `SetReactions`, edit, delete, subscribe and unsubscribe are idempotent; pending notifications lost to a silo restart are recorded as acceptable; the per-room board read and write on first enter and last leave is a recorded trade-off; version skew degrades to "no Thoughts menu" on an old WebEx; deletion really clears text; grain log lines carry ids only; the admin commands require the Admin role; the HTML policy allows no images and the client re-adds `noopener noreferrer`.

**CTO1** · **major** · `Items/n3q.Grains/NoteBoardNotificationSender.cs:347` · *introduced* · board → user → room → board wait cycle resolved only by the Orleans timeout.
The board awaits `IUser.SendSystemInstantMessage` (`IUser.cs:106`, neither `[OneWay]` nor `[AlwaysInterleave]`) from a mutation or timer turn with `Interleave = false`.
That user grain may be awaiting `IRoom.HandleUserRequest` (`User.cs:186`) for an enter, ping or leave; `RoomGrain.Maintain` awaits the board's `SubscribePostCountChangedObserver`, `UnsubscribePostCountChangedObserver` (`ThoughtsItemManager.cs:116`, `:128`) on every first-enter and last-leave transition and `SetDestinationIfUnset` on the first enter with a destination.
When the room is the board's own room, the three wait on each other until the default 30 s response timeout breaks one leg: the user's room request fails with an internal error, the room logs "Subscribe to note board failed!", the board logs "SendSystemInstantMessage failed".
`DispatchToDirectChildren` awaits up to 300 such sends in sequence inside one board turn.
At the base no board called the user grain and the room called no board; both legs are new. `IUser.cs:89` records `[OneWay]` as the house remedy for exactly this.
Merge instead: mark `IUser.SendSystemInstantMessage` `[OneWay]` (the sender discards the result and only logs an exception, so nothing is lost), or send without awaiting the user grain.
(anti-patterns.md Unbounded blocking; nine3q CLAUDE.md Deadlock Prevention; lenses/cto.md small risks compound across the install base)

**CTO2** · **major** · `Items/n3q.WebIt/ItemFrameModel.cs:63` · *introduced* (amplifies a pre-existing mechanism) · full note text with the user id is written to the persistent log on every create and edit.
`OnPageHandlerExecutionAsync` puts every handler argument except a truncated `context` into `LogData.Key.Arguments` together with `Context.UserId` and logs at Info; `MicrosoftLoggingCallbackLogger.DoLog` serializes the dictionary into the record and the Azure Table logger provider keeps it.
The new `OnPostCreatePostAsync`, `OnPostCreateCommentAsync`, `OnPostEditPostAsync` and `OnPostEditCommentAsync` (`Thoughts.cshtml.cs:224`, `:232`, `:247`, `:264`) bind the note body as `text`, up to `NotesMaxTextLength` of 10000 characters, so every write stores the complete note HTML with the author and room in log storage for as long as logs are kept.
The base handlers carried ids and action arguments only; the change puts user-written text on the path.
The grains clear the text on delete (`NoteGrain.SoftDelete`); the log copy survives every delete, edit and moderation.
Merge instead: add `text` (and the pre-existing `pageCallArgsJson`) to the redaction switch, logging only the length, and log the returned note id as the identifier of what was written.
(anti-patterns.md PII / secrets in logs & over-collection)

**CTO3** · **major** · `Items/n3q.WebIt/Scripts/Pages/ItemFrame/itemFrame.ts:42` · *introduced* · no-client mode is guaranteed by neither path: no test, no document, automatic activation in every unframed item frame.
`new URLSearchParams(window.location.search).has('noClientMode') || window === window.top` substitutes `NoClientModeWeblinClientApi` (381 lines emulating about 40 client messages with an own-items polling loop, backed by `OnPostGetOwnItems`) for every one of the 19 pages built on `ItemFrameUtils`, whenever the page is opened top-level.
WebIt has no TypeScript test tier; `OnPostGetOwnItems` has no test (E4); neither `docs/`, the two design documents nor `Docs/Features/iframeAPI/iframeApi.md` mention the mode (E3); the only descriptions are the commit message and a one-line `Changes.cs` entry.
The code is not self-explaining where it matters: a public API call can hang and a failed page call logs a second misleading line (E7).
Nobody can vouch for this part long-term.
Merge instead: put it on at least one path, a document naming the activation rule and which `Client.*` requests are emulated, failed or ignored, plus a WebIt test for `OnPostGetOwnItems`; or, if it is a developer convenience, gate it on the explicit `noClientMode` parameter only and drop the `window === window.top` activation so production users opening a frame URL top-level do not run untested emulation.
(lenses/cto.md Principle, two acceptable ways to guarantee functionality; the maintainability finding on the undocumented mode is merged into E3 and referenced here)

**CTO4** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:230` · *introduced* · after a failed state write the activation keeps state storage does not have; `LastNoteId` can be re-issued.
`CreatePost` (lines 678-700) advances `LastNoteId` in memory, has the thread persist the post, then writes its own state; if that write throws, the exception reaches WebIt but the activation keeps the mutated state and serves it, so the user sees an error and then sees the post anyway.
If the activation idles out before another successful write, the persisted board has neither the post nor the counter: the thread and note rows are orphaned until an admin repair, and the next `CreatePost` re-issues the same id and fails with "already has a post" (`NoteThreadGrain.cs:344`).
`NoteThreadGrain.CreateComment` (line 380) has the same shape; `NoteThreadUserReactionsGrain.Save` (lines 188-206) does it right by writing before adopting.
The notes document (line 42) claims ids are never re-issued.
Merge instead: on a failed `WriteStateAsync` catch, log, `DeactivateOnIdle()` and rethrow, or adopt the mutated state only after the write succeeds; apply to the thread's writes too.
(desired-patterns.md Idempotent, restart-safe operations)

**CTO5** · **minor** · `Items/n3q.Grains/ThoughtsItemManager.cs:117` · *introduced* · a failed Thoughts-item update is not retried until the next post-count change.
`SubscribeToNoteBoard` sets `_hasSubscribedToNoteBoard = true` before awaiting `SetPostCount` → `UpdateThoughtsItem` → `inventory.Modify`; if the inventory call fails, the catch logs, `_thoughtsItemPostCount` stays null, and `Maintain` sees the subscription as done and never updates the item again.
`RoomGrain.OnNoteBoardPostCountChanged` (lines 71-74) is `[OneWay]` with no catch, so an inventory exception there is dropped after `_knownPostCount` is already updated.
The room then shows no Thoughts item or a stale count to every visitor until the next post is created or deleted; the Thoughts document promises reconciliation within the enter request.
Merge instead: have `Maintain` call `UpdateThoughtsItem()` in its own `try`/`catch` (already a no-op when counts match), and catch and log inside the one-way callback.
(desired-patterns.md Idempotent, restart-safe operations; Fail closed, degrade gracefully)

**CTO6** · **minor** · `Items/n3q.Grains/NoteBoardGrain.cs:985` · *introduced* · a post whose note row is missing is served to everyone as an empty note with id 0, cached, unlogged.
`GetPosts` substitutes `NoteState.Empty` for a null fetch, stores it in `_noteStateCache` for the rest of the activation and returns it, so a post with id "0", empty author and empty text appears in every user's list; `NoteThreadGrain.GetNote` (line 1631) does the same for comments after a cleared note grain returns the zeroed default.
The comment defers to "the next Repair", which is admin-only, and nothing reports that it is needed; the notes document (line 513) says callers treat the zeroed default as "not found", which these callers do not.
Merge instead: skip the entry, do not cache it, and log a warning with the board and post id.
(anti-patterns.md Swallowed errors / silent failure)

**CTO7** · **minor** · `Items/n3q.Grains/NoteBoardNotificationSender.cs:277` · *introduced* · notification dispatch reads comments as user `"system"`, activating a reactor grain and taking a cache slot per dispatch.
`GetComments` is a per-user read that fills the thread's per-user reactions cache (`NoteThreadGrain.cs:1514-1541`): for `"system"` it activates a `NoteThreadUserReactionsGrain` keyed `{board}:{post}:system`, reads a row that never exists, and takes one of the 30 LRU slots, evicting a real user's cached reactions, on every parent edit or delete with children, on every board.
The literal is not `IUser.InstantMessageSystemUserId` (`"<system>"`), and `"system"` passes `IsValidUserId`, so a real user of that name shares the key.
Merge instead: a thread method that lists direct children without the per-user work, or a nullable `userId` on `GetComments` that skips it.
(criteria/engineering.md §5 separation of concerns; lenses/cto.md small risks compound across the install base)

**CTO8** · **minor** · `Items/n3q.WebIt/Pages/ItemFrame/ThoughtsWire.cs:77` · *introduced* · every reader receives each author's live online status, `isUser` flag and language.
`NotesPageWire.From` passes the grain-side `UserProfile` through unchanged, so the page's initial JSON and every posts, comments and ancestors response carry `isUser`, `userId`, `name`, `imageUrl`, `lang` and `onlineStatus` per author; `UserAccount.GetUserProfile` refreshes `OnlineStatus` from the user's live connections on each call.
Room boards are readable by any authenticated user without presence, and `clientRoomId` lets a reader pick any room, so anyone learns whether each author is online right now.
The SPA declares and reads only `userId`, `name`, `imageUrl` and `lang`; the notes document (line 693) says the layer uses id, name and image.
Merge instead: a `UserProfileWire { UserId, Name, ImageUrl, Lang }` next to `NoteWire`, mapped in `NotesPageWire.From`.
(anti-patterns.md PII / secrets in logs & over-collection, widening who can see data)

Recorded decisions worth knowing, not findings: a deleted or edited parent's previous text is sent to child authors as `originalText` and persisted in their instant-message grains (notes document line 578 names the payload); a board that ever had a post keeps its row, including the page URL, after all posts are gone.

## Coverage

Lenses run: Engineering (nine dimension reviewers: scope and blast radius, simplicity, naming, engineering principles, correctness C#, correctness TypeScript, project conventions C#, project conventions TypeScript, behaviour coverage), Code-review skill (two compliance finders, two bug finders, five validators), Scope, CTO (maintainability, reliability, functionality guarantee, privacy); every reviewer's findings were checked by a separate refuting verifier.
Skipped as planned: Architecture, Security, Adversarial security, Feature regression.
PR description and discussion read; the intent came from the PR description supplemented by the commit messages and the two design documents.
Statistics counted from the diff against the merge-base `2409cf12`; the lines delta excludes test files, blank lines and comment-only lines.
The code-review line scan covered the whole diff minus the 29 vendored bundles and the binary images.
Tests: `NoteGrainTest.cs`, `NoteThreadUserReactionsGrainTest.cs`, `GrainInterfaceRegistryTest.cs`, `NotesHtmlPolicyTest.cs`, `INoteBoardTest.cs`, `RoomGrainTest_Notes.cs`, `NoteBoardGrainTest_FullBoardLifecycle.cs`, `ThoughtsPageTest.cs` and `WebItWebServiceTestBase.cs` were read in full by at least one reviewer; `NoteBoardGrainTest.cs` and `NoteThreadGrainTest.cs` were sampled by section and by grep for the useless-test shapes.
The extension repository was read only to name the consuming side of the config keys; its tests were not run.
The CRLF line-ending candidate from the compliance finder was refuted and left no finding.
The clone for this run used the slug `server-PR-92-thoughts-all-fable-rerun` because the previous run's clone directory was held open by another process and could not be removed.
