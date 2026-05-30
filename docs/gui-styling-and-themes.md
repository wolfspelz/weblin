# GUI Styling and Themes (Browser-Extension Client)

How the browser-extension client (`github-n3qExt`) styles its entire GUI: how the UI is
isolated from the host page inside a shadow DOM, how a single stylesheet of CSS **design
tokens** (custom properties) drives every colour, spacing, shadow and font, how windows
and widgets are built from shared classes that consume those tokens, and how the **theme**
system re-skins the whole UI — and item iframes — at runtime by overriding a handful of
tokens.

This is the cross-cutting styling reference. Feature-specific rendering docs (e.g.
[Backpack Item Rendering](backpack-item-rendering.md)) describe *what* each surface draws;
this document describes the *styling machinery* they all share.

All paths are relative to `github-n3qExt/ChromeExt/src/`.

---

## 1. The big picture

The client paints a full-screen overlay on top of an arbitrary, hostile host page. Three
properties make that survivable and themeable:

1. **Style isolation** — all UI lives in a *closed* shadow DOM, so the host page's CSS
   cannot leak in and the client's CSS cannot leak out (§2).
2. **One token layer** — a single `<style>` of ~400 CSS custom properties on the shadow
   root's content container `#n3qD` defines the whole design system; component rules read
   tokens and never hard-code colours/sizes (§3, §4).
3. **Themes as token overrides** — a theme is just a second `<style>` injected after the
   base one that redefines some of those tokens; the derived values recompute and the GUI
   re-skins without touching any component rule (§6).

```
host page DOM
  └─ <div id="n3q">                        ← shadow host (invisible, position:fixed, max z-index)
       └─ #shadow-root (closed)
            ├─ <style data-type="base">     ← contentscript.css  (tokens + all component rules)
            ├─ <div id="n3qD" class="client">   ← full-screen overlay container; all UI lives here
            │     └─ windows, avatars, menus, toasts, …
            └─ <style data-type="theme">    ← injected/rebuilt at runtime; overrides tokens
```

---

## 2. Shadow DOM and style isolation

The display is created in `ContentAppDisplay.initDisplay()`
(`contentscript/ContentAppDisplay.ts:61`):

- A bare `<div>` anchor is created and a **closed** shadow root attached:
  `attachShadow({mode: 'closed'})` (`ContentAppDisplay.ts:66`). The anchor's id is set to
  `n3q` in `maintainDisplay()` (`ContentAppDisplay.ts:111`).
- The base stylesheet is appended to the shadow root as
  `<style data-type="base">…</style>` (`ContentAppDisplay.ts:72`).
- The content container `<div id="n3qD" class="client" dir="ltr">` is appended next
  (`ContentAppDisplay.ts:74`). **Every visible element lives inside `#n3qD`.**

### 2.1 `#n3q` vs `#n3qD`

- **`#n3q`** is the shadow *host* in the light DOM. It is forced invisible and inert with
  `:host(#n3q) { all: revert !important; position: fixed; width:0; height:0;
  overflow:hidden; z-index: 2147483647 }` (`contentscript.css:14`). It is a 0×0 box; it
  only exists to anchor the shadow root at the top of the page stack.
- **`#n3qD`** (`.client`) is the actual full-screen overlay *inside* the shadow root:
  `position: fixed; top/bottom/left/right: 0; z-index: 2147483647; pointer-events: none`
  (`contentscript.css:28`). `pointer-events: none` lets host-page clicks pass through the
  empty areas; individual widgets re-enable `pointer-events: auto` where they need input.

The host element is re-appended to the end of `<body>` and (optionally) promoted with the
Popover API so it stays above max-z-index page content (`ContentAppDisplay.ts:102` and
`:115`).

### 2.2 The `#n3qD` selector-prefix convention

**Every rule in `contentscript.css` is prefixed with `#n3qD`** (e.g. `#n3qD div`,
`#n3qD .button`, `#n3qD .window-title-bar`). Shadow DOM already isolates the styles, but
the prefix:

- scopes the aggressive neutral-reset rules (§2.3) to the client's own subtree, and
- keeps specificity uniform so that theme overrides on `#n3qD` (§6) reliably win.

