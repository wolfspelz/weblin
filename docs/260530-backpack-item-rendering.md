# Backpack Item Rendering (Browser-Extension Client)

The Backpack window of the browser-extension client (`github-n3qExt`) is a renderer of item state owned by the background worker:
one free-positioned tile per item,
re-rendered from the full property set on every update,
with visual state composed from independent CSS classes,
"quasi-tab" filters that fade instead of swapping panes,
subset windows (e.g. Contacts) locked to one filter,
and every server-side property change arriving over the WebSocket through the same pipeline.

Tiles are painted from the GUI-wide CSS design-token layer, and theme items are ordinary backpack items
— §8 covers only the backpack side of both;
the styling and theming machinery is in [GUI Styling and Themes](260530-gui-styling-and-themes.md).

All paths are relative to `github-n3qExt/ChromeExt/src/`.

---

## 1. The Backpack window and its pane

`BackpackWindow` (`contentscript/BackpackWindow.ts:14`) extends `FullWindow` (the draggable/resizable window base).
Settings, in `initWindowSettings()` (`BackpackWindow.ts:39`):

- `windowSettingsId = 'Backpack'`, geometry persisted, CSS class `backpackwindow`
- title "Local Stuff" (i18n `BackpackWindow.Inventory`), 600×400 default, action bar enabled

The constructor (`BackpackWindow.ts:27`) wires up three collaborators and the three callbacks that connect them:

```
this.filters       = new BackpackWindowItemFilters(app, singleFilterId, hideFilterTags, windowSettingsId,
                                                    guiVisibilityHandler, itemVisibilityHandler)
this.selectedItems = new BackpackSelectedItems(app, this)
this.backpackItems : Map<itemId, BackpackItem>          // one tile per item
```

- `guiVisibilityHandler(visible)` → `setActionBarVisibleState()` — shows/hides the filter tab bar.
- `itemVisibilityHandler(itemId, visibility)` → `itemFilterVisibilityHandler()` — applies per-tile visibility.

The content pane is a single absolutely-positioned canvas, `<div class="backpack-pane">`,
appended to the window content (`makeContent()`),
carrying a `PointerEventDispatcher` for click/drag and rubber-band selection.
Item tiles are children of this pane and are **free-positioned** (no grid) — see §2.

---

## 2. Tile creation and update: `BackpackItem.setProperties()`

A tile is a `BackpackItem` (`contentscript/BackpackItem.ts:11`).
It is a plain class (not a window) that owns one DOM subtree:

```html
<div class="backpack-item" data-id="{itemId}">
  <div class="backpack-item-icon">
    <div class="backpack-item-overlays"></div>   <!-- §5: overlay badges -->
    <!-- <img> icon appended asynchronously -->
  </div>
  <div class="backpack-item-label">…</div>
  <!-- optional <div class="backpack-item-online-status"> -->
</div>
```

Tiles are created lazily by the window in `setItemVisibility()` (`BackpackWindow.ts:305`)
the first time an item needs to be shown, and inserted into `backpackItems`.

### The single update entry point

Both creation and every later change funnel through one method,
`BackpackItem.setProperties(properties)` (`BackpackItem.ts:213`).
It replaces the stored property set and **re-applies every visual aspect unconditionally**
— it does not diff which property changed:

```
setProperties(properties) {
    this.properties = properties
    this.applyText()          // Pid.Label / Pid.Description -> label text + title
    this.applySize()          // Pid.Width / Pid.Height (default 50px) -> icon + label box
    this.applyPosition()      // Pid.InventoryX / Pid.InventoryY -> absolute left/top
    this.applyImage()         // Pid.ImageUrl -> scaled/clipped <img>, swapped in async
    this.applyOnlineStatus()  // contact online indicator
    this.itemOverlaysState = app.getItemOverlays().updateItemOverlays(properties, this.itemOverlaysState)
    // rezzed state, see §3
}
```

Positioning is free-form and **centre-based**:
`applyPosition()` (`BackpackItem.ts:167`) reads `Pid.InventoryX/Y` as the tile centre
and offsets to the top-left corner (`left = x - width/2`, `top = y - height/2`).
The pane origin is its top-left; values increase toward the bottom-right.

