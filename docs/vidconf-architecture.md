# Video Conference (Vidconf) Architecture

Documentation of how weblin.io integrates Jitsi Meet for in-room video conferencing. Covers the two entry points (public room vidconf and private 1:1 vidconf), the iframe URL pipeline, where the Jitsi server is configured, and the alternate `JitsiMeetExternalAPI` pages that currently exist but are not wired in.

## Overview

weblin.io embeds Jitsi Meet as an iframe inside a weblin window. No Jitsi login is required: users join anonymous rooms whose names are derived from the weblin room JID (public) or from a shared secret exchanged via XMPP (private).

There are two code paths for opening a vidconf:

| Path | Trigger | Window class | Room name |
|---|---|---|---|
| **Public** | User opens the vidconf button in a room | `VidconfWindow` | `weblin<URI-encoded room JID>` |
| **Private 1:1** | User clicks "Videoconference" on another participant's Person popup | `PrivateVidconfWindow` (extends `VidconfWindow`) | `weblin-private-<sortedUserIds>-<sharedSecret>` |

Both paths build the iframe URL from the same template: `config.room.vidconfUrl`.

## The iframe URL template

The template is a single string shipped to the extension via the `ClientConfig` API. Placeholders `{room}` and `{name}` are substituted client-side.

**Current value (served by WebEx server):**
```
https://jitsi.weblin.io/weblin{room}?t=<shared-secret>#userInfo.displayName=%22{name}%22
```