**Implementor rule:** new selectors go under `#n3qD`, use kebab-case class names, and read
tokens via `var(--…)` rather than literal values.

### 2.3 Neutral reset

Because the shadow root inherits a few inheritable properties and because the client uses
plain tags, a reset block neutralises browser defaults on the common elements
(`#n3qD div, #n3qD span, … #n3qD button { margin:0; outline:none; background:none
transparent; border:none; padding:0; font:inherit; color:inherit; … }`,
`contentscript.css:500`). Form controls are then re-made interactive
(`display: inline-block; pointer-events: auto`, `contentscript.css:534`). This gives every
component a clean, predictable starting point so token-driven styling is the *only* source
of appearance.

### 2.4 How the stylesheet is bundled

`contentscript.css` is **not** shipped as a separate file. It is imported as a string
(`import styleText from './contentscript.css'`, `ContentAppDisplay.ts:9`) and injected at
runtime (§2). The build configures `css-loader` with `exportType: 'string'` specifically
for this file (`rspack.base.config.mjs:54`), and resolves `url(...)` references
(icons/images/sounds) to inline `data:` URLs via `asset/inline`
(`rspack.base.config.mjs:63`) so the shadow DOM is fully self-contained with no external
asset fetches (avoiding CORS/CSP problems on the host page). Optionally a `styleUrl`
parameter can replace the bundled CSS with a fetched stylesheet
(`ContentAppDisplay.ts:69`).

Embedded mode (`embedded/embedded.ts`) imports the same `contentscript.css` and uses the
same `ContentAppDisplay`; the shadow-DOM/CSS setup is identical to the extension — only the
background transport differs.

---

## 3. The CSS design-token layer

The whole design system is one large rule on `#n3qD`
(`contentscript.css:42` … ~`:495`, ~400 custom-property declarations). Defining the tokens
in one place means a single declaration site styles the entire UI; because every window
and widget lives inside `#n3qD`, the tokens cascade to all of them.

### 3.1 Token layering

Tokens are **layered**, derived from one another with `var()`, `calc()` and relative
`lch(from … )` colours, so a small set of base tokens drives many concrete values:

- **Base tokens** — the roots: `--brightness-factor` (`1` light / `-1` dark),
  `--base-background-color`, `--base-shadow-color`, `--base-border-color`,
  `--base-icon-color`, `--base-text-color`, `--base-text-font`/`--base-text-size`,
  `--base-spacing`, `--selected-color` (`contentscript.css:46`–`:150`).
- **Semantic tokens** — derived from the base: `--pane-background`, `--widget-background`,
  `--soft-shadow-filter`/`--hard-shadow-color`, `--tiny-pane-background`,
  `--fadeout-opacity`, the `--online-led-*`, `--drop-target-*`, and
  `--narrow-/wide-spacing-*` families.
- **Component tokens** — per-widget values that reference the semantic layer: `--button-*`,
  `--field-*`, `--menu-*` / `--menu-item-*`, `--window-*`, `--participant-chat-*`.

A change to one base token ripples through every derived token, e.g. changing
`--base-text-size` flows into `--base-text-line-height`
(`calc(var(--base-line-height-factor) * …)`) and on into button/menu heights.

### 3.2 Relative colours and `--brightness-factor`

The stylesheet leans heavily on **CSS relative colour syntax** `lch(from <color> L C H /
A)` to derive variants without hard-coding values:

- Alpha variants: `--soft-shadow-color: lch(from var(--base-shadow-color) l c h / 0.5)`
  (`contentscript.css:83`); `--drop-target-background-color:
  lch(from var(--drop-target-color) l c h / 0.1)` (`contentscript.css:139`).
- Lightness shifts keyed on the theme: `--thick-icon-color:
  lch(from var(--base-icon-color) calc(var(--brightness-factor) * 15 + l) c h)`
  (`contentscript.css:50`).

`--brightness-factor` (`1` light / `-1` dark, `contentscript.css:47`) is the **keystone**:
derived tokens multiply it into relative lightness shifts and `calc()` brightness factors,
e.g. hover brightness `calc(1.0 + var(--brightness-factor) * -0.1)` darkens by 10 % in
light mode and *brightens* by 10 % in dark mode (`contentscript.css:296`). Flipping this
one value inverts the whole UI's light/dark behaviour.