One code path handles a new, moved, renamed or re-imaged item:
the caller hands over the latest full property set (§6 says where it comes from).

---

## 3. Visual state: independent CSS classes on `.backpack-item`

A tile's appearance is the sum of **independent CSS classes** on the root `.backpack-item` element.
Each class reflects one piece of state and is toggled by its own subsystem without consulting the others,
so a tile can be rezzed *and* selected *and* filter-hidden at once.
Classes are flipped via `BackpackItem.setCssClass(cls, on)` (`BackpackItem.ts:183`),
a wrapper over `DomUtils.setElemClassPresent`.

| Class | Meaning | Set by | Driven from |
|-------|---------|--------|-------------|
| `rezzed` | Item is currently placed in a room | `setProperties()` (`BackpackItem.ts:224`) | `Pid.IsRezzed` |
| `selected` | User has selected the tile | `BackpackSelectedItems.setItemSelectedStyle()` | local UI selection |
| `filter-hide` | Tile does not match the active filter | `setItemVisibility()` (`BackpackWindow.ts:319`) | active filter (§4) |
| `hidden` | Hidden while being dragged | `BackpackSelectionDraggedItem.setDraggedStyle()` | drag in progress |
| `selection-rect-add` / `selection-rect-remove` | Rubber-band preview | `BackpackUserSelectionRect` | live marquee (§3.2) |

The styling lives in `contentscript.css`
(e.g. `.backpack-item.filter-hide` grays/blurs the icon, hides label and overlays, and sets `pointer-events: none`).

### 3.1 Selection

`BackpackSelectedItems` (`contentscript/BackpackSelectedItems.ts`) holds the selection as a `Map<itemId, BackpackItem>`
and exposes `itemSelect`, `itemDeselect`, `itemToggleSelect`, `itemSelectExclusively`, `itemDeselectAll`.
Each change toggles the `selected` class on the affected tile.
When a tile becomes filter-hidden it is also force-deselected (`BackpackWindow.ts:316`),
so the selection never contains invisible items.

### 3.2 Rubber-band selection

`BackpackUserSelectionRect` (`contentscript/BackpackUserSelectionRect.ts`) draws a marquee `<div class="user-selection-rect">` on the drag layer.
As the pointer moves it tests each tile's bounding box against the rectangle
and previews the result via the `selection-rect-add` / `selection-rect-remove` classes.
Modifier keys pick the mode — plain = replace, Shift = add, Ctrl = remove —
and on drag-end the previewed set is committed into `BackpackSelectedItems`.

### 3.3 Item info popup

Left-clicking a tile (`BackpackWindow.onItemLeftClick`) selects it
and toggles a `BackpackItemInfo` (`contentscript/BackpackItemInfo.ts`, a `PopupWindow`) showing header properties,
an optional item iframe UI, action buttons (rez/derez, go-to, delete, …) and, with Alt, a debug dump of all properties.

---

## 4. "Quasi-tab" filter groups

Filtering is owned by `BackpackWindowItemFilters` (`contentscript/BackpackWindowItemFilters.ts`)
plus its GUI helper `BackpackWindowItemFiltersGui`.
The tabs are "quasi" because they are **not real tabs that swap panes**
— every tile stays in the one pane, and the active filter only changes which tiles are faded out.

### 4.1 `ItemFilter`: config-driven `BasicItemFilter` with a rule grammar

A filter implements `ItemFilters.ItemFilter` (`lib/ItemFilters.ts:11`):

```
interface ItemFilter {
    getId(); hasTag(tag); getIconUrl(); getLabelText(lang); getHelpText(lang);
    isMatchingItem(item: ItemProperties): boolean
}
```

The concrete `BasicItemFilter` wraps an id, tag set, icon, multilingual labels/help, and a `matchFun(item) => boolean`.
Filters are **data-driven from configuration**, not code:
`parseFilters()` reads `Config.get('backpack.filters')`
and `parseItemFilters()` turns each JSON definition into a `BasicItemFilter`.
The rule grammar (`parseRule`, `lib/ItemFilters.ts`) supports `any`, `propertyIsTrue`, `propertyIsNotEmpty`,
`propertyStringValueIsOneOf`, and the combinators `not` / `or` — all evaluated against an item's `Pid` properties.
If no filters are configured, a fallback `defaultAll` filter (label "All", matches everything) is created.