The `?t=<secret>` query param is required by the nginx access gate (see [Jitsi server configuration](#jitsi-server-configuration) below). It must come **before** the `#` hash fragment, otherwise it is part of the fragment and not seen by the server.

All Jitsi config overrides (resolution, prejoin off, invite off, etc.) live in the self-hosted server's `config.js` + `interface_config.js`.

The hash part (`#...`) is how Jitsi Meet accepts runtime config overrides when loaded in an iframe without the `JitsiMeetExternalAPI` wrapper. See https://jitsi.github.io/handbook/docs/user-guide/user-guide-advanced/. The display-name substitution via hash is load-bearing: without it, Jitsi shows the prejoin screen regardless of `prejoinConfig.enabled`.

**Deployed-client fallback** in `Config.ts:225` still points at the old `meet.jit.si` URL with the buggy `config.resolution=true` hash param. It is only used when the WebEx server is unreachable, and cannot be updated without a client release.

## Configuration chain

The URL template originates on the server and flows through three layers to the extension:

```
WebExConfigDefinition.RoomVidconfUrl       <-- schema + compiled-in default
        |
        v
WebExConfig.RoomVidconfUrl                 <-- runtime value (can be overridden via ConfigSharp)
        |
        v
ConfigController.GetConfig(...)            <-- HTTP endpoint that serves ClientConfig
    config.room.vidconfUrl = ...
        |
        v (HTTP)
Extension background: stores config in browser.storage
        |
        v
Config.get('room.vidconfUrl', fallback)    <-- called from contentscript
```

**File map:**
- `github-nine3q/App/n3q.AppInterfaces/WebExConfigDefinition.cs:24` — compile-time default
- `github-nine3q/Base/n3q.WebEx/WebExConfig.cs:54` — runtime setter
- `github-nine3q/Base/n3q.WebEx/ClientConfig.cs:69` — `vidconfUrl` field shipped to client
- `github-nine3q/Base/n3q.WebEx/Controllers/ConfigController.cs:70` — assembles the client config
- `github-n3qExt/ChromeExt/src/lib/Config.ts:225` — extension fallback (used if server is unreachable)

`ConfigController.cs:68` also lists `ignoredDomainSuffixes = { "video.weblin.io", "vulcan.weblin.com", "meet.jit.si" }` so the extension does not try to overlay avatars when the user navigates into the Jitsi iframe.

## Public room vidconf

**Entry:** `ContentApp.showVidconfWindow()` → `Room.showVideoConference()` at `Room.ts:547`.

1. User clicks the vidconf button in the bottom menu
2. `Room.showVideoConference(aboveElem, displayName)`:
   - Fetches `config.room.vidconfUrl`
   - Substitutes `{room}` with `encodeURIComponent(this.jid)` (the XMPP room JID for the current page)
   - Substitutes `{name}` with `encodeURIComponent(displayName)`
   - Creates a new `VidconfWindow` with that URL and shows it undocked
3. `VidconfWindow.makeContent()` creates an `<iframe>` with `allow="camera; microphone; fullscreen; display-capture"`
4. The iframe loads Jitsi Meet, which joins room `weblin<encoded-jid>` anonymously

State persistence: `ContentApp.vidconfIsOpen` is saved in `Memory.setLocal` keyed by room JID so the window reopens on page revisit.

## Private 1:1 vidconf

**Entry:** `ContentInstantMessageManager.initiatePrivateVidconf()` at `ContentInstantMessageManager.ts:52`.

Triggered by the "Videoconference" button in the other user's Person backpack iframe item (`github-nine3q/Items/n3q.WebIt/Pages/ItemFrame/Person.cshtml:84`). The button sends `Client.OpenPrivateVidconfRequest` via the iframe API; `BackpackItemInfo.handleOpenPrivateVidconfRequest()` forwards to `instantMessageManager.initiatePrivateVidconf()`.

**Shared-secret room name** (keeps the 1:1 private from being guessable):
1. Initiator generates a `vidconfId` secret (stored in `privateVidconfSecrets` map + `Memory.setLocal`)
2. Initiator sends XMPP chat message type `vidconfInvite` with `{vidconfId}` payload
3. Recipient accepts → both sides independently build:
   ```
   vidconfRoomId = `-private-${sortedUserIds.join('-')}-${vidconfId}`
   ```
4. Each side opens a `PrivateVidconfWindow` with the same URL template substituted with this room ID

The URL template is the same as the public path, so hash-fragment Jitsi config overrides apply identically.

## Window geometry

Defined in `VidconfWindow.ts:20-25`:

| Property | Value |
|---|---|
| `defaultWidth` | 600 |
| `defaultHeight` | 400 |
| `minWidth` | 180 |
| `minHeight` | 180 |
| `defaultBottom` | 200 |
| `defaultLeft` | 50 |

The undocked-popup dimensions come from `Config`:

| Config key | Fallback (`Config.ts:227-228`) |
|---|---|
| `room.vidconfWidth` | 630 |
| `room.vidconfHeight` | 530 |
| `room.vidconfBottom` | 200 |
| `room.vidconfPopout` | true |

These small dimensions are why the video feed should be capped at 360p — bigger resolutions waste bandwidth and CPU without visible benefit.

## Dormant alternate paths

Two Razor pages embed Jitsi via `JitsiMeetExternalAPI` (the official JS wrapper, which accepts `configOverwrite` as a proper JS object rather than URL hash strings). **They are not currently wired into the extension flow** but exist as reference implementations:

- `github-nine3q/Base/n3q.WebEx/Pages/Vidconf.cshtml` — originally used via `https://video.weblin.io/Vidconf?room=...` template (now commented out in Config.ts)
- `github-nine3q/Items/n3q.WebIt/Pages/Video.cshtml` — landing page advertised on the public site

Both set correct 360p constraints and show the approach that would be needed if weblin moves to JWT-authenticated Jitsi (since JWT tokens cannot be safely passed via URL hash).

A third reference is `Basic.cs:VideoIframeTemplate.MakeVideoFrameUrl()` which builds the URL template for video-iframe item templates (e.g. `PublicViewing` in `Standard.cs`). Properties defined in a template's `Properties()` method are **resolved dynamically on every `Item.Get()` read** — not snapshotted at item creation. So after `MakeVideoFrameUrl()` is changed and the Silo redeployed, all existing items created from derived templates immediately serve the new URL. No data migration needed.

## Jitsi server configuration

Self-hosted at `jitsi.weblin.io` (Ubuntu 24.04). Server-wide config overrides live in `/etc/jitsi/meet/jitsi.weblin.io-config.js` (appended block at end of file) and `/usr/share/jitsi-meet/interface_config.js`:

| Setting | Value | File |
|---|---|---|
| `resolution` | `360` | config.js |
| `constraints.video.height` | `{ ideal: 360, max: 480, min: 240 }` | config.js |
| `prejoinPageEnabled` / `prejoinConfig.enabled` | `false` | config.js |
| `disableInviteFunctions` | `true` | config.js |
| `enableNoisyMicDetection` | `false` | config.js |
| `doNotStoreRoom` | `true` | config.js |
| `enableInsecureRoomNameWarning` | `false` | config.js |
| `disableH264` | `true` | config.js |
| `SHOW_CHROME_EXTENSION_BANNER` | `false` | interface_config.js |

Let's Encrypt cert via acme.sh; renewal via root cron at 01:15 daily. Original distro configs backed up as `*.bak` next to the edited files.

### Access gate (loose)

`/etc/nginx/sites-available/jitsi.weblin.io.conf` contains a shared-secret gate at the top of the 443 server block. A request is allowed if **any** of these is true:

- `?t=<shared-secret>` query string (initial page load from weblin client)
- `Referer` header matches `^https://jitsi\.weblin\.io/` (subresources loaded inside the Jitsi page)
- `Origin: https://jitsi.weblin.io` (WebSocket upgrades — which don't send Referer)

Otherwise the request is redirected internally to `location = /_weblin_forbidden` which returns 403. (Using `if ($var = "000") { return 403; }` at server scope silently failed on this nginx version — the rewrite-to-named-location workaround is reliable.)

Implementation: three `map` directives at http-context level compute `$weblin_arg_ok`, `$weblin_ref_ok`, `$weblin_org_ok`; `$weblin_gate` concatenates them; `if ($weblin_gate = "000")` triggers the rewrite.

Purpose is loose protection against scanners finding open Jitsi servers — the secret is visible in client DevTools Network tab. To rotate, change the `map $arg_t` value in nginx and `RoomVidconfUrl` in `WebExConfig.cs` + `WebExConfigDefinition.cs`, then reload both. In-progress calls survive rotation via the Referer/Origin exceptions.

## Known issues

1. **`config.resolution=true` bug** in all three hard-coded copies (`Config.ts:225`, `WebExConfig.cs:54`, `WebExConfigDefinition.cs:24`, `Basic.cs:44`). Should be `config.resolution=360`. The `Vidconf.cshtml` path has the correct value (as a number, plus `constraints.video.height` ideal/max/min).
4. **Prejoin field renamed in Jitsi**: `prejoinPageEnabled` was replaced by `prejoinConfig.enabled` in recent Jitsi versions. The URL template still uses the legacy field, which is silently ignored. Even with prejoin disabled, Jitsi shows the prejoin screen when no `userInfo.displayName` is supplied — so the display-name substitution in the URL is load-bearing for skipping it.
2. Three hard-coded copies of the URL template drift apart easily — a new Jitsi server URL must be changed in at least `WebExConfig.cs`, `WebExConfigDefinition.cs`, `Config.ts`, and `Basic.cs`, plus the `ignoredDomainSuffixes` list.
3. Private 1:1 secret is logged to `Memory.setLocal` unencrypted — acceptable since it only protects against casual room-ID guessing, not a determined attacker.

## Key entry points for modifications

| Task | File |
|---|---|
| Change Jitsi server URL | `WebExConfig.cs`, `WebExConfigDefinition.cs`, `Config.ts:225`, `Basic.cs:44`, `ignoredDomainSuffixes` in `ConfigController.cs:68` |
| Change default window size | `VidconfWindow.ts:20-25` |
| Change undocked popup size | `Config.ts:227-228` |
| Change Jitsi runtime config (resolution, features) | Hash-fragment part of the URL template (all four locations above) |
| Switch to `JitsiMeetExternalAPI` | Use `Vidconf.cshtml` as template; change `vidconfUrl` to point at it instead of directly at Jitsi |
| Change private-room naming scheme | `ContentInstantMessageManager.ts:187` |