### 3.3 Token taxonomy (orientation)

| Group | Example tokens | Controls |
|-------|----------------|----------|
| Colour base | `--base-background-color`, `--base-icon-color`, `--base-shadow-color`, `--base-text-color` | foundational palette |
| Theme switch | `--brightness-factor` | light/dark behaviour of all derived tokens |
| Shadows | `--soft-shadow-filter`, `--hard-shadow-filter`, `--*-shadow-color` | drop shadows on panes/widgets |
| Spacing | `--base-spacing`, `--narrow-/wide-spacing-*` | paddings & gaps |
| Typography | `--base-text-font/-size/-line-height`, `--base-title-*`, `--code-text-*`, `--base-link-*` | fonts, sizes, links |
| Panes/widgets | `--pane-background`, `--pane-border-radius`, `--widget-background`, `--widget-focus-outline`, `--widget-border-radius`, `--sub-content-pane-*` | container & control chrome |
| Fields | `--field-background`, `--field-border-color`, `--field-box-shadow`, `--field-focus-*`, `--field-disabled-*` | inputs/textareas/checkboxes |
| Buttons | `--button-height`, `--button-side-padding`, `--button-*-{background,filter,box-shadow}`, radio/disabled variants | all button styles |
| Menus | `--menu-*`, `--menu-item-*`, `--menu-separator-line-color` | dropdown/context menus |
| Windows | `--window-*` (handle thickness, title/actionbar/content padding, border radius, filter) | window frame chrome |
| Status/feedback | `--online-led-*`, `--drop-target-*`, `--selected-color`, `--selection-rect-color`, `--danger-text-color`, `--disabled-opacity`, `--fadeout-opacity` | LEDs, drag targets, selection, errors, disabled |
| Chat | `--participant-chat-*` (`-from-server-*`, `-to-server-*`) | speech bubbles & input |

The component rules below the token block (`contentscript.css:497`+) carry essentially **no
hard-coded colours**; they read tokens, e.g. `#n3qD .button.style-default { color:
var(--button-text-color); background: var(--button-background) }`.

### 3.4 Dark mode is infrastructure, not (yet) automatic

The base CSS ships **light only**: `--brightness-factor: 1` and `color-scheme: light`, and
there is **no `@media (prefers-color-scheme: dark)`** rule (the only media query is
`@media print`, `contentscript.css:2`). The whole machinery to support dark mode exists —
flipping `--brightness-factor` to `-1` (plus a few base colours) re-skins everything — but
it is activated by a **theme** (§6), not by the OS preference. Wiring it to
`prefers-color-scheme` would be a one-rule change.

---

## 4. Windows and shared widgets

### 4.1 The window framework

All windows derive from `WindowBase` (`contentscript/WindowBase.ts`), with two common
specialisations:

- `FullWindow` (`contentscript/FullWindow.ts`) — `style = 'window'`, titlebar + close
  button, movable & resizable. Base for `BackpackWindow`, `ChatWindow`, `SettingsWindow`,
  `VidconfWindow`, …
- `PopupWindow` (`contentscript/PopupWindow.ts`) — `style = 'popup'`, movable but not
  resizable; base for transient popups (e.g. `BackpackItemInfo`).

`WindowBase.makeWindowFrameAndDecorations()` builds a standard DOM skeleton with stable
class names:

```html
<div class="window window-style-{style} window-sizing-{mode} {windowCssClasses…}">
  <div class="window-rows">
    <div class="window-title-bar"><div class="window-title-text">…</div></div>   <!-- optional -->
    <div class="window-actionbar">…</div>                                        <!-- optional -->
    <div class="window-buttons">…close/pin/undock…</div>
    <div class="window-content-wrapper"><div class="window-content">…</div></div>
  </div>
  <div class="resize-handle-n|s|e|w|ne|nw|se|sw"></div>                           <!-- if resizable -->
</div>
```

These classes are all styled from `--window-*` tokens (handle thickness, title/actionbar/
content padding, border-radius, `--window-filter` shadow), and the per-`style` variants
(`.window-style-window`, `.window-style-popup`, `.window-style-overlay`,
`.transparent`, `.window-style-noninteractive`) pick soft vs hard shadow, transparency, or
non-interactivity (`contentscript.css`, "Windows/popups/overlays" section ~`:1931`).