### 4.2 The tab bar (GUI)

`BackpackWindowItemFiltersGui` builds `<div class="backpack-filters">` containing, per filter,
a hidden radio `<input>` (`name="n3q-backpack-{window}-filter"`) plus a clickable label button.
Selecting a button fires `onUserSelectFilter(filterId)`.
Visual selection is the radio's `checked` state (`showFilterSelected`).
Buttons for filters that currently match **zero** items are hidden by toggling the `removed` class (`updateFilters`),
so empty tabs disappear.
The whole bar is shown only when more than one filter is non-empty
— `guiVisibilityHandler(nonEmptyFilters.size > 1)` (`BackpackWindowItemFilters.ts:103`).

### 4.3 Filter application: `filter-hide` class toggle, never a DOM rebuild

Tiles are destroyed only when the item leaves the backpack, never because they fail the active tab.
The manager tracks, per filter, a `matchingItemIds` set,
and globally two sets: `fullVisibilityItemIds` and `fadedVisibilityItemIds`.
There are three per-item visibility outcomes (`ItemVisibility = 'none' | 'faded' | 'full'`, `BackpackWindowItemFilters.ts:21`)
that the manager pushes to the window via the `itemVisibilityHandler` callback:

- `full` — matches the active filter → tile visible & interactive (no extra class).
- `faded` — does not match the active filter → tile keeps its DOM node
  but gets the `filter-hide` class (grayed, blurred, non-interactive).
- `none` — item is gone from the backpack (deleted, or `getIsVisibleInBackpack(item)` is false)
  → the window **destroys** the tile and removes it from `backpackItems`.

`selectFilter()` (`BackpackWindowItemFilters.ts:169`) recomputes the full set as the chosen filter's `matchingItemIds`,
then emits `faded` for tiles leaving the full set and `full` for tiles entering it.
The window's `setItemVisibility()` (`BackpackWindow.ts:297`) turns those into the `filter-hide` class toggle
(or tile destruction for `none`).
The chosen filter is persisted per window in `Memory` (`window.{name}.currentItemFilterId`)
and restored on the next update.

---

## 5. Subset windows (Contacts) — one forced filter

A subset window is a `BackpackWindow` subclass that sets `singleFilterId`;
it reuses the entire Backpack machinery (tiles, update, visual state, selection) locked to that one filter.
`PersonsWindow` (`contentscript/PersonsWindow.ts`) is the Contacts list:

```
class PersonsWindow extends BackpackWindow {
    initWindowSettings() {
        super.initWindowSettings()
        this.windowSettingsId = 'Contacts'
        this.windowCssClasses.push('personswindow')
        this.titleText = 'Contacts'
        this.singleFilterId = Config.get('personsWindow.itemFilterId', 'persons')   // <- the forced filter
    }
}
```

Because `singleFilterId` is set before the base constructor builds `this.filters`,
the filter manager enters **single-filter mode** (`BackpackWindowItemFilters.ts:53`):

- `parseFilters()` keeps only the filter whose id equals `singleFilterId`; all others are discarded.
  `hideFilterTags` is ignored in this mode.
- `getFilterIdToSelect()` always returns `singleFilterId`,
  so the active filter is locked and cannot be changed by the user.
- With a single non-empty filter, `guiVisibilityHandler(size > 1)` is `false`, so the quasi-tab bar stays hidden.
- The per-window filter memory is not written in single-filter mode.

The `persons` filter matches items whose person property is set.

---

## 6. Content script ↔ background worker: who owns the items

The background worker owns the item state; the content script only renders it.
Two channels connect them:

- **Commands (content → background)**: request/response RPCs via the static methods on `lib/BackgroundMessage.ts`,
  carried by `ContentToBackgroundCommunicator`.
  Each method builds `{ type: BackgroundMessage.<method>.name, … }`; the request-type *string is the method name*.
  `BackgroundApp` dispatches them in a `switch` (`case BackgroundMessage.<method>.name`).
- **Updates (background → content)**: one-way push of `ContentMessage.type_onBackpackUpdate` to every tab (§7).

