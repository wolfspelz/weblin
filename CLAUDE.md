# Claude Code Guidelines

## Commits
- Do NOT add "Co-Authored-By: Claude" to commit messages
- Keep commit messages short: max 2 lines

## Hints
- in the folder, do not ask for these commands: wc, grep, ls, find

## Project Structure
-  there are 2 projects:
	- github-n3qExt: client: a browser extension that show avatars on web pages
	- github-nine3q: server: all the server code for 
		- the base services (config, url-mapping, meeting, chat) 
		- advanced features like item inventory and product web site

## Running
- The extension is compiled by "npm run watch" and executed by reloading the browser extension in Chrome
  - do not run npm build except for compileability and for unit testing
  - tell me to run in (it usually runs anyway)
- The Server is run using the Visual Studio Solutions
	- n3q.sln for the Orleans silo
	- Web.ln for the 2 web projects
	- The Silo solution needs azurite to be running
	- tell me to run/compile/retart the projects in MS Visual Studio
	- ompile and run only for compileability and for unit testing

### General Purpose
- **Social Web Layer:** Overlays avatars and interaction UI on any web page
- **Real-time Presence:** See other users browsing the same page
- **Rooms:** Each web page becomes a virtual room where users can meet

### Communication
- **Chat:** Real-time text chat within rooms
- **Direct Messages:** Private messaging between users
- **Video Conferencing:** Built-in video calls (VidconfWindow)
- **XMPP Integration:** Prosody server for messaging infrastructure

### Platform Support
- **Chrome Extension:** Manifest v3 compatible
- **Firefox Extension:** Separate build target

## Architecture Deep Dive

### Extension Architecture (github-n3qExt)

**Three-layer message passing:**
```
[Web Page DOM] ←→ [Content Script] ←→ [Background Service Worker]
                      ↑                        ↑
                  contentscript/           background/
                  (69 TS files)           (21 TS files)
```

**Key communication classes:**
- `BackgroundMessage.ts` / `ContentMessage.ts` - Message type definitions
- `BackgroundToContentCommunicator.ts` - Background → Content messaging
- `ContentToBackgroundCommunicator.ts` - Content → Background messaging
- `SimpleRpc.ts` - RPC framework for cross-context calls

**Content script key files:**
- `Room.ts` - Manages the virtual room state for current page
- `Avatar.ts` - Renders and animates user avatars
- `Entity.ts` - Base class for all visible entities
- `ChatConsole.ts` - Handles chat input/output
- `*Window.ts` files - UI windows (settings, chat, vidconf, etc.)

**Background script key files:**
- `background.ts` - Service worker entry point
- `Backpack.ts` - User inventory state
- `WebsocketManager.ts` - Server connection handling
- `RoomPresenceManager.ts` - Tracks which rooms user is in
- `ItemProvider.ts` - Fetches and caches item data

**Configuration:**
- `lib/Config.ts` - Central configuration (large file, ~2000+ lines)
- Settings stored via browser.storage API

### Backend Architecture (github-nine3q)

## Services
- `n3q.WebEx` (Base/) - Runtime configuration for the extension
- `n3q.WebIt` (Items/) - Project web site, item management web service, item GUI iframes, WebSocket for real-time item updates
- `n3q.Silo` () - Orleans Silo: distributed computing for stateful grains

**Orleans Virtual Actor Model:**
```
[Web Services]  →  [Orleans Client]  →  [Orleans Silo]  →  [Storage]
(WebIt)                                 (Grains)           (Azure)
```

**Key Grains (Items/n3q.Grains/):**
- `UserGrain` - User profile and state
- `RoomGrain` - Room/page state and participants
- `InventoryGrain` - User's item collection
- `BlobGrain` - Binary data storage
- `PartnerGrain` - User partnerships
- `TextGrain` / `BlogGrain` - Content storage

### Data Flow Patterns

**User enters a page:**
1. Content script detects page load
2. Sends room URL to background
3. Background connects to XMPP for presence
4. Background notifies content of other participants
5. Content script renders avatars

**Item interaction:**
1. User action in content script
2. Message to background via ContentToBackgroundCommunicator
3. Background calls WebIt via WebSocket/HTTP
4. WebIt invokes Orleans grain
5. Grain updates state, persists to storage
6. Response flows back through chain & item updates via websocket

### Key Entry Points for Modifications

| Task | Look Here |
|------|-----------|
| Add UI window | `contentscript/*Window.ts` |
| New chat command | `ChatConsole.ts` |
| Avatar behavior | `Avatar.ts`, `Entity.ts` |
| Background API call | `background/` + relevant manager |
| New grain | `Items/n3q.Grains/` |
| Web API endpoint | `Base/n3q.WebEx/Controllers/` or `Items/n3q.WebIt/` |
| Configuration | `lib/Config.ts` (extension), `appsettings.json` (server) |
| Item properties | `lib/ItemProperties.ts` |

## Documentation

- Located in `docs/` at project root
- All documentation must be written in **English**
- **Before implementing or extending a feature**, check if documentation exists for it in that folder
- If no documentation exists for the feature to be implemented, **do not start coding** - instead, point this out and collaborate with the user to create the documentation first (analyze the existing system together, then document it, then implement)
- This ensures a shared understanding of the current system before making changes

### Available Documentation
- [Upload Avatar Process](docs/upload-avatar-process.md) - End-to-end flow from empty Upload Avatar item to rendered animation