### 4.2 Per-window styling scope

A concrete window adds its own class in `initWindowSettings()` via
`this.windowCssClasses.push('…')` (e.g. `'backpackwindow'`, `'chatwindow'`,
`'settingswindow'`, `'personswindow'`, `'vidconfwindow'`). That class lands on the
`.window` element, enabling scoped rules like `#n3qD .backpackwindow .backpack-pane { … }`.
`windowSettingsId` is the key under which geometry is persisted in `Memory`
(`window.{id}`), so setting it gives a window saved position/size for free. `withActionbar`,
`withCloseButton`, `isMovable`, `isResizable`, `style`, `sizingMode` are the other knobs.

### 4.3 Shared widgets

Common, token-driven building blocks (all under `#n3qD`):

- **Buttons** — `.button` with a style modifier: `.style-default`, `.style-undecorated`,
  `.style-window` (round window-control buttons), `.style-popup`, `.style-item` (icon over
  label), `.style-square`, `.style-big`, `.style-merged` (grouped). Hover/active/disabled/
  radio states all come from `--button-*` tokens (`contentscript.css` "Buttons" ~`:892`).
- **Fields** — text/password inputs, textareas, checkboxes/radios styled via `--field-*`
  (`contentscript.css` "Fields" ~`:776`).
- **Menus** — `.menu` / `.menu.rootmenu` / `.menu.submenu` / `.menu .item` with `.focus`,
  `.disabled`, separators, driven by `--menu-*` (`contentscript.css` "Menu" ~`:1372`).
- **Utility classes** — `.hidden` (`visibility:hidden`, keeps layout), `.removed`
  (`display:none`), `.danger`, `.title`, `.code`, `.link`, `.icon-wrap` / `.icon-wrap.mask`
  (mask-image icons), `.sub-content-pane` (`contentscript.css` ~`:591`+).

### 4.4 Manipulating DOM and visual state from code

UI code does **not** write inline styles for state; it toggles CSS classes and lets the
stylesheet decide appearance. Helpers in `lib/DomUtils.ts`:

- `DomUtils.elemOfHtml(html)` / `elemsOfHtml(html)` — build elements from HTML templates
  (used to construct all window DOM and the `<style>` elements).
- `DomUtils.setElemClassPresent(elem, className, on)` — the canonical add/remove-class
  toggle, e.g. `setElemClassPresent(paneElem, 'drop-target', isTarget)`,
  `setElemClassPresent(actionbarElem, 'removed', !visible)`.
- `makeUniqueElemId()`, geometry helpers (`getElemLeftBottomRect()`), `makeElemAutoscroll()`.

Pointer interaction (drag, click, rubber-band) goes through `PointerEventDispatcher`
(`lib/PointerEventDispatcher.ts`), and stacking uses the `ContentApp.Layer*` constants
(entities 30, windows 50, popups 60, toasts 70, drag 99, menu 110;
`ContentApp.ts` ~`:1200`) via `app.toFront(elem, layer)` rather than ad-hoc z-index values.

**Implementor rule:** express state as a class and style it in `contentscript.css` reading
tokens — don't set colours/sizes inline. This keeps everything themeable (§6) and
consistent.

---

## 5. Item iframes

Interactive items render in **iframes** (separate documents, e.g. via
`assets/iframe.html`, `ItemFrameWindow.ts`), so the shadow-DOM CSS does **not** cascade into
them. To keep embedded item UIs visually consistent, the active theme CSS is **pushed** into
each frame over the iframe API as a `ClientThemeCssNotification`
(`lib/WeblinClientIframeApi.ts`), sent by `ContentItemFrames.sendThemeCssToAllFrames()`
(`contentscript/ContentItemFrames.ts:228`) whenever themes change, and by `BackpackItemInfo`
for its detail-popup frame. The iframe injects that CSS into its own document. This is the
only path by which tokens/themes reach iframe content.

---

## 6. The theme system

A **theme** re-skins the GUI at runtime. Crucially, a theme is *not* a separate styling
mechanism — it is a second stylesheet that **overrides the same tokens** described in §3.
Because tokens are the single source of appearance, a theme typically needs only a short
list of overrides.