### 6.1 Division of labour

| Layer | Holds | Proof |
|-------|-------|-------|
| Background `Backpack` (authoritative) | `private readonly items: Map<itemId, ItemProperties>` (`background/Backpack.ts:29`) — the single source of truth, plus the `rooms` map and the `providers` | `getItems()` (`Backpack.ts:63`) returns it read-only |
| Content `OwnItemRepository` (mirror) | `private readonly ownItems: Map<itemId, ItemProperties>` (`contentscript/OwnItemRepository.ts:16`) | `onBackpackUpdate()` (`OwnItemRepository.ts:55`) is the **only** writer of this map |
| Content `BackpackWindow` (view) | `backpackItems: Map<itemId, BackpackItem>` tiles, built from the mirror | `setItemVisibility()` reads `app.ownItems.getItemById(itemId)` (`BackpackWindow.ts:307`) |

All authoritative mutation happens in the background `Backpack`, which delegates to a provider (§7.1).

### 6.2 Initial load: the first, full-set `onBackpackUpdate`

Opening the window and a later item change funnel through the same `onBackpackUpdate` entry point;
the mirror is populated on demand:

1. The content side requests the full set with `BackgroundMessage.requestBackpackState()`
   (`lib/BackgroundMessage.ts:390`, type string `requestBackpackState`).
2. `BackgroundApp` handles it (`handle_requestBackpackState(tabId)`) and calls `Backpack.sendAllOwnItemsToTab(tabId)`,
   which packs **all** `items` into a `BackpackUpdateData([], items)`
   and sends `ContentMessage.type_onBackpackUpdate` to that one tab.
3. The tab receives it like any other update (§7)
   → `OwnItemRepository.onBackpackUpdate` fills `ownItems`
   → `BackpackWindow.onBackpackUpdate` renders the tiles.

### 6.3 Command path (content → background → provider → echo)

User actions on tiles call a `ContentApp` method, which sends a `BackgroundMessage` RPC.
The background mutates the authoritative state via the item provider
and then **pushes the result back** through the update pipeline (§7).
The content side does not patch its own tile from the RPC return value (with one exception, §6.4)
— it waits for the authoritative echo.

| Action | Content entry point | RPC (`BackgroundMessage`) / type string | `BackgroundApp` handler | `Backpack` method → provider |
|--------|--------------------|------------------------------------------|--------------------------|------------------------------|
| Move tile (drag) | `ContentApp.setItemBackpackPosition()` writing `Pid.InventoryX/Y` | `modifyBackpackItemProperties` | `handle_modifyBackpackItemProperties` | `modifyItemProperties()` → `provider.modifyItemProperties()` |
| Rez | `ContentApp.rezItem()` | `rezBackpackItem` | `handle_rezBackpackItem` | `rezItem()` → provider `itemAction('Rezable.Rez', …)` |
| Derez | `ContentApp.derezItem()` | `derezBackpackItem` | `handle_derezBackpackItem` | `derezItem()` → provider `itemAction('Rezable.Derez', …)` |
| Delete | `ContentApp.deleteItem()` | `deleteBackpackItem` | `handle_deleteBackpackItem` | `deleteItem()` → `provider.deleteItem()` |
| Item action (info popup, iframe, badges) | `BackpackItemInfo` / `IframeApi` | `executeBackpackItemAction` | `handle_executeBackpackItemAction` | `executeItemAction()` → `provider.itemAction()` |

Rez/derez/delete are triggered from the tile click handlers and the `BackpackItemInfo` buttons
(`BackpackWindow.onItemLeftClick`, `BackpackItemInfo.ts`).
**Drag-move persists position** by writing `Pid.InventoryX/InventoryY` through `modifyBackpackItemProperties`
— the same per-property mutation path as any other property edit,
so a moved tile comes back through the normal update pipeline and `applyPosition()` re-renders it (§2).

### 6.4 Echo-driven, with one optimistic exception

The model is **command → authoritative echo**:
after a mutation RPC,
the visible change arrives only when the background broadcasts `type_onBackpackUpdate` (`Backpack.sendUpdateToAllTabs`)
and the tile is re-rendered via `setProperties`.
Rez, derez, delete and item actions are purely echo-driven — no local pre-update.