The relationship is worth stating precisely, because it is easy to get backwards:

- **CSS variables are core to the CSS of the *whole* GUI, not to themes.** They exist and
  style every surface even when **no theme is active** — the base stylesheet is itself just
  a set of token definitions plus rules that read them (§3). Remove every theme and the
  tokens still paint the entire UI.
- **They are what makes theming *easy*, not what makes theming *work*.** A theme could in
  principle restyle individual selectors with no tokens at all; the token layer simply gives
  it **direct access to styling at a single point**. Re-skinning the whole client becomes
  "change a few variable values" instead of "rewrite the rules that consume them" — override
  `--brightness-factor` plus a few base colours and the derived semantic/component tokens
  recompute across panes, buttons, fields, menus, LEDs and tiles in one stroke; or override
  one narrow token (e.g. `--selected-color`) to retint a single aspect.

### 6.1 What a theme is

`ThemeUtils.Theme` (`lib/ThemeUtils.ts:21`) =
`{ id, name, orderIndex, sourceType, sourceId, isEnabled, css }`. The `css` is arbitrary CSS
text. A theme re-skins the UI by redefining tokens on `#n3qD`, e.g. a dark theme:

```css
#n3qD {
    --brightness-factor: -1;
    --base-background-color: lch(20 0 0);
    --base-icon-color:       lch(80 0 0);
    --base-text-color:       lch(95 0 0);
}
```

Every derived semantic/component token recomputes from these, re-skinning panes, buttons,
fields, menus, LEDs and tiles in one stroke. A theme *can* also target concrete selectors
for finer control, but the common, recommended case is a handful of token overrides.

### 6.2 Theme sources and aggregation (background)

`BackgroundThemeManager` (`background/BackgroundThemeManager.ts`) aggregates themes from
three **sources**, ordered by priority in `ThemeUtils` (`lib/ThemeUtils.ts:10`):

1. `extension` (priority 1) — themes offered by other browser extensions.
2. `item` (priority 2) — **theme items in the backpack** (see §6.3).
3. `user` (priority 3, applied last/on top) — user-authored themes.

All themes are kept in a map, persisted to local storage under `Themes`, and sorted with
`cmpTheme()` (`lib/ThemeUtils.ts:40`): by source priority, then `sourceId`, then
`orderIndex`, then `name`. The sorted list is broadcast to every tab as
`ContentMessage.type_themes` (a `ContentThemesMessage`, `BackgroundThemeManager.ts:241`) on
init, on tab-content-ready, and whenever themes change. The feature is gated by
`Config.get('themes.enabled')` (default on); extension themes refresh on a timer
(`themes.updateIntervalSec`).

### 6.3 Theme *items* ride the backpack pipeline

A theme item is an ordinary inventory item carrying `Pid.ThemeAspect` (marks it a theme),
`Pid.ThemeCss` (the CSS text) and its activation flag `Pid.ActivatableIsActive` (enabled?).
`ItemProperties.getThemeData()` (`lib/ItemProperties.ts`) extracts
`{id, name, isEnabled, css, x, y}` from any item with `ThemeAspect`.

`BackgroundThemeManager.onBackpackUpdate()` (`BackgroundThemeManager.ts:193`) listens to
backpack updates, maps every theme item through `getThemeData()`, orders them by backpack
position (`y`, then `x`, then name/id), and registers them as the `item`-source themes.
Because of this, **enabling/disabling or editing a theme item propagates exactly like any
other item-property change** — through the same `onBackpackUpdate` → broadcast pipeline
documented in [Backpack Item Rendering §6–§7](backpack-item-rendering.md). No special
channel is involved.

### 6.4 Injection into the shadow DOM (content)

`ContentThemeManager` (`contentscript/ContentThemeManager.ts`) receives `type_themes`,
keeps the **enabled** themes, and concatenates their CSS (each prefixed with a
`/* Theme {id} */` comment) in `updateEnabledThemesCss()` (`ContentThemeManager.ts:65`).
`updateDisplay()` (`ContentThemeManager.ts:78`) then **removes any previous theme style and
appends a fresh** `<style data-type="theme">…</style>` **to the shadow root**
(`app.getShadowDomRoot().append(...)`).

Two consequences make theming work:

- **Order/cascade.** The theme `<style>` is appended to the shadow root *after* the base
  `<style data-type="base">` (and after the `#n3qD` div). Same-specificity declarations on
  `#n3qD` from the theme therefore win over the base token definitions — which is exactly
  why overriding a token on `#n3qD` re-skins the UI.
- **Live rebuild.** The element is dropped and rebuilt on every change, and the change is
  also announced via `themesChangedListeners`, which is what triggers re-sending CSS into
  item iframes (§5). Disabling all themes removes the element entirely, reverting to the
  base tokens.

### 6.5 Authoring a theme — practical notes

- Scope overrides to `#n3qD` (or a more specific in-client selector). Don't rely on bare
  `:root`/`html` — the tokens live on `#n3qD`.
- Prefer overriding **base tokens** (`--brightness-factor`, the `--base-*` colours,
  `--base-spacing`) and let derived tokens recompute; override semantic/component tokens
  only to retune a specific area; fall back to concrete selectors only when a token doesn't
  exist for what you need.
- Relative-colour overrides compose well: setting `--base-shadow-color` automatically
  updates `--soft-shadow-color`/`--hard-shadow-color` and everything that uses them.

---

## 7. Summary / checklist for GUI implementors

- All UI lives inside a **closed shadow DOM** under `#n3qD`; the host `#n3q` is an invisible
  max-z-index anchor. Prefix every selector with `#n3qD`; rely on the neutral reset for a
  clean baseline.
- The stylesheet is **injected as a string** (bundled with `css-loader exportType:'string'`,
  assets inlined). There is no separate CSS file at runtime.
- **Never hard-code colours, spacings, shadows or fonts.** Read CSS **tokens** (`var(--…)`)
  defined in the `#n3qD` block of `contentscript.css`. Tokens are layered
  base → semantic → component; `--brightness-factor` is the light/dark keystone; colours use
  relative `lch(from …)` syntax.
- Build windows on `WindowBase`/`FullWindow`/`PopupWindow`; add a per-window class via
  `windowCssClasses` and a `windowSettingsId` for geometry persistence. Reuse the shared
  `.button`/`.field`/`.menu`/utility classes.
- Express visual state as **CSS classes toggled in code** (`DomUtils.setElemClassPresent`),
  not inline styles. Use `ContentApp.Layer*` + `toFront()` for stacking.
- A **theme** is a `<style>` appended after the base one that **overrides tokens** on
  `#n3qD`; it re-skins everything without changing component rules. Themes come from
  extension/item/user sources; **item themes are backpack items** and flow through the
  normal backpack-update pipeline. Theme CSS is also pushed into item **iframes** because
  shadow-DOM styles don't reach them.
- **Dark mode** is token-ready but ships disabled (no `prefers-color-scheme` rule); it is
  delivered as a theme that flips `--brightness-factor`.

### Key files

| Concern | File |
|---------|------|
| Shadow DOM creation + base/theme `<style>` injection | `contentscript/ContentAppDisplay.ts` |
| Design tokens + all component CSS | `contentscript/contentscript.css` (`#n3qD` block ~`:42`–`:495`) |
| CSS bundling (string export, asset inlining) | `rspack.base.config.mjs` |
| Window base classes + DOM skeleton, geometry persistence | `contentscript/WindowBase.ts`, `FullWindow.ts`, `PopupWindow.ts` |
| DOM/class helpers | `lib/DomUtils.ts` (`setElemClassPresent`, `elemOfHtml`) |
| Stacking layers | `contentscript/ContentApp.ts` (`Layer*`, `toFront`) |
| Theme type + source ordering | `lib/ThemeUtils.ts` |
| Theme item properties / extraction | `lib/ItemProperties.ts` (`ThemeAspect`, `ThemeCss`, `getThemeData`) |
| Theme aggregation + broadcast (background) | `background/BackgroundThemeManager.ts` |
| Theme CSS injection (content) | `contentscript/ContentThemeManager.ts` |
| Theme CSS → item iframes | `contentscript/ContentItemFrames.ts`, `lib/WeblinClientIframeApi.ts` |
| Theme feature flags | `lib/Config.ts` (`themes.enabled`, `themes.updateIntervalSec`) |