The single exception is **drag-move** (`ContentApp.setItemBackpackPosition`):
for responsiveness it applies an *optimistic* local `onBackpackUpdate` with the new `InventoryX/Y` immediately,
fires the `modifyBackpackItemProperties` RPC,
and **rolls back** to the old coordinates if the RPC rejects.
The authoritative echo then confirms (or corrects) the position.

---

## 7. Server-side property changes: `ItemsNotification` → `type_onBackpackUpdate`, always

Every inventory change made on the server reaches the tiles through one pipeline
— server → WebSocket → background → all content scripts → window → tile —
and the client re-renders from the **full updated property set**, not from a diff.
The same pipeline carries the echo of every content-issued command (§6.3) and the initial full-set load (§6.2).

| Hop | Where | What happens |
|-----|-------|--------------|
| Server → WS | `ItemsNotification` (`lib/WebsocketMessage.ts`) | `{InventoryId, ItemsUpdatedOrCreated[], ItemsDeleted[]}`; each updated item is its **complete** property set |
| WS → background | `WebsocketManager.handleItemsNotification()` | ignores notifications for other inventories, then calls `Backpack.onItemUpdateFromProvider(deleted, updatedOrCreated)` |
| background state | `Backpack.onItemUpdateFromProvider()` (`background/Backpack.ts:92`) | stores full items, computes really-changed/deleted, then `sendUpdateToAllTabs()` |
| background → content | `ContentMessage.type_onBackpackUpdate` + `BackpackUpdateData{itemsHide, itemsShowOrSet}` | broadcast to every tab |
| content reception | `ContentApp` switch → `onBackpackUpdate()` | forwards to `OwnItemRepository.onBackpackUpdate()` |
| content state | `OwnItemRepository` (`contentscript/OwnItemRepository.ts`) | stores full items in `ownItems`, fires per-item + backpack listeners |
| window | `BackpackWindow.onBackpackUpdate()` (`BackpackWindow.ts:324`) | for each shown item calls `BackpackItem.setProperties(item)`; then `filters.onBackpackUpdate(...)` recomputes matches and re-applies visibility |
| tile | `BackpackItem.setProperties()` | re-applies text, size, position, image, overlays, rezzed (§2) |

Two points make this hold regardless of which property changed:

1. **No client-side property diffing.**
   `ItemsUpdatedOrCreated` carries each item's whole property set;
   `BackpackItem.setProperties()` re-runs all `apply*` methods every time.
   Rename, move, re-image, rez/derez and theme toggle all run the same path.
2. **Subscriptions only narrow what the server bothers to send.**
   `ItemUpdateSubscription` (`lib/ItemUpdateSubscription.ts`) lets a client component
   declare which items (`ownItems`/`otherItems`, `matchProperties`)
   and which properties (`pidsToSend`) it wants.
   The server filters *before* emitting `ItemsNotification`; the client never filters incoming changes.
   The player's own backpack subscribes to its own items, so all of its changes flow through.

`itemsHide` (and items whose `getIsVisibleInBackpack` is false) flow through the filter manager as `'none'`,
destroying those tiles;
everything else is shown or updated.

### 7.1 The provider layer

`Backpack` does not talk to the server directly;
it delegates per item to an `IItemProvider` (`background/ItemProvider.ts`),
in practice `HostedInventoryItemProvider` (`background/HostedInventoryItemProvider.ts`).
`Backpack` keeps a `providers` map and routes each mutation through `getProvider(itemId)`:

- Command methods (`modifyItemProperties`, `rezItem`, `derezItem`, `deleteItem`, `executeItemAction`)
  forward to the provider,
  which performs the server call (an `itemAction` over its hosted-inventory connection).
- The server's authoritative result re-enters `Backpack` via `Backpack.onItemUpdateFromProvider(deletedIds, createdOrUpdated)`,
  which updates the `items` map and calls `sendUpdateToAllTabs()`.

Unsolicited server pushes (received over the WebSocket by `WebsocketManager`) and the results of client-issued commands
converge on the one `onItemUpdateFromProvider → sendUpdateToAllTabs → type_onBackpackUpdate` path.
The content side cannot tell whether a tile changed
because of its own command, another tab's command, or a server-side event.

---

## 8. Styling and themes

> Shadow-DOM isolation, the CSS design-token (custom-property) layer, the window/widget framework and the theme system are GUI-wide:
> [GUI Styling and Themes](260530-gui-styling-and-themes.md).
> Below: the backpack's hooks into them.

### 8.1 Tiles are painted from shared design tokens

Backpack tiles use **no hard-coded colours, spacings or shadows**.
Every visual constant is read from a CSS custom property (`var(--…)`)
defined in the shared token block on `#n3qD` (`contentscript.css`),
so a tile stays consistent with panes, buttons and other windows and re-skins when a theme overrides those tokens.
Tokens consumed by the backpack rules:

| Backpack rule | Token(s) used |
|---------------|---------------|
| `.backpack-item` box shadow / fill | `--hard-shadow-color`, `--base-background-color` |
| `.backpack-item-label` background | `--tiny-pane-background` |
| `.backpack-item.selected` / marquee-add | `--selected-color` (derived shadow + fill) |
| `.backpack-item.dragging` | `--fadeout-opacity` |
| `.backpack-item-online-status[...]` | `--online-led-{unknown,offline,online}-{color,background,filter}` |
| `.backpack-pane.drop-target(.highlight)` | `--drop-target-background-color`, `--drop-target-highlight-background-color` |

See [GUI Styling and Themes §3](260530-gui-styling-and-themes.md) for the token taxonomy
and how `--brightness-factor` drives light/dark behaviour.

### 8.2 Theme items are backpack items

A **theme item** is an ordinary inventory item carrying `Pid.ThemeAspect` (marks it a theme),
`Pid.ThemeCss` (the CSS text, in practice token overrides) and the activation flag `Pid.ActivatableIsActive`.
It is distinct from the per-tile overlay **badges** of §2.

Because a theme item is a backpack item,
**enabling/disabling or editing it propagates through the same update pipeline as any other item-property change** (§6–§7).
`BackgroundThemeManager.onBackpackUpdate()` listens to backpack updates,
extracts each theme item via `ItemProperties.getThemeData()`,
and feeds the `item`-source themes into the theme system;
`ContentThemeManager` then rewrites a single `<style data-type="theme">` element in the shadow DOM.
The end-to-end theme mechanism (sources, aggregation, shadow-DOM injection, iframe forwarding)
is documented in [GUI Styling and Themes §6](260530-gui-styling-and-themes.md).

---

## 9. Key files

| Concern | File |
|---------|------|
| Window + pane, visibility callbacks | `contentscript/BackpackWindow.ts` |
| Tile DOM, `setProperties`, CSS-class state | `contentscript/BackpackItem.ts` |
| Filters, quasi-tabs, single-filter mode | `contentscript/BackpackWindowItemFilters.ts` |
| Filter definitions / rule grammar | `lib/ItemFilters.ts` |
| Contacts subset window | `contentscript/PersonsWindow.ts` |
| Selection / rubber-band | `contentscript/BackpackSelectedItems.ts`, `contentscript/BackpackUserSelectionRect.ts` |
| Item info popup | `contentscript/BackpackItemInfo.ts` |
| Overlay badges | `contentscript/ItemOverlays.ts` |
| Content → background command RPCs | `lib/BackgroundMessage.ts` |
| Background request dispatch | `background/BackgroundApp.ts` |
| Content-side mutation entry points (rez/derez/delete/move) | `contentscript/ContentApp.ts` |
| Authoritative item store + provider routing | `background/Backpack.ts` |
| Server call provider | `background/ItemProvider.ts`, `background/HostedInventoryItemProvider.ts` |
| WS update intake / fan-out | `background/WebsocketManager.ts`, `background/Backpack.ts` |
| Content-side item store (read mirror) | `contentscript/OwnItemRepository.ts` |
| Update subscriptions | `lib/ItemUpdateSubscription.ts` |
| Property constants | `lib/ItemProperties.ts` |
| GUI-wide styling, design tokens & themes | [GUI Styling and Themes](260530-gui-styling-and-themes.md) |
| Theme item properties / extraction | `lib/ItemProperties.ts` (`ThemeAspect`, `ThemeCss`, `getThemeData`) |
